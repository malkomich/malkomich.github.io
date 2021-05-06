---
date: 2021-05-03 17:33:54
layout: post
title: Authenticating REST services with OAuth2
description: REST services authenticated with an OAuth2 Client for Java
optimized_image: /assets/img/uploads/image.jpg
category: "{{slug}}"
tags:
  - OAuth2
  - REST
  - microservice
  - authorization
  - security
  - Spring
author: malkomich
paginate: false
---
## 1. Introduction

When it comes to adding authorization to call secured services, we realize not only that the configuration changes depending on which framework you are going to use, but that for each HTTP client you use, you must configure OAuth2 in one way or another.

For this reason, the simplest thing when implementing an authorization layer through OAuth2 to call those services, would be to outsource the generation of the tokens to a new personalized client. This way we would have a maintainable integration, isolated from the REST client we are using.

This article guides you through the creation of a simple library which will allow you to grant your HTTP requests with the required authorization token, and integrate in your services whatever client you may use.

## 2. Building the OAuth2 request

We have to build the request to the server which will authorize our service as a granted client.
To achieve this, we need to define the OAuth2 configuration we are using, including the grant type, the authorization server URL, the credentials for the given grant type, and the scope for the resource we are requesting.

```java
public class OAuth2Config {
  private String grantType;
  private String clientId;
  private String clientSecret;
  private String username;
  private String password;
  private String accessTokenUri;
  private String scope;
}
```

Once given we have the configuration values initialized, we can use them to build the HTTP request for the authorization server.
As defined in the [OAuth 2.0 Authorization Protocol specification](https://tools.ietf.org/html/draft-ietf-oauth-v2-22):

> The client MUST use the HTTP "POST" method when making access token requests.

Depending on the grant type we define, we must define different parameters on the POST request.

```java
final List<NameValuePair> formData = new ArrayList<>();
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
```

## 3. Executing the OAuth2 request



\`\``java



\`\``