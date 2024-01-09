---
layout: post
title: "UsernameNotFoundException이 발생하지 않는 이유"

categories:
  - Spring

tags:
  - [spring boot, security, UsernameNotFoundException]
---

### **이슈발생**
***

Spring Security를 사용하여 로그인 및 인증 기능을 구현하던 도중에 알 수 없는 오류가 발생했다. 로그인 시에 
아이디가 DB에 존재하지 않을 때 `MemberDetailsService`에서 `UsernameNotFoundException`을 
발생시키고 `SecurityConfig`에서 failureHandler를 통해 <mark>"존재하지 않는 계정입니다. 회원가입 
후 로그인해주세요."</mark> 라는 문구를 출력하게 코드를 작성했다. 하지만 예상과 다르게 비밀번호가 맞지 않을 
때 출력되는 문구인 <mark>"비밀번호가 맞지 않습니다. 다시 확인해주세요."</mark> 문구가 출력되는 것이었다. 다시말해
`UsernameNotFoundException`이 발생해야 하는데 항상 `BadCredentialsException`이 발생한다는 것이다.  

<br/>

**MemberDetailsService**

```java
@RequiredArgsConstructor
@Component
public class MemberDetailsService implements UserDetailsService {

    private final MemberRepository memberRepository;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberRepository.findByUsername(username).orElseThrow(() ->
                new UsernameNotFoundException("NotFoundUserName"));
            //아이디(Username)을 찾을수 없으면 UsernameNotFoundException을 발생시키도록 함
        return new MemberDetails(member);
    }

}
```

**CustomAuthFailureHandler**

```java
@Component
@Slf4j
public class CustomAuthFailureHandler extends SimpleUrlAuthenticationFailureHandler {  //로그인 실패 시 exception관리하는 handler
    
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
                                        AuthenticationException exception) throws IOException, ServletException {

        String errorMessage;

        if(exception instanceof UsernameNotFoundException){
            errorMessage = "존재하지 않는 계정입니다. 회원가입 후 로그인해주세요.";
        } else if (exception instanceof BadCredentialsException) {
            errorMessage = "비밀번호가 맞지 않습니다. 다시 확인해주세요.";
        } else if (exception instanceof InternalAuthenticationServiceException) {
            errorMessage = "내부 시스템 문제로 로그인 요청을 처리할 수 없습니다. 관리자에게 문의하세요.";
        } else if (exception instanceof AuthenticationCredentialsNotFoundException) {
            errorMessage = "인증 요청이 거부되었습니다. 관리자에게 문의하세요.";
        } else {
            errorMessage = "알 수 없는 오류로 로그인 요청을 처리할 수 없습니다. 관리자에게 문의하세요.";
        }

        log.info("failureHandler : " + errorMessage);

        errorMessage = URLEncoder.encode(errorMessage, StandardCharsets.UTF_8);  //한글 인코딩 깨지는 문제 방지
        setDefaultFailureUrl("/login/action?error=true&exception=" + errorMessage);
        super.onAuthenticationFailure(request, response, exception);
    }
}
```
그래서 테스트코드도 돌려보고 인터넷도 찾아보면서 원인과 해결책을 찾았다.

### **원인**
***


인증 프로세스를 처음부터 설명하자면 `AuthenticationManager`에는 인증 프로세스를 처리하는 여러 
`AuthenticationProvider`가 등록된다. 보통 `DaoAuthenticationProvider`가 등록되는데, 이는 
`UserDetailsService`를 사용하여 사용자 정보를 가져오고 비밀번호가 맞는지 검증을 하게된다. 그래서 
처음에 유저가 로그인을 시도하면 `AuthenticationManager`가 호출되고 `DaoAuthenticationProvider`
의 실제 인증을 처리하는 `authenticate`함수를 실행한다. 그리고 `DaoAuthenticationProvider`는 
`UserDetailsService`의 `loadUserByUsername`함수를 호출해 유저 정보를 가져오게 되는데 이때 유저의 정보가
DB상에 존재하지 않아 `UsernameNotFoundException`이 발생하게 되면
`AbstractUserDetailsAuthenticationProvider`의 `hideUserNotFoundExceptions`값을 확인한다.
그래서 만약 위 값이 true로 설정되어 있다면 `BadCredentialsException`을 발생시키게 된다.
<br/>

**AbstractUserDetailsAuthenticationProvider**

```java
public abstract class AbstractUserDetailsAuthenticationProvider
  implements AuthenticationProvider, InitializingBean, MessageSourceAware { 
    
    ...
    
    protected boolean hideUserNotFoundExceptions = true;
    
    ...

    @Override 
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication, 
          () -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports", 
            "Only UsernamePasswordAuthenticationToken is supported"));
        String username = determineUsername(authentication);
        boolean cacheWasUsed = true;
        UserDetails user = this.userCache.getUserFromCache(username);
        if (user == null) {
            cacheWasUsed = false;
            try {
                user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
            } 
            catch (UsernameNotFoundException ex) {
                this.logger.debug("Failed to find user '" + username + "'");
                if (!this.hideUserNotFoundExceptions) {  //값이 true일때 BadCredentialsException을 발생시킴
                    throw ex;
                }
                throw new BadCredentialsException(this.messages
                  .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
            }
            Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
        }
        try {
            this.preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
        } 
        catch (AuthenticationException ex) {
            if (!cacheWasUsed) {
                throw ex;
            }
            // There was a problem, so try again after checking
          // we're using latest data (i.e. not from the cache)
          cacheWasUsed = false;
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
            this.preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
        }
        this.postAuthenticationChecks.check(user);
        if (!cacheWasUsed) {
            this.userCache.putUserInCache(user);
        }
        Object principalToReturn = user;
        if (this.forcePrincipalAsString) {
            principalToReturn = user.getUsername();
        }
        return createSuccessAuthentication(principalToReturn, authentication, user);
    }
    
    ...
  
    public void setHideUserNotFoundExceptions(boolean hideUserNotFoundExceptions) {
        this.hideUserNotFoundExceptions = hideUserNotFoundExceptions;
    }
}

```

그 이유를 찾아보니 Spring Security에서 보안상의 이유로 사용자가 존재하는지 여부를 외부에 노출하지 않기 
위해 동일한 응답을 주는 것 이라고 한다. `UsernameNotFoundException`을 그대로 노출하면, 서버에 
등록된 유저의 아이디를 확인하는 데 도움을 얻을 수 있을 수 있어서 항상 `BadCredentialsException`을
발생시키는 것이다.

### **해결방법**
***

`AbstractUserDetailsAuthenticationProvider`의 `hideUserNotFoundExceptions`값을 false
로 설정하게 된다면 유저의 아이디를 찾을 수 없는 경우에 `BadCredentialsException` 예외를 발생시키지
않고 `UsernameNotFoundException` 예외를 발생시켜 클라이언트에게 반환될 것이다. 그러므로 
`SecurityConfig`클래스에서 `DaoAuthenticationProvider`를 정의 해서 Bean으로 등록시키면 해결
될 것이다.
<br/>

**SecurityConfig**

```java
@RequiredArgsConstructor
@Configuration
@EnableWebSecurity //security filter등록
@EnableGlobalMethodSecurity(prePostEnabled = true) //특정 페이지에 특정권한이 있는 유저만 접근을 허용할 경우 권한 및 인증을 미리 체크하겠다는 설정을 활성화
public class SecurityConfig {
    
    ...
  
    private final MemberDetailsService memberDetailsService;
  
    ...
  
    @Bean
    AuthenticationManager authenticationManager(
            AuthenticationConfiguration authenticationConfiguration, AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(daoAuthenticationProvider());
        return authenticationConfiguration.getAuthenticationManager();
    }
    
    public DaoAuthenticationProvider daoAuthenticationProvider() { //아이디가 맞지 않을 때 UserNotFoundException을 발생하게 하는 메소드
        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();
        authenticationProvider.setUserDetailsService(memberDetailsService);
        authenticationProvider.setPasswordEncoder(bCryptPasswordEncoder());
        authenticationProvider.setHideUserNotFoundExceptions(false);
        return authenticationProvider;
    }
    
    ...

}
```
daoAuthenticationProvider() 메소드를 생성해 새로운 객체를 만들고 `memberDetailsService`와 
`BCryptPasswordEncoder`를 등록한다. 그리고 `HideUserNotFoundException`의 값을 false로 
설정한 뒤에 `AuthenticationManager`에 Bean으로 등록해주면 된다.

<br/>

### **결과화면**
***

![이미지](/assets/img/2023-08-22-springexception/usernotfoundexception.png "존재하지 않는 아이디 문구")
존재하지 않는 아이디를 입력했을 때에 맞는 경고 문구가 뜬다.
