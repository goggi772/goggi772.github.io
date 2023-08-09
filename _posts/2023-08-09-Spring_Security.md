---
layout: post
title: "Spring Security란?"
---

회원 관리 기능을 구현하기 위해서는 __인증(Authentication)__ 과 __인가(Authorization)__ 에 대한 
처리가 필수적이다. 그래서 내가 회원 관리 기능을 만들 때 사용했던 것이 Spring Secruity였다.
Spring Security는 Spring에서 사용할 수 있는 별도의 프레임워크로 회원 관리 관련 기능들을 제공한다.
또한 Spring Security는 보안과 관련해서 많은 옵션을 쳬계적으로 제공해 주어서 일일히 보안관련 로직을 
작성할 필요가 없어서 편리하다. 이제 이 Spring Security가 무엇이고 어떤기능들을 제공하는지 알아보자.

# Spring Security

---

Spring Security는 Spring기반의 애플리케이션의 보안 부분인 인증과 권한, 인가 등을 담당하는
Spring 하위 프레임워크이다. Spring Security에 대해 자세히 설명하기 전에 먼저 인증(Authentication)
과 인가(Authorization)에 대해 설명해보면

<br>

+ **_인증(Authentication)_**: 접근하는 사용자가 본인이 맞는지 검증하는 절차    
+ **_인가(Authorization)_**: 인증이 된 사용자가 요청한 자원에 접근 가능한 권한이 있는지
확인하는 절차  

<br>

즉, 인증은 사용자가 아이디와 비밀번호를 입력하여 로그인 할 때 입력한 정보를 확인하여
정당한 사용자인지 검증하는 것이고, 인가는 일반 사용자가 관리자 페이지 같이 접근 권한이 없는
자원에 접근하려 할 때 시도를 거부하고 접근을 제한하는 절차를 나타낸 것이다.  
<br>
Spring Security에서는 위와 같은 인증 절차를 거치고 인가 절차를 거쳐 검증하게 된다. 이때 
Spring Security에서는 이러한 인증과 인가 절차를 거치기 위해 Principal을 아이디로, 
Credential을 비밀번호로 사용하는 Credential 기반 인증 방식을 사용하게 된다.  
여기서 Principal은 접근 주체이자 아이디로 보호받는 리소스에 접근하는 대상을 나타내고
Credential은 리소스에 접근하려고 하는 대상의 비밀번호를 나타낸다.

<br>

이제 Spring Security의 Architecture를 살펴보자.

![image](/assets/img/2023-08-09-Spring_Security/spring_security_architecture.png "아키텍쳐")
<br>

위와 같이 Spring Security는 인증과 인가에 대한 부분을 Filter의 흐름에 따라 처리하고 있다.
Spring Security Architecture의 처리 과정:  
- Http request가 들어온다.  
- `AuthenticationFilter`가 request의 정보를 가로채 `UsernamePasswordAuthenticationToken`의
인증용 객체를 생성한다.  
- `AuthenticationManager`를 상속받는 `ProviderManager`에게 앞에서 생성한 
`UsernamePasswordToken` 객체를 전달한다.  
- `AuthenticationManager`는 등록된 `AuthenticationProvider`를 조회하여 인증을 요청한다.  
- 사용자 인증 정보를 가져올 수 있는 `UserDetailService`에게 사용자 정보를 넘긴다.
- 사용자 정보로 `UserDetails` 객체를 생성하여 `AuthenticationProvider`에게 넘긴다.
- 받은 `UserDetails`객체의 사용자 정보를 DB의 사용자 정보와 비교한다.
- 인증이 성공적으로 되면 사용자 정보와 권한 등을 담은 `Authentication`객체를 `AuthenticationFilter`
에 반환한다.
- 받은 `Authentication`객체를 `SecurityContext`에 저장한다.



