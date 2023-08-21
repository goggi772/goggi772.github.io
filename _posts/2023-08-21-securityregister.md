---
layout: post
title: "Spring Security로 로그인구현하기(2)"

categories:
  - Spring

tags:
  - [spring boot, security]


---

앞서 로그인을 구현하는 방법을 살펴보았다. 하지만 로그인을 하기 위해서는 회원가입이 이루어져야 하므로 
회원가입 로직도 필요하다. 이제 회원가입하는 코드를 작성해보자. 

### 회원가입 화면 
<br/>
먼저 간단하게 회원 가입 화면을 thymeleaf로 짜보았다.

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
    <form th:action="@{/login/register/join}" method="post">
        <div class="form-group">
            <label th:for="username">아이디</label>
            <input type="text" name="username" class="form-control" placeholder="아이디 입력해주세요">
        </div>
        <div class="form-group">
            <label th:for="password">비밀번호</label>
            <input type="password" class="form-control" name="password" placeholder="비밀번호 입력해주세요">
        </div>
        <div class="form-group">
            <label th:for="email">이메일</label>
            <input type="text" class="form-control" name="email" placeholder="비밀번호 입력해주세요">
        </div>
        <button type="submit" class="btn btn-primary">회원가입</button>
    </form>
</div>
</body>
</html>
```
<br/>

![이미지](/assets/img/2023-08-21-securityregister/register.png "회원가입 화면")
<br/>

회원가입 화면을 만들었으니 객체를 전달받을 MemberReigsterDTO를 작성해보았다.

### MemberRegisterDTO

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class MemberRegisterDTO {  //회원가입 DTO

    private Long id;

    private String username;

    private String password;

    private String email;

    private Role role;

    public Member toEntity() {
        return Member.builder()
                .username(username)
                .password(password)
                .email(email)
                .role(Role.USER)
                .build();
    }
}
```
<br/>
회원가입에 필요한 파라미터들을 생성하고 전달받은 파라미터로 Member객체를 생성하는 toEntity()메소드를
작성하여 DTO로 Member객체를 생성할 수 있게 하였다. 이제 서비스로직을 구현해보자.

### MemberService

```java
@RequiredArgsConstructor
@Service
public class MemberService {
    
    private final BCryptPasswordEncoder bCryptPasswordEncoder;

    @Transactional
    public void register(MemberRegisterDTO dto) {  //회원가입 로직(암호화)
        dto.setPassword(bCryptPasswordEncoder.encode(dto.getPassword()));
        memberRepository.save(dto.toEntity());
    }
}
```

MemberRegisterDTO객체를 전달받아 toEntity()메소드로 Member객체를 데이터베이스에 저장하는 방식으로
작성하였다. 이때 데이터베이스에 입력받은 사용자의 암호를 그대로 저장한다면 보안상의 문제가 발생할
수 있다. 그러므로 비밀번호를 해싱하여 암호화한 뒤 저장해야 한다. <br/> 그래서 나는 `BCryptPasswordEncoder`
를 사용하여 비밀번호를 암호화 한뒤 데이터베이스에 저장하였다. 여기서 `BCryptPasswordEncoder`는
`BCrypt`라는 암호화 해싱 함수를 사용한 구현체로 Spring Security에서 제공하는 비밀번호 암호화 
인터페이스이다. 이는 복호화가 불가능한 단방향 해시 함수처럼 특정 값에 대해 항상 동일한 값을 갖는 것이 
아니라 같은 값이라도 매번 다르게 해싱되어 가장 강력한 해시 매커니즘이라고 할 수 있다. <br/> 이 인터페이스를 
사용하려면 SecurityConfig에 Bean으로 등록해주기만 하면 된다.

```java
@RequiredArgsConstructor
@Configuration
@EnableWebSecurity //security filter등록
@EnableGlobalMethodSecurity(prePostEnabled = true) //특정 페이지에 특정권한이 있는 유저만 접근을 허용할 경우 권한 및 인증을 미리 체크하겠다는 설정을 활성화
public class SecurityConfig {

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
<br/>

이제 마지막으로 회원가입을 진행하기 위한 Controller를 작성해보자.

### Controller

```java
@RequiredArgsConstructor
@Controller
public class MemberLoginController {
    
    private final MemberService memberService;

    @GetMapping("/login/register")
    public String register() {
        return "register";
    }

    @PostMapping("/login/register/join")  //회원가입
    public String join(@ModelAttribute MemberRegisterDTO dto) {
        memberService.register(dto);
        return "redirect:/login";
    }

}
```

html에서 form action에 입력된 경로로 Mapping해주고 MemberRegisterDTO객체로 받은 뒤 memberService
에서 생성한 register메소드를 실행하며 회원가입이 완료되면 로그인화면으로 redirect시켜준다.

### 결과화면

![이미지](/assets/img/2023-08-21-securityregister/registerbefore_db.png "가입전 db")
회원가입 전 DB
<br/>

![이미지](/assets/img/2023-08-21-securityregister/input_register.png "회원가입 화면 입력")

<br/>
아이디가 user이고 비밀번호가 1234, 이메일이 zxcv@gmail.com인 유저 회원가입

![이미지](/assets/img/2023-08-21-securityregister/registerafter_db.png "가입 후 db")
유저의 비밀번호가 암호화 되어 Member테이블에 저장된 것을 볼 수 있다.
