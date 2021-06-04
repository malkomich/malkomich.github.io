---
date: 2021-06-04 15:07:57
layout: post
title: Authenticating REST services with OAuth2
description: REST services authenticated with an OAuth2 Client for Java
image: /assets/img/uploads/image.jpg
optimized_image: /assets/img/uploads/image.jpg
category: security
tags:
  - OAuth2
  - REST
  - microservice
  - authorization
  - security
  - Spring
  - Vertx
  - Quarkus
author: malkomich
paginate: false
---
## 1. Introduction

When it comes to adding authorization to call secured services, we realize not only that the configuration changes depending on which framework you are going to use, but that for each HTTP client you use, you must configure OAuth2 in a different way.

For this reason, the simplest thing when implementing an authorization layer through OAuth2 to call those services, would be to outsource the generation of the tokens to a new personalized client. This way we would have a maintainable integration, isolated from the REST client we are using.

This article guides you through the creation of a simple library which allow you to grant your HTTP requests with the required authorization token, and integrate in your services whatever client you may use.

![OAuth2 Schema](/assets/img/uploads/oauth-and-openid-connect-core-concepts1.png "OAuth2 Schema")

The authorization flow is described in the image above:

1. Authorization request is sent from client to OAuth server.
2. Access token is returned to the client.
3. Access token is then sent from client to the API service (acting as resource server) on each request for protected resource access.
4. Resource server check the token with the OAuth server, to confirm the client is authorized to consume that resource.
5. Server responds with required protected resources.



## 2. Setting up the required dependencies

We will need a few libraries to build our custom OAuth2 client.

First of all, the **Apache HTTP** client library, which will provide us with the HTTP client for the integration with the authorization server, as well as a toolset for the request building. So it would be the core library for our client.

In the second one, we find another Apache library, called ***cxf-rt-rs-security-oauth2***. In this case, this dependency would be optional, since we only need a set of predefined values in the OAuth2 Protocol definition, gathered in the `OAuthConstants` class. We could also defined those values by ourselves, to get rid of that dependency.

Lastly, we include the json library. This library is a helpful toolset when we are handling JSON data. It is really useful to parse and manipulate JSON in Java.

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5</version>
    </dependency>

    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-rt-rs-security-oauth2</artifactId>
        <version>3.4.2</version>
    </dependency>

    <dependency>
        <groupId>org.json</groupId>
        <artifactId>json</artifactId>
        <version>20160212</version>
    </dependency>
</dependencies>
```

## 3. Building the OAuth2 request

We have to build the request to the server which will authorize our service as a granted client.
To achieve this, we need to define the OAuth2 configuration we are using, including the grant type, the authorization server URL, the credentials for the given grant type, and the scope for the resource we are requesting.

```java
class OAuth2Config {
  String grantType;
  String clientId;
  String clientSecret;
  String username;
  String password;
  String accessTokenUri;
  String scope;
}
```

Once we have the configuration values initialized, we can use them to build the HTTP request for the authorization server.
Typically, the HTTP method used to get the access token, will be a POST, as defined in the [OAuth 2.0 Authorization Protocol specification](https://tools.ietf.org/html/draft-ietf-oauth-v2-22):

> The client MUST use the HTTP "POST" method when making access token requests.

Depending on the grant type we define, we must define different parameters on the POST request. We will use a list of `NameValuePair` to gather all those needed parameters.

```java
HttpUriRequest buildRequest() {
  List<NameValuePair> formData = new ArrayList<>();
  formData.add(new BasicNameValuePair(GRANT_TYPE, config.getGrantType()));

  if (config.getScope() != null && !config.getScope().isBlank()) {
    formData.add(new BasicNameValuePair(SCOPE, config.getScope()));
  }

  if (CLIENT_CREDENTIALS_GRANT.equals(config.getGrantType())) {
    formData.add(new BasicNameValuePair(CLIENT_ID, config.getClientId()));
    formData.add(new BasicNameValuePair(CLIENT_SECRET, config.getClientSecret()));
  }

  if (RESOURCE_OWNER_GRANT.equals(config.getGrantType())) {
    formData.add(new BasicNameValuePair(RESOURCE_OWNER_NAME, config.getUsername()));
    formData.add(new BasicNameValuePair(RESOURCE_OWNER_PASSWORD, config.getPassword()));
  }

  return RequestBuilder.create(HttpPost.METHOD_NAME)
                       .setUri(config.getAccessTokenUri())
                       .setEntity(new UrlEncodedFormEntity(formData, StandardCharsets.UTF_8))
                       .build();
}
```

## 4. Executing the OAuth2 request

Since we are building an OAuth2 client as basic as possible, we will use the default HTTP client from ***Apache HTTP*** library, to send our request to the authorization server.

```java
CloseableHttpResponse doRequest(HttpUriRequest request) {
  CloseableHttpClient httpClient = HttpClients.createDefault();
  try {
    return httpClient.execute(request);
  } catch (IOException e) {
    throw new OAuth2ClientException("An error occurred executing the request.", e);
  }
}
```

Once we receive the response, we need to handle it, extracting the information we need for the access token.

```java
class OAuth2Response {
  HttpEntity httpEntity;
}
```

We should check for errors before parsing the content to get the access token. We can consider here errors in the credentials we defined, a wrong or malformed URL, or any internal error from the authorization server.

```java
OAuth2Response execute(HttpUriRequest request) {
  CloseableHttpResponse httpResponse = doRequest(request);
  HttpEntity httpEntity              = httpResponse.getEntity();
  int statusCode                     = httpResponse.getStatusLine()
                                                   .getStatusCode();
  if (statusCode >= 400) {
    throw new OAuth2ClientException(statusCode, httpEntity);
  }
  return new OAuth2Response(httpEntity);
}
```

We should not forget to close the `httpResponse`, to avoid the memory leakage. But is pretty important to wait until it is read properly, since it contains an InputStream which would become inaccessible once we have closed it.



Typically, the response content will come on a JSON format, with the access token data in a key-value schema. However, we should consider a server handling the data on a different format, like XML or URL encoded.

For the scope of this article, we will consider our authorization server are giving us a JSON formatted content. The ***org.json:json*** library we included earlier will help us on the deserialization.

```java
JSONObject handleResponse(HttpEntity entity) {
  String content     = extractEntityContent(entity);
  String contentType = Optional.ofNullable(entity.getContentType())
                               .map(Header::getValue)
                               .orElse(APPLICATION_JSON.getMimeType());
  return new JSONObject(content);
}

String extractEntityContent(HttpEntity entity) {
  try {
    return EntityUtils.toString(entity, StandardCharsets.UTF_8);
  } catch (IOException e) {
    throw new OAuth2ClientException("An error occurred while extracting entity content.", e);
  }
}
```

Given the `JSONObject`, it becomes much easier to handle the response, since we can retrieve instantly each value we are interested in.

## 5. Putting all together

The goal here is to obtain an access token to call the secured services we need. However, sometimes we also need to know some additional data, like the timestamp when the token is going to expire, the token type we are receiving, or the refresh token in the case the grant type is defined so.

```java
class AccessToken {
  long expiresIn;
  String tokenType;
  String refreshToken;
  String accessToken;

  AccessToken(JSONObject jsonObject) {
    expiresIn    = jsonObject.optLong(ACCESS_TOKEN_EXPIRES_IN);
    tokenType    = jsonObject.optString(ACCESS_TOKEN_TYPE);
    refreshToken = jsonObject.optString(REFRESH_TOKEN);
    accessToken  = jsonObject.optString(ACCESS_TOKEN);
  }
}
```

Finally, we will get a client which will retrieve the access token data need to grant our calls to the services, based on the configuration we defined.

```java
AccessToken accessToken() {
  HttpUriRequest request  = buildRequest();
  OAuth2Response response = execute(request);

  return new AccessToken(handleResponse(response.getHttpEntity()));
}
```

## 6. Put into practice

But, how could we integrate this custom client in our service?

Well, as I mentioned at the beginning of the article, the idea of this custom OAuth2 client is to be isolated from the framework and/or the HTTP client we are using to consume the secured services.

So I will show you a few examples of how to integrate it in different service environments.

#### 6.1. Spring Framework - WebClient

```java
class WebClientConfig {

  @Bean(name = "securedWebClient")
  WebClient fetchWebClient(@Value("${host}") String host,
                           OAuth2Config oAuth2Config) {
    OAuth2Client oAuth2Client = OAuth2Client.withConfig(oAuth2Config).build();
    return WebClient.builder()
                    .filter(new OAuth2ExchangeFilter(oAuth2Client))
                    .baseUrl(host)
                    .build();
  }

  @Bean
  @ConfigurationProperties(prefix = "security.oauth2.config")
  OAuth2Config oAuth2Config() {
    return new OAuth2Config();
  }

  class OAuth2ExchangeFilter implements ExchangeFilterFunction {
    
    OAuth2Client oAuth2Client;

    @Override
    public Mono<ClientResponse> filter(ClientRequest request,
                                       ExchangeFunction next) {
      String token = Optional.ofNullable(oAuth2Client.accessToken())
                             .map(AccessToken::getAccessToken)
                             .map("Bearer "::concat)
                             .orElseThrow(() -> new AccessDeniedException());

      ClientRequest newRequest = ClientRequest.from(request)
                                              .header(HttpHeaders.AUTHORIZATION, token)
                                              .build();
      return next.exchange(newRequest);
    }
  }
}
```

#### 6.2. Spring Framework - Feign Client

```java
class FeignClientConfig {

  @Bean
  OAuthRequestInterceptor repositoryClientOAuth2Interceptor(OAuth2Client oAuth2Client) {
    return new OAuthRequestInterceptor(oAuth2Client);
  }

  class OAuthRequestInterceptor implements RequestInterceptor {

    OAuth2Client oAuth2Client;

    @Override
    public void apply(RequestTemplate requestTemplate) {
      String authToken = Optional.ofNullable(oAuth2Client.accessToken())
                                 .map(AccessToken::getAccessToken)
                                 .map("Bearer "::concat)
                                 .orElseThrow(() -> new AccessDeniedException());

      requestTemplate.header(HttpHeaders.AUTHORIZATION, authToken);
    }
  }
}
```

#### 6.3. Vert.x - Web Client

```java
class ProtectedResourceHandler implements Handler<RoutingContext>  {

  OAuth2Config oAuth2Config;

  ProtectedResourceHandler() {
    // Resource handler initialization
    oAuth2Config = oauth2Config(config);
  }

  private OAuth2Config oauth2Config(JsonObject oauth2Properties) {
    
    return OAuth2Config.builder()
        .grantType(oauth2Properties.getString("grantType"))
        .accessTokenUri(oauth2Properties.getString("accessTokenUri"))
        .clientId(oauth2Properties.getString("clientId"))
        .clientSecret(oauth2Properties.getString("clientSecret"))
        .username(oauth2Properties.getString("username"))
        .password(oauth2Properties.getString("password"))
        .scope(oauth2Properties.getString("scope"))
        .build();
  }

  @Override
  public void handle(RoutingContext routingContext) {

    WebClient.create(routingContext.vertx())
        .getAbs(host)
        .uri(endpoint)
        .putHeader(HttpHeaders.AUTHORIZATION.toString(), generateToken())
        .send()
        .onSuccess(httpResponse -> { /* Successful response handler */ })
        .onFailure(err -> { /* Error response handler */ });
  }

  String generateToken() {

    return Optional.of(OAuth2Client.withConfig(oAuth2Config).build())
        .map(OAuth2Client::accessToken)
        .map(AccessToken::getAccessToken)
        .map("Bearer "::concat)
        .orElseThrow(() -> new AccessDeniedException());
  }
}
```



#### 6.4. Quarkus - RestEasy

```java
@RegisterRestClient
@RegisterClientHeaders(SecurityHeaderFactory.class)
interface DocumentClient {
  
  // External endpoints definition
}

class SecurityHeaderFactory implements ClientHeadersFactory {

  OAuth2Client oAuth2Client;

  @Inject
  SecurityHeaderFactory(OAuth2Properties oAuth2Properties) {
    oAuth2Client = OAuth2Client
        .withConfig(oauth2Config(oAuth2Properties))
        .build();
  }

  @Override
  public MultivaluedMap<String, String> update(MultivaluedMap<String, String> incomingHeaders,
                                               MultivaluedMap<String, String> outgoingHeaders) {
    outgoingHeaders.add(HttpHeaders.AUTHORIZATION.toString(), generateToken());
    return outgoingHeaders;
  }

  String generateToken() {
    return Optional.of(oAuth2Client)
        .map(OAuth2Client::accessToken)
        .map(AccessToken::getAccessToken)
        .map("Bearer "::concat)
        .orElseThrow(() -> new AccessDeniedException());
  }

  OAuth2Config oauth2Config(OAuth2Properties oAuth2Properties) {
    return OAuth2Config.builder()
        .grantType(oAuth2Properties.getGrantType())
        .accessTokenUri(oAuth2Properties.getAccessTokenUri())
        .clientId(oAuth2Properties.getClientId())
        .clientSecret(oAuth2Properties.getClientSecret())
        .username(oAuth2Properties.getUsername())
        .password(oAuth2Properties.getPassword())
        .scope(oAuth2Properties.getScope())
        .build();
  }
}

```



#### 7. Conclusion

In this article, we have seen how we can set up a simple OAuth2 Client, and how we can integrate it in your REST calls to retrieve a secured resource from an external service.

You can check the code used for the OAuth2 Client, the repository is available over on [Github](https://github.com/malkomich/oauth2-token-client).