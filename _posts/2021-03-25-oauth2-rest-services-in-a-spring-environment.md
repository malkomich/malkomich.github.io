---
date: 2021-03-20 10:41:35
layout: post
title: OAuth2 REST services in a Spring environment
description: OAuth2 REST services in a Spring environment
category: "{{slug}}"
author: malkomich
paginate: false
---
Today, we will be configuring a REST client in a Spring Framework environment, and the securization for that client.



Probably it would be easier if you configure a singleton RestTemplate, with the recommended configuration from Spring Boot. However, in any enterprise application you will have different client configurations at some point, so I recommended to use a RestTemplateBuilder, with your common client configuration, to be able to generate custom instances for each client you may need a new configuration.

For example, imagine