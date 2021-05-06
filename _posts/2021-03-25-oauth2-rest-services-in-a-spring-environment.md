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

We have to build the request to the server which will authenticate our service as a granted client.





## 3. Put into practice