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

Repository
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

DetailsService
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

이렇게 `UserDetailsService`를 상속받는 클래스는 `loadUserByUsername`을 override하게 되는데
이는 Spring Security가 유저 아이디(username)을 이용해 유저를 찾고 그 유저에 대한 UserDetails를 
반환하게 된다. 이때 유저가 존재하지 않으면 `UsernameNotFoundException`을 발생시키게 된다.


![이미지](/assets/img/2023-08-11-securitylogin/securityloginscreen.png "로그인 화면")

<br/>
이 화면이 Spring Security에서 제공하는 기본 로그인 화면이다. 주소창의 localhost:8080뒤에 어떤 것을
쳐도 이와 같은 로그인 화면이 나올 것이다. 그 이유는 Spring Security가 기본적으로 모든 페이지에 대해 
권한을 요구하는 것이기 때문이다. 즉, 페이지에 대한 권한을 설정하지 않아서 Spring Security가 요청을
가로채 Spring Security login화면으로 redirection시키기 때문이다. 그래서 우리가 직접 각 페이지에
대해 요구하는 권한이나 로그인 방법, 성공 및 실패 핸들러 등을 설정해주어야 한다. Spring Security의
빈을 설정해주는 클래스 `SecurityConfig`를 만들어 로그인 로직 등을 커스텀하여 사용할 수 있다.  

로그인기능을 구현하기 위해서는 데이터베이스 설정과 서비스 설정을 해야한다.
Entity, Repository, DetailsService를 생성하면 된다.




<br/>

이렇게 작성해 주면 Member이름을 가진 테이블이 생성되고 JPA를 사용하는 Repository에서 한줄의 명령어
로 데이터를 가져오고 수정할 수 있게된다. 

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
                .antMatchers("/", "/login/**", "/main/**", "/js/**",
                  "/css/**", "/image/**").permitAll()   //위 경로는 인증필요 X
                .anyRequest()  //그 외의 다른 요청들은
                .authenticated()    //인증 필요
                .and()
                .formLogin()   //formLogin형식 사용
                .loginPage("/login")  //커스텀 로그인 화면
                .defaultSuccessUrl("/")     //로그인 성공시 url
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
.loginPage("/login")의 코드가 내가 만들어 놓은 로그인 페이지를 알려주는 것이다. 그래서 controller
에 위 경로를 지정해주면 커스텀한 로그인 화면을 사용할 수 있다.

```java

@Controller
public class MemberLoginController {

    @GetMapping("/login")
    public String login_page() {
        return "login";
    }
}
    
```

thymeleaf로 작성한 html 로그인 화면

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

![이미지](/assets/img/2023-08-11-securitylogin/loginpage.png "커스텀 로그인 화면")


