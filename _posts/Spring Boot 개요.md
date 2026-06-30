---
title: "Spring Boot의 구조와 동작 원리"
date: 2026-06-30 10:00:00 +0900
categories: [Spring, Spring Boot]
tags: [spring, springboot, architecture, mvc, backend]
---

# Spring Boot의 구조와 동작 원리

Spring Boot는 Spring Framework를 기반으로 애플리케이션을 빠르게 개발할 수 있도록 도와주는 프레임워크입니다.

이번 글에서는 Spring Boot 프로젝트의 구조와 요청이 처리되는 전체 흐름을 살펴보겠습니다.

---

## Spring Boot 프로젝트 구조

일반적인 프로젝트는 다음과 같은 구조를 가집니다.

```text
my-project
│
├── src
│   ├── main
│   │   ├── java
│   │   │    └── com.example.demo
│   │   │         ├── DemoApplication.java
│   │   │         ├── controller
│   │   │         ├── service
│   │   │         ├── repository
│   │   │         ├── entity
│   │   │         ├── dto
│   │   │         ├── config
│   │   │         └── exception
│   │   │
│   │   └── resources
│   │        ├── application.yml
│   │        ├── static
│   │        ├── templates
│   │        └── schema.sql
│   │
│   └── test
│
├── build.gradle
└── settings.gradle
```

---

# 계층(Layer) 구조

Spring Boot는 일반적으로 다음과 같은 계층 구조를 사용합니다.

```text
Controller
    │
    ▼
Service
    │
    ▼
Repository
    │
    ▼
Database
```

각 계층은 하나의 책임만 가지도록 설계하는 것이 중요합니다.

---

## Controller

Controller는 클라이언트의 HTTP 요청을 가장 먼저 받는 계층입니다.

### 역할

- URL 매핑
- 요청(Request) 처리
- 응답(Response) 반환
- Service 호출

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService){
        this.userService = userService;
    }

    @GetMapping
    public List<UserDto> findAll(){
        return userService.findAll();
    }
}
```

---

## Service

Service는 비즈니스 로직을 담당합니다.

예를 들어 회원가입이라면

```text
아이디 중복 확인
      │
비밀번호 암호화
      │
회원 저장
      │
이메일 발송
```

와 같은 작업이 Service에서 수행됩니다.

```java
@Service
public class UserService {

    public List<UserDto> findAll(){
        ...
    }

}
```

---

## Repository

Repository는 데이터베이스와 직접 통신합니다.

Spring Data JPA를 사용하는 경우 대부분 다음과 같이 작성합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

### 역할

- CRUD
- SQL 실행
- 데이터 조회 및 저장

---

## Entity

Entity는 데이터베이스 테이블과 매핑됩니다.

```java
@Entity
public class User {

    @Id
    private Long id;

    private String name;

    private String email;

}
```

예를 들어

| Database | Java |
|----------|------|
| USER | User.java |

---

## DTO (Data Transfer Object)

DTO는 계층 간 데이터를 전달하기 위한 객체입니다.

Entity를 그대로 외부에 노출하지 않고 DTO를 사용하는 것이 일반적입니다.

```java
public class UserResponse {

    private String name;

    private String email;

}
```

---

## Config

애플리케이션 설정을 담당하는 계층입니다.

대표적으로

- Spring Security
- CORS
- Swagger
- Bean 등록

등을 관리합니다.

```java
@Configuration
public class SecurityConfig {

}
```

---

# Spring Boot 실행 과정

Spring Boot는 다음 순서로 실행됩니다.

```text
main()

↓

SpringApplication.run()

↓

ApplicationContext 생성

↓

Bean 등록

↓

Embedded Tomcat 실행

↓

HTTP 요청 대기
```

메인 클래스는 다음과 같습니다.

```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

---

# HTTP 요청 처리 과정

예를 들어

```
GET /users
```

요청이 들어오면 다음 순서로 처리됩니다.

```text
Client
    │
HTTP Request
    │
DispatcherServlet
    │
HandlerMapping
    │
Controller
    │
Service
    │
Repository
    │
Database
    │
Repository
    │
Service
    │
Controller
    │
JSON Response
```

---

# DispatcherServlet

Spring MVC의 핵심 컴포넌트입니다.

모든 HTTP 요청은 DispatcherServlet을 먼저 통과합니다.

```text
Client

↓

DispatcherServlet

↓

Controller
```

DispatcherServlet은 **Front Controller 패턴**을 구현한 객체입니다.

---

# HandlerMapping

요청 URL에 맞는 Controller를 찾아주는 역할을 합니다.

예를 들어

```text
GET /users
```

↓

```text
UserController
```

를 찾아 연결합니다.

---

# HandlerAdapter

찾아낸 Controller를 실제로 실행합니다.

```text
UserController.findAll()
```

---

# ViewResolver

Spring MVC에서 화면(View)을 반환할 때 사용됩니다.

```java
@Controller
public class HomeController {

    @GetMapping("/")
    public String home() {
        return "home";
    }

}
```

반환된 문자열

```text
home
```

은

```text
templates/home.html
```

과 연결됩니다.

REST API에서는 `@RestController`를 사용하므로 ViewResolver를 거의 사용하지 않습니다.

---

# IoC (Inversion of Control)

Spring은 객체 생성을 개발자가 직접 하지 않습니다.

기존 Java에서는

```java
UserService service = new UserService();
```

처럼 객체를 생성하지만

Spring에서는

```java
@Service
public class UserService {

}
```

만 선언하면 Spring Container가 객체를 생성하고 관리합니다.

이를 **IoC(Inversion of Control)** 라고 합니다.

---

# DI (Dependency Injection)

Spring은 필요한 객체를 자동으로 주입합니다.

```java
@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

}
```

`UserRepository`는 개발자가 생성하지 않아도 Spring이 자동으로 주입합니다.

이를 **DI(Dependency Injection)** 라고 합니다.

---

# Bean

Spring이 생성하여 관리하는 객체를 Bean이라고 합니다.

대표적인 Bean 등록 어노테이션은 다음과 같습니다.

```java
@Component
@Service
@Repository
@Controller
@Configuration
```

생성된 Bean은 **ApplicationContext**에서 관리됩니다.

---

# 전체 구조

```text
                  Spring Boot

            SpringApplication.run()
                     │
                     ▼
             ApplicationContext
                     │
            (Bean 생성 및 관리)
                     │
             Embedded Tomcat
                     │
               HTTP Request
                     │
                     ▼
            DispatcherServlet
                     │
            HandlerMapping
                     │
                     ▼
              Controller
                     │
                     ▼
                Service
                     │
                     ▼
              Repository
                     │
                     ▼
                Database
                     │
                     ▼
              JSON Response
```

---

# 마무리

Spring Boot를 처음 학습할 때는 다음 세 가지를 먼저 이해하는 것이 중요합니다.

1. **Controller → Service → Repository**의 계층 구조
2. **DispatcherServlet**을 중심으로 한 요청 처리 과정
3. **IoC / DI / Bean**을 통한 객체 관리 방식

이 세 가지 개념을 이해하면 Spring Boot 애플리케이션의 대부분의 동작 원리를 쉽게 파악할 수 있으며, 이후 Spring MVC, Spring Data JPA, Spring Security 등을 학습하는 기반이 됩니다.