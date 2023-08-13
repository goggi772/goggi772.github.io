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
<br/>
그 다음 db연동을 해준다. 내가 학교에서 데이터베이스 과목을 수강할 때 postgresql을 사용해서 익숙하기
때문에 이 프로젝트에서는 postgresql을 사용했다. 연동하는 법은 먼저 postgresql을 설치하고 dependencies
에 postgres jdbc 드라이버를 설치한 뒤 `application.properties`파일에 위 코드를 추가한다.    

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/데이터베이스이름
spring.datasource.username=사용자이름
spring.datasource.password=비밀번호
```

생성한 데이터베이스와 사용자이름, 비밀번호를 입력만 하면 된다. 그 후에 실행하고 localhost:8080으로
접속하게 되면 밑의 사진과 같은 화면이 나온다.  
<br/>

![이미지](/assets/img/2023-08-11-securitylogin/securityloginscreen.png "로그인 화면")

<br/>
이 화면이 Spring Security에서 제공하는 기본 로그인 화면이다. 주소창의 localhost:8080뒤에 어떤 것을
쳐도 이와 같은 로그인 화면이 나올 것이다. 그 이유는 Spring Security가 기본적으로 모든 페이지에 대해 
권한을 요구하는 것이기 때문이다. 즉, 페이지에 대한 권한을 설정하지 않아서 Spring Security가 요청을
가로채 Spring Security login화면으로 redirection시키기 때문이다. 그래서 우리가 직접 각 페이지에
대해 요구하는 권한이나 로그인 방법, 성공 및 실패 핸들러 등을 설정해주어야 한다. Spring Security의
빈을 설정해주는 클래스 `SecurityConfig`를 만들어 로그인 로직 등을 커스텀하여 사용할 수 있다.  

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
                .loginProcessingUrl("/login/action")   //로그인 로직 url
                .defaultSuccessUrl("/")     //로그인 성공시 url
                .failureHandler(customAuthFailureHandler)      // 실패시 요청을 처리할 핸들러
                .and()
                .logout()
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout")) // 로그아웃 URL
                .logoutSuccessUrl("/"); // 로그아웃 성공시 url

        return http.build();

    }
}
```
<br/>

