---
layout: post
title: "Spring Security로 login구현하기"
---

이번에 Spring Boot도 공부할 겸 간단한 프로젝트를 만들었다. 그 중에서 가장 기본이 되는 로그인 기능을 
Spring Security를 이용하여 구현해 보았다. Spring Security로 구현한 로그인, 회원가입, 예외처리 등의 
기능들을 살펴보자.

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
<br/>
그 다음 db연동을 한다. 내가 학교에서 데이터베이스 과목을 수강할 때 postgre SQL을 사용해서 익숙하기
때문에 이 프로젝트에서는 postgresql을 사용했다. 연동하는 법은 먼저 postgresql을 설치하고 dependencies
에 postgres jdbc 드라이버를 설치한 뒤 `application.properties`파일에 위 코드를 추가한다.    

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/데이터베이스이름
spring.datasource.username=사용자이름
spring.datasource.password=비밀번호
```

생성한 데이터베이스와 사용자이름, 비밀번호를 입력만 하면 된다. 
<br/>

이제 유저 Entity를 만들어보자.

### Entity

```java
@Entity
@Getter
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;   //회원 고유 number

    @Column(nullable = false, length = 30, unique = true)
    private String username;  //회원 id

    @Column(nullable = false, length = 100)
    private String password;   //비밀번호

    @Column(nullable = false, length = 50)
    private String email;  //이메일

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;
}
```

나는 이렇게 id, username, password, email, role로 Entity를 만들었다. 이렇게 Entity 클래스를
만들고 @Entity 어노테이션을 쓰기만 하면 실행할 때 자동으로 클래스 이름을가진 테이블이 생성된다. 
그 다음 Repository를 생성해보자.

### Repository
```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    Optional<Member> findByUsername(String username);
    
}
```

이는 Primary Key의 데이터 타입이 Long이고 레포지토리를 사용할 객체가 Member인 JPARepository를
사용한다는 것이다. 그리고 위와같이 findBy속성 을 작성해서 사용하게 되면 SQL문을 작성하지 않더라도
데이터베이스로부터 정보를 가져올 수 있다.  
<br/>
이제 Spring Security에서 사용하는 DetailsService를 작성해보자.

### DetailsService
```java
@RequiredArgsConstructor
@Component
public class MemberDetailsService implements UserDetailsService {

    private final MemberRepository memberRepository;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberRepository.findByUsername(username).orElseThrow(() ->
                new UsernameNotFoundException("NotFoundUserName"));
        return new MemberDetails(member);
    }

}
```

이와같이 `UserDetailsService`를 상속받는 클래스는 `loadUserByUsername`을 override하게 되는데
이는 Spring Security가 유저 아이디(username)을 이용해 유저를 찾고 그 유저에 대한 UserDetails를 
반환하는 메소드이다. 이때 유저가 존재하지 않으면 `UsernameNotFoundException`을 발생시키게 된다.
여기서는 `MemberDetails`클래스를 리턴하는데 이 클래스는 `UserDetails`를 상속받은 클래스이다.

### MemberDetails
```java
@AllArgsConstructor
public class MemberDetails implements UserDetails {
    
    private final Member member;

    @Override //계정이 갖고있는 권한 목록을 리턴한다
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Collection<GrantedAuthority> collectors = new ArrayList<>();

        collectors.add(() -> "ROLE_" + member.getRole());

        return collectors;
    }

    @Override
    public String getPassword() {
        return member.getPassword();
    }

    @Override
    public String getUsername() {
        return member.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}

```

<br/>
기본적으로 Spring Security에서 유저 정보를 담는 객체인 `UserDetails`가 있는데 이를 클래스에 상속
시켜 사용하면 내가 임의대로 유저 정보를 담는 객체를 커스텀 할 수 있다.  
<br/>
그 후 Spring Security에서 여러 기능들을 커스텀 할 수 있는 `SecurityConfig`를 생성하였다.
`SecurityConfig`가 무엇인지 설명해보자면 처음에 Spring Security를 추가하고 실행시키면
아무 코드도 짜지 않았지만 밑의 사진과 같이 로그인 화면이 나오게 된다.


![이미지](/assets/img/2023-08-11-securitylogin/securityloginscreen.png "로그인 화면")

<br/>
이 화면은 Spring Security에서 제공하는 기본 로그인 화면이다. 주소창의 localhost:8080뒤에 어떤 것을
쳐도 이와 같은 로그인 화면이 나올 것이다. 그 이유는 Spring Security가 기본적으로 모든 페이지에 대해 
권한을 요구하는 것이기 때문이다. 즉, 페이지에 대한 권한을 설정하지 않아서 Spring Security가 요청을
가로채 Spring Security login화면으로 redirection시킨다. 그래서 우리가 직접 각 페이지에
대해 요구하는 권한이나 로그인 방법, 성공 및 실패 핸들러 등을 설정해주어야 한다. 이 설정을 관리하는 클래스
가 `SecurityConfig`이다. `SecurityConfig`는 Spring Security의 빈을 설정해주는 클래스를 만들어 
로그인 로직 등을 커스텀하여 사용할 수 있다.
<br/>

### SecurityConfig
```java
@RequiredArgsConstructor
@Configuration
@EnableWebSecurity //security filter등록
@EnableGlobalMethodSecurity(prePostEnabled = true) //특정 페이지에 특정권한이 있는 유저만 
// 접근을 허용할 경우 권한 및 인증을 미리 체크하겠다는 설정을 활성화
public class SecurityConfig {
    
    private final CustomAuthFailureHandler customAuthFailureHandler;
    
    @Bean   //AuthenticationManager Bean 등록
    AuthenticationManager authenticationManager(
            AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http.csrf().disable()
                .authorizeRequests()  
                .antMatchers("/login/**", "/main/**", "/js/**",
                  "/css/**", "/image/**").permitAll()   //위 경로는 인증필요 X
                .anyRequest()  //그 외의 다른 요청들은
                .authenticated()    //인증 필요
                .and()
                .formLogin()   //formLogin형식 사용
                .loginPage("/login")  //커스텀 로그인 화면
                .loginProcessingUrl("/login/action")   //로그인 실패 시 error, exception 파라미터 전송
                .defaultSuccessUrl("/")     //로그인 성공시 url
                .failureHandler(customAuthFailureHandler)   // 실패시 요청을 처리할 핸들러
                .and()
                .logout()
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout")) // 로그아웃 URL
                .logoutSuccessUrl("/"); // 로그아웃 성공시 url

        return http.build();

    }
}
```
<br/>

이렇게 커스텀을 해주면 form로그인을 사용하므로 원하는대로 로그인 화면을 바꿀 수 있다. 
loginPage("/login")의 코드가 내가 만들어 놓은 로그인 페이지를 알려주는 것이다. 그래서 Controller
에 위 경로를 지정해주면 커스텀한 로그인 화면을 사용할 수 있다. 또한 
loginProcessingUrl을 지정해주면 로그인을 처리할 Url을 지정할 수 있다. 그리고 failureHandler는
로그인 실패시 요청을 처리할 Handler를 지정해주는 명령어이다. 그래서 커스텀 핸들러 클래스를 만들어 
존재하지 않는 유저이거나 비밀번호가 안맞는 등의 로그인 실패 상황을 처리할 수 있게 만들었다.
<br/>

### CustomAuthFailureHandler
```java
@Component
@Slf4j
public class CustomAuthFailureHandler extends SimpleUrlAuthenticationFailureHandler {  //로그인 실패 시 exception관리하는 handler
    
    /**
     * HttpServletRequest : Request 정보
     * HttpServletResponse : Response에 대해 설정할 수 있는 변수
     * AuthenticationException : 로그인 실패 시 예외에 대한 정보
     */
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
이와 같이 `SimpleUrlAuthenticationFailureHandler`를 상속받는 클래스를 만들게 되면 Spring 
Security에서 로그인에 실패 했을 때 그 핸들러 클래스가 요청을 처리할 수 있게 한다. 그래서 위와같이
`CustomAuthFailureHandler`클래스를 만들어 exception이 발생했을 때 종류별로 에러 메시지를 
"/login/action"으로 보내게 된다.
<br/>

### Controller
```java

@Controller
public class MemberLoginController {

    @GetMapping("/login")
    public String login_page() {
        return "login";
    }

    @GetMapping("/login/action")  //로그인 실패 시 error와 exception 값을 받아 에러메시지 출력
    public String getLoginPage(Model model,
                               @RequestParam(value = "error", required = false) String error,
                               @RequestParam(value = "exception", required = false) String exception) {
        model.addAttribute("error", error);
        model.addAttribute("exception", exception);
        return "login2";
    }
}
```
Controller에서는 `SecurityConfig`에서 설정한 것처럼 로그인 페이지와 로그인을 처리할 Url을 지정해서
로그인 과정이 진행되게 한다. getLoginPage는 위에서 말했듯이 error여부와 exception메시지를 model
로 보내 화면에 띄워준다.
<br/>

### login.html
```html
<!DOCTYPE html>
<!-- Latest compiled and minified CSS -->
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8">
</head>
<body>
<div class="container">
    <h1>로그인</h1>
        <td>
            <span th:if="${error}"
                  th:text="${exception}" style="color:red"></span>
        </td>
    <form th:action="@{/login/action}" method="post">
        <div class="form-group">
            <label th:for="username">아이디</label>
            <input type="text" name="username" class="form-control" placeholder="아이디를 입력해주세요">
        </div>

        <div class="form-group">
            <label th:for="password">비밀번호</label>
            <input type="password" class="form-control" name="password" placeholder="비밀번호를 입력해주세요">
        </div>
        <button type="submit" class="btn btn-primary">로그인</button>
        <button type="button" class="btn btn-primary" onClick="location.href='/login/register'">회원 가입</button>
    </form>
    <br/>
</div>
</body>
</html>
```

화면은 thymeleaf를 사용하였다. 로그인 화면을 띄울 때 에러가 발생하게 되면 exception메시지를 
띄우게 된다. 
<br/>

로그인 화면
![이미지](/assets/img/2023-08-11-securitylogin/loginpage.png "커스텀 로그인 화면")
<br/>

로그인 실패시
![이미지](/assets/img/2023-08-11-securitylogin/loginfailure.png "로그인 실패 화면")
<br/>

로그인 성공
![이미지](/assets/img/2023-08-11-securitylogin/main404.png "로그인 성공시 메인 화면")



