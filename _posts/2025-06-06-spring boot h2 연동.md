---
layout: post
title: "Spring Boot에서 H2 연동"
subtitle: "Spring Boot에서 H2 연동"
categories: "Spring Boot"
tags: ["Spring Boot"]
sidebar: ['article-menu']
---

## H2 연동?

### build.gradle
``` yaml
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
runtimeOnly 'com.h2database:h2'
```

### test/resource/application.yml
``` yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:~/text
    username: sa
    password:

  jpa:
    hibernate:
      ddl_auto: update
    open-in-view: false
```
