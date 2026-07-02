---
title: "Spring Security 완벽 이해 - 인증(Authentication)과 인가(Authorization)의 시작"
date: 2026-06-30 16:00:00 +0900
categories: [Spring, Spring Boot]
tags: [spring, springboot, spring-security, authentication, authorization, filter]
---

# Spring Security 완벽 이해 - 인증(Authentication)과 인가(Authorization)의 시작

Spring Boot로 웹 애플리케이션이나 REST API를 개발하다 보면 가장 먼저 고려해야 하는 요소 중 하나가 **보안(Security)** 입니다.

로그인 기능을 구현한다고 해서 보안이 끝나는 것은 아닙니다.

* 로그인한 사용자인가?
* 이 사용자가 해당 API를 호출할 권한이 있는가?
* 비밀번호는 안전하게 저장되고 있는가?
* 요청이 위변조되지 않았는가?

이러한 보안 요구사항을 처리하기 위해 Spring에서는 **Spring Security**를 제공합니다.

이번 글에서는 Spring Security가 무엇인지, 왜 필요한지, 그리고 내부적으로 어떻게 요청을 처리하는지 알아보겠습니다.

---

# Spring Security란?

Spring Security는 **Spring 애플리케이션의 인증(Authentication)과 인가(Authorization)를 담당하는 보안 프레임워크**입니다.

다음과 같은 기능을 기본적으로 제공합니다.

* 사용자 인증(Authentication)
* 접근 권한 제어(Authorization)
* 비밀번호 암호화
* 세션 관리
* CSRF(Cross-Site Request Forgery) 보호
* CORS 설정 지원
* Security Filter Chain
* OAuth2 및 JWT 연동

Spring Boot에서는 의존성만 추가하면 기본적인 보안 기능이 자동으로 활성화됩니다.

---

# 인증(Authentication)과 인가(Authorization)의 차이

Spring Security를 이해하려면 먼저 두 개념을 구분해야 합니다.

## 인증(Authentication)

인증은 **"당신은 누구인가?"**를 확인하는 과정입니다.

예를 들어,

```text
아이디 입력

↓

비밀번호 입력

↓

사용자 정보 확인

↓

로그인 성공
```

아이디와 비밀번호를 통해 사용자의 신원을 검증하는 과정이 인증입니다.

---

## 인가(Authorization)

인증이 완료되면 다음은 **"무엇을 할 수 있는가?"**를 확인합니다.

예를 들어,

```text
로그인 성공

↓

관리자인가?

↓

관리자 페이지 접근 허용
```

또는

```text
일반 사용자

↓

관리자 페이지 접근

↓

403 Forbidden
```

처럼 권한을 검사하는 과정이 인가입니다.

---

# Spring Security의 요청 처리 흐름

Spring Security는 Controller보다 먼저 요청을 처리합니다.

전체 흐름은 다음과 같습니다.

```text
Client
   │
HTTP Request
   │
Embedded Tomcat
   │
Security Filter Chain
   │
DispatcherServlet
   │
Controller
   │
Service
   │
Repository
   │
Database
```

여기서 가장 중요한 컴포넌트가 **Security Filter Chain**입니다.

---

# Security Filter Chain

Spring Security는 여러 개의 Filter를 체인 형태로 연결하여 요청을 처리합니다.

```text
HTTP Request

↓

Filter 1

↓

Filter 2

↓

Filter 3

↓

...

↓

DispatcherServlet
```

즉,

Controller가 실행되기 전에 여러 보안 검사를 수행합니다.

예를 들어

* 로그인 여부 확인
* 권한 확인
* JWT 검증
* CSRF 검사

등이 모두 Filter에서 처리됩니다.

---

# 왜 Filter를 사용할까?

Spring MVC의 Interceptor는 DispatcherServlet 이후에 실행됩니다.

반면 Filter는 DispatcherServlet보다 먼저 실행됩니다.

```text
Client

↓

Filter

↓

DispatcherServlet

↓

Interceptor

↓

Controller
```

보안은 애플리케이션 내부 로직이 실행되기 전에 검사해야 하므로 Spring Security는 Filter 기반으로 설계되었습니다.

---

# Security Filter Chain 내부 구조

실제로는 수십 개의 Filter가 순서대로 실행됩니다.

대표적인 흐름은 다음과 같습니다.

```text
SecurityContextHolderFilter

↓

UsernamePasswordAuthenticationFilter

↓

BasicAuthenticationFilter

↓

ExceptionTranslationFilter

↓

AuthorizationFilter

↓

DispatcherServlet
```

모든 Filter를 직접 사용할 필요는 없지만, 이러한 구조를 이해하면 Spring Security의 동작 원리를 파악하는 데 도움이 됩니다.

---

# 인증 과정

로그인 요청이 들어오면 다음과 같은 과정이 수행됩니다.

```text
로그인 요청

↓

UsernamePasswordAuthenticationFilter

↓

AuthenticationManager

↓

AuthenticationProvider

↓

UserDetailsService

↓

Database
```

사용자 정보를 조회한 후 비밀번호를 비교하여 인증 결과를 반환합니다.

---

# UserDetailsService

사용자 정보를 조회하는 역할을 합니다.

```java
@Service
public class CustomUserDetailsService
        implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) {

        ...

    }

}
```

Spring Security는 로그인 시 이 메서드를 호출하여 사용자 정보를 가져옵니다.

---

# PasswordEncoder

비밀번호는 절대로 평문으로 저장하면 안 됩니다.

Spring Security는 다양한 암호화 알고리즘을 제공합니다.

대표적으로 `BCryptPasswordEncoder`를 사용합니다.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

회원가입 시에는

```java
String encoded =
    passwordEncoder.encode(password);
```

로그인 시에는

```java
passwordEncoder.matches(
    rawPassword,
    encodedPassword
);
```

를 통해 비밀번호를 비교합니다.

---

# SecurityContext

인증이 성공하면 사용자 정보는 **SecurityContext**에 저장됩니다.

```text
로그인 성공

↓

Authentication 생성

↓

SecurityContext 저장
```

이후에는 현재 로그인한 사용자를 언제든 조회할 수 있습니다.

```java
Authentication authentication =
    SecurityContextHolder
        .getContext()
        .getAuthentication();
```

---

# Spring Security 설정

Spring Boot 3.x에서는 `SecurityFilterChain` Bean을 등록하여 보안 정책을 설정합니다.

```java
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http)
            throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }

}
```

위 설정은 다음과 같은 의미를 가집니다.

* `/login`은 누구나 접근 가능
* `/admin/**`는 `ADMIN` 권한 필요
* 나머지 요청은 로그인 필요

---

# JWT와 Spring Security

최근 REST API에서는 Session보다 JWT를 많이 사용합니다.

요청 흐름은 다음과 같습니다.

```text
로그인

↓

JWT 발급

↓

Client 저장

↓

API 요청

↓

Authorization Header

↓

JWT 검증

↓

Controller
```

JWT 인증도 Security Filter Chain 내부에서 처리됩니다.

즉,

```text
Request

↓

JWT Filter

↓

인증 성공

↓

Controller
```

와 같은 구조로 동작합니다.

---

# Spring Security를 사용하는 이유

Spring Security를 사용하면 다음과 같은 장점을 얻을 수 있습니다.

* 인증과 인가를 일관성 있게 처리할 수 있다.
* 검증된 보안 기능을 활용할 수 있다.
* 다양한 인증 방식(Session, JWT, OAuth2)을 지원한다.
* 비밀번호 암호화를 쉽게 적용할 수 있다.
* Filter 기반으로 요청을 안전하게 보호할 수 있다.
* Spring Boot와 자연스럽게 통합된다.

특히 대부분의 기업 프로젝트에서는 Spring Security를 기반으로 JWT 또는 OAuth2 인증을 구현하는 경우가 많습니다.

---

# 마무리

Spring Security는 단순히 로그인 기능을 제공하는 라이브러리가 아니라 **Spring 애플리케이션 전체의 보안을 담당하는 핵심 프레임워크**입니다.

모든 요청은 먼저 **Security Filter Chain**을 통과하며, 이 과정에서 인증(Authentication), 인가(Authorization), 권한 검사, 예외 처리 등이 수행됩니다. 인증이 완료된 사용자 정보는 `SecurityContext`에 저장되어 애플리케이션 전반에서 활용됩니다.

Spring Security를 제대로 이해하려면 단순히 설정 방법을 외우기보다 **"요청이 Filter를 거쳐 인증되고, 권한을 확인한 후 DispatcherServlet으로 전달된다"**는 전체 흐름을 이해하는 것이 중요합니다.

다음 글에서는 **JWT(Json Web Token)를 이용한 인증 방식**을 중심으로, Spring Security와 JWT가 어떻게 연동되어 Stateless 인증을 구현하는지 자세히 살펴보겠습니다.
