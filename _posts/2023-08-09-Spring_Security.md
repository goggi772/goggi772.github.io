---
layout: post
title: "Spring Security란?"

categories:
  - Spring

tags:
  - [spring boot, security]
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
처리하는 과정을 하나씩 살펴보자.

<br/>

<h4 class="text-center"> UsernamePasswordAuthenticationToken </h4> 

<br/>

```java
public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {

    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;

    private final Object principal;

    private Object credentials;

    //인증 전 객체
    public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
        super(null);
        this.principal = principal;
        this.credentials = credentials;
        setAuthenticated(false);
    }

    //인증 후 객체
    public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
                                               Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.credentials = credentials;
        super.setAuthenticated(true); // must use super, as we override
    }

    public static UsernamePasswordAuthenticationToken unauthenticated(Object principal, Object credentials) {
        return new UsernamePasswordAuthenticationToken(principal, credentials);
    }

    public static UsernamePasswordAuthenticationToken authenticated(Object principal, Object credentials,
                                                                    Collection<? extends GrantedAuthority> authorities) {
        return new UsernamePasswordAuthenticationToken(principal, credentials, authorities);
    }

    @Override
    public Object getCredentials() {
        return this.credentials;
    }

    @Override
    public Object getPrincipal() {
        return this.principal;
    }

    @Override
    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        Assert.isTrue(!isAuthenticated,
                "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
        this.credentials = null;
    }
}
```

request가 들어오면 먼저 `AuthenticationFilter`가 request를 가로채 그 정보를 바탕으로 위 `UsernamePasswordAuthenticationToken`
객체를 생성해 `AuthenticationManager`에게 전달한다. 이 때, 위 클래스에는 생성자가 두개 있는데 첫 번째 생성자는 인증이 되기 전의 객체를
생성하고 두 번째 생성자는 인증이 된 후의 객체를 생성한다. 그리고 위에서 설명한 것과 같이 사용자의 ID를
Principal, 비밀번호를 Credentials로 사용함을 볼 수 있다.
<br/>

<h4 class="text-center"> AuthenticationManager </h4>

<br/>

```java
public interface AuthenticationManager {
    
    Authentication authenticate(Authentication authentication) throws AuthenticationException;

}
```
```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

    public ProviderManager(AuthenticationProvider... providers) {
        this(Arrays.asList(providers), null);
    }
    
    public ProviderManager(List<AuthenticationProvider> providers) {
        this(providers, null);
    }
    
    public ProviderManager(List<AuthenticationProvider> providers, AuthenticationManager parent) {
        Assert.notNull(providers, "providers list cannot be null");
        this.providers = providers;
        this.parent = parent;
        checkState();
    }
}
```

<h4 class="text-center"> AuthenticationProvider </h4>

```java
public interface AuthenticationProvider {

    Authentication authenticate(Authentication authentication) throws AuthenticationException;

    boolean supports(Class<?> authentication);

}
```

그 후 인터페이스 `AuthenticationManager`를 상속 받은 `ProviderManager`가 사용자 인증 요청에 
필요한 `AuthenticationProvider`목록을 살펴보고 전달된 인증 개체인 `UsernamePasswordAuthenticationToken`
를 기반으로 사용자 인증을 한다. 이 때 사용자 이름인 username을 기반으로 사용자의 세부 정보를 찾기
위해 `UserDetailsService`를 사용한다.  
<br/>

<h4 class="text-center"> UserDetailsService </h4>

<br/>

```java
public interface UserDetailsService {
    
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;

}
```

Username을 기반으로 DB에 있는 회원 정보를 검색하여 일치하는 회원 정보가 있으면 `UserDetails`를 구현한
객체를 반환한다. 개발자는 이 `UserDetailsService`를 상속받는 커스텀 DetailsService를 구현하여야 한다.  
<br/>
위 과정을 거쳐 `AuthenticationProvider`에 의해 사용자 인증이 성공적으로 이루어 지면 완전한 인증객체가
반환이 되고 이를 `AuthenticationManager`가 `AuthenticationFilter`에게 전달하고 필터는
`AuthenticationFilter`는 이를 `SecurityContext`에 저장한다. 이 과정에서 사용자 인증에 
실패하게 되면 `AuthenticaionException`을 발생시키게 된다.


<br/>

<h4 class="text-center"> 요약 </h4>

<br/>

Spring Security Architecture의 처리 과정:  

- Http request가 들어온다.
- `AuthenticationFilter`가 request의 정보를 가로채 `UsernamePasswordAuthenticationToken`의
  인증용 객체를 생성한다.
- `AuthenticationManager`를 상속받는 `ProviderManager`에게 앞에서 생성한
  `UsernamePasswordAuthenticationToken` 객체를 전달한다.
- `AuthenticationManager`는 등록된 `AuthenticationProvider`를 조회하여 인증을 요청한다.
- 사용자 인증 정보를 가져올 수 있는 `UserDetailService`에게 사용자 정보를 넘긴다.
- 사용자 정보로 `UserDetails` 객체를 생성하여 `AuthenticationProvider`에게 넘긴다.
- 받은 `UserDetails`객체의 사용자 정보를 DB의 사용자 정보와 비교한다.
- 인증이 성공적으로 되면 사용자 정보와 권한 등을 담은 `Authentication`객체를 `AuthenticationFilter`
  에 반환한다.
- `SecurityContextHolder`가 `Authentication`객체를 `SecurityContext`에 저장한다.



