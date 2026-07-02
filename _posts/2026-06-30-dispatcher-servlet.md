---
title: "Spring Boot 요청 처리 과정 완벽 이해 - DispatcherServlet을 중심으로"
date: 2026-06-30 10:30:00 +0900
categories: [Spring, Spring Boot]
tags: [spring, springboot, dispatcher-servlet, mvc, backend]
---

# Spring Boot 요청 처리 과정 완벽 이해 - DispatcherServlet을 중심으로

Spring Boot로 REST API를 개발하다 보면 `@RestController`, `@GetMapping`과 같은 어노테이션은 자연스럽게 사용하게 됩니다.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto findById(@PathVariable Long id) {
        ...
    }
}
```

하지만 이 메서드는 **어떻게 호출되는 것일까요?**

브라우저에서 요청을 보내면 Spring은 어떤 과정을 거쳐 Controller를 실행하고 응답을 반환할까요?

이번 글에서는 **Spring MVC의 핵심 컴포넌트인 `DispatcherServlet`을 중심으로 HTTP 요청이 처리되는 과정**을 단계별로 살펴보겠습니다.

---

# Spring MVC의 핵심, DispatcherServlet

Spring MVC에서는 모든 HTTP 요청이 하나의 Servlet을 먼저 통과합니다.

그 역할을 하는 것이 바로 **DispatcherServlet**입니다.

```text
Client
   │
HTTP Request
   │
DispatcherServlet
   │
Controller
```

DispatcherServlet은 **Front Controller Pattern**을 구현한 객체입니다.

즉, 모든 요청을 하나의 진입점에서 받아 적절한 Controller로 전달하는 역할을 수행합니다.

만약 DispatcherServlet이 없다면 각각의 Controller가 직접 URL을 처리해야 하므로 관리가 매우 어려워집니다.

---

# 전체 요청 처리 흐름

Spring Boot에서 요청은 다음과 같은 순서로 처리됩니다.

```text
Client
   │
   ▼
Embedded Tomcat
   │
   ▼
DispatcherServlet
   │
   ▼
HandlerMapping
   │
   ▼
HandlerAdapter
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
Response
```

이제 각 단계를 자세히 살펴보겠습니다.

---

# 1. Client가 HTTP 요청을 보낸다

예를 들어 다음과 같은 요청이 들어왔다고 가정합니다.

```http
GET /users/1
```

브라우저 또는 API Client(Postman 등)가 요청을 전송하면

```text
Browser

↓

HTTP Request

↓

Embedded Tomcat
```

으로 전달됩니다.

Spring Boot는 별도의 WAS를 설치하지 않아도 **Embedded Tomcat**이 실행되어 요청을 받아줍니다.

---

# 2. DispatcherServlet이 요청을 받는다

Tomcat은 모든 요청을 DispatcherServlet에게 전달합니다.

```text
GET /users/1

↓

DispatcherServlet
```

DispatcherServlet은 요청을 분석한 후

* 어떤 Controller가 처리해야 하는지
* 어떤 메서드를 호출해야 하는지

를 찾기 시작합니다.

---

# 3. HandlerMapping이 Controller를 찾는다

DispatcherServlet은 **HandlerMapping**에게 요청을 전달합니다.

```text
GET /users/1

↓

HandlerMapping
```

HandlerMapping은 등록된 URL 정보를 검색합니다.

예를 들어

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto findById(...) {
    }

}
```

라면

```text
/users/{id}

↓

UserController.findById()
```

를 찾아 DispatcherServlet에게 반환합니다.

---

# 4. HandlerAdapter가 Controller를 실행한다

Controller를 찾았다고 해서 바로 실행되는 것은 아닙니다.

Spring은 **HandlerAdapter**를 통해 Controller를 실행합니다.

```text
DispatcherServlet

↓

HandlerAdapter

↓

UserController.findById()
```

HandlerAdapter는 다음과 같은 작업을 수행합니다.

* PathVariable 변환
* RequestParam 변환
* RequestBody 변환
* HttpServletRequest 주입
* HttpServletResponse 주입

예를 들어

```java
@GetMapping("/{id}")
public UserDto findById(@PathVariable Long id)
```

에서 URL의 `"1"`이라는 문자열을 `Long` 타입으로 변환하는 작업도 HandlerAdapter가 담당합니다.

---

# 5. Controller가 Service를 호출한다

이제 Controller가 실행됩니다.

```java
@GetMapping("/{id}")
public UserDto findById(@PathVariable Long id) {
    return userService.findById(id);
}
```

Controller의 역할은 단순합니다.

* 요청 수신
* Service 호출
* 결과 반환

비즈니스 로직은 Controller에서 수행하지 않는 것이 일반적입니다.

---

# 6. Service가 비즈니스 로직을 수행한다

Service에서는 실제 업무 로직을 처리합니다.

```java
@Service
public class UserService {

    public UserDto findById(Long id) {
        ...
    }

}
```

예를 들어

```text
회원 조회

↓

권한 검사

↓

데이터 가공

↓

Repository 호출
```

과 같은 작업이 수행됩니다.

---

# 7. Repository가 Database를 조회한다

Repository는 데이터베이스와 통신합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

흐름은 다음과 같습니다.

```text
Service

↓

Repository

↓

JPA

↓

Database
```

Database에서 조회한 결과는 다시 Service로 전달됩니다.

---

# 8. 응답 객체를 반환한다

Service의 결과는 Controller로 돌아옵니다.

```java
return userDto;
```

`@RestController`를 사용하고 있다면 Spring은 객체를 자동으로 JSON으로 변환합니다.

```json
{
  "id": 1,
  "name": "Kim",
  "email": "kim@example.com"
}
```

이 변환은 **HttpMessageConverter**가 담당합니다.

대표적으로 **Jackson** 라이브러리를 사용하여 Java 객체를 JSON으로 직렬화합니다.

---

# DispatcherServlet 내부에서 실제로 일어나는 일

요청 하나가 처리되는 과정을 간단하게 표현하면 다음과 같습니다.

```text
HTTP Request
      │
      ▼
DispatcherServlet
      │
      ▼
HandlerMapping
      │
      ▼
HandlerAdapter
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
Controller
      │
      ▼
HttpMessageConverter
      │
      ▼
JSON Response
```

DispatcherServlet은 이 전체 과정을 조율하는 **중앙 관리자(Coordinator)** 역할을 수행합니다.

---

# 왜 DispatcherServlet이 중요한가?

DispatcherServlet이 존재하기 때문에 Spring MVC는 다음과 같은 장점을 얻습니다.

* 모든 요청을 하나의 진입점에서 처리할 수 있다.
* URL과 Controller를 자동으로 연결할 수 있다.
* 요청 파라미터를 자동으로 변환할 수 있다.
* 예외를 일관성 있게 처리할 수 있다.
* 응답을 JSON 또는 View로 자동 변환할 수 있다.
* 공통 기능(로그, 인증, 인터셉터 등)을 쉽게 추가할 수 있다.

즉, DispatcherServlet은 **Spring MVC의 핵심 제어자(Front Controller)**로서 요청 처리의 시작과 끝을 책임지는 중요한 컴포넌트입니다.

---

# 마무리

Spring Boot에서 HTTP 요청은 단순히 Controller가 직접 받는 것이 아닙니다.

먼저 DispatcherServlet이 요청을 수신하고, HandlerMapping을 통해 적절한 Controller를 찾은 뒤, HandlerAdapter를 이용해 메서드를 실행합니다. 이후 Service와 Repository를 거쳐 데이터를 처리하고, 마지막으로 HttpMessageConverter가 응답 객체를 JSON으로 변환하여 클라이언트에 반환합니다.

이러한 요청 처리 흐름을 이해하면 Spring Boot의 내부 동작 원리를 보다 명확하게 파악할 수 있으며, 예외 처리, 인터셉터, 필터, Spring Security와 같은 고급 기능도 훨씬 쉽게 이해할 수 있습니다.
