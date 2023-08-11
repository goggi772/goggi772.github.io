---
layout: post
title: "Spring Security로 login구현하기"
---

이번에 Spring Boot도 공부할 겸 간단한 프로젝트를 만들었다. 그 중에서 가장 기본이 되는 Login기능을 
Spring Security를 이용하여 구현해 보았다. Spring Security를 적용하는 단계부터 천천히 알아보자.

### Spring Security Dependencies 추가

```gradle
dependencies {
    ...
    implementation 'org.springframework.boot:spring-boot-starter-security'
    ...
}
```

`build.gradle`의 dependencies에 의존성을 추가해주기만 하면 Spring Security를 바로 사용할 수 있다. 
그 외에 Intellij나 start.spring.io에서 Spring Boot 프로젝트를 생성할 때 Dependencies에 Spring
Security를 추가해도 사용할 수 있다.
