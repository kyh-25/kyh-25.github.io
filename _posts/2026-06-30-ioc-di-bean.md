---
title: "Spring Boot의 핵심 개념: IoC, DI, Bean으로 이해하는 객체 관리"
date: 2026-06-30 11:30:00 +0900
categories: [Spring, Spring Boot]
tags: [spring, springboot, ioc, di, bean, applicationcontext]
---

# Spring Boot의 핵심 개념: IoC, DI, Bean으로 이해하는 객체 관리

Spring Boot를 처음 접하면 다음과 같은 어노테이션을 자주 보게 됩니다.

```java
@Service
public class UserService {

}
```

또는 생성자에서 객체를 직접 생성하지 않았는데도 정상적으로 동작하는 코드를 볼 수 있습니다.

```java
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

여기서 많은 개발자가 처음에는 이런 의문을 갖습니다.

> **"UserService를 생성한 적이 없는데 어떻게 사용할 수 있는 걸까?"**

그 답은 Spring의 핵심 개념인 **IoC(Inversion of Control)**, **DI(Dependency Injection)**, 그리고 **Bean**에 있습니다.

이번 글에서는 이 세 가지 개념이 어떻게 연결되어 객체를 관리하는지 살펴보겠습니다.

---

# 객체를 직접 생성하는 일반적인 Java 프로그램

Spring을 사용하지 않는 일반적인 Java 애플리케이션에서는 필요한 객체를 직접 생성합니다.

```java
public class UserService {

}
```

```java
public class UserController {

    private final UserService userService = new UserService();

}
```

객체를 사용하는 개발자가 객체 생성도 직접 책임집니다.

흐름을 그림으로 표현하면 다음과 같습니다.

```text
UserController

↓

new UserService()

↓

객체 생성
```

이 방식은 간단하지만 프로젝트가 커질수록 여러 문제가 발생합니다.

* 객체 간 결합도가 높아진다.
* 객체 생성 방식이 변경되면 사용하는 모든 코드를 수정해야 한다.
* 테스트를 위한 Mock 객체를 주입하기 어렵다.
* 객체 생명주기를 일관성 있게 관리하기 어렵다.

이러한 문제를 해결하기 위해 Spring은 객체 관리의 책임을 프레임워크가 대신 맡도록 설계되었습니다.

---

# IoC(Inversion of Control)란?

IoC는 **제어의 역전(Inversion of Control)** 을 의미합니다.

기존에는 개발자가 객체를 생성하고 관리했습니다.

```text
개발자

↓

new UserService()

↓

객체 생성
```

하지만 Spring에서는 객체 생성의 책임이 Spring Container로 넘어갑니다.

```text
Spring Container

↓

객체 생성

↓

개발자에게 제공
```

즉,

> **객체 생성과 생명주기의 제어권이 개발자에서 Spring으로 이동한 것**이 IoC입니다.

---

# Spring Container란?

IoC를 실제로 수행하는 주체가 바로 **Spring Container**입니다.

Spring Boot가 실행되면 다음 코드가 가장 먼저 실행됩니다.

```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

`SpringApplication.run()`이 호출되면 Spring은 **ApplicationContext**를 생성합니다.

```text
SpringApplication.run()

↓

ApplicationContext 생성

↓

Bean 등록

↓

애플리케이션 실행
```

`ApplicationContext`가 바로 Spring Container입니다.

---

# Bean이란?

Spring Container가 생성하고 관리하는 객체를 **Bean**이라고 합니다.

예를 들어 다음 클래스는 Bean으로 등록됩니다.

```java
@Service
public class UserService {

}
```

실행 과정은 다음과 같습니다.

```text
UserService

↓

@Service

↓

Spring Container

↓

Bean 등록
```

Bean으로 등록되면 개발자가 직접 객체를 생성할 필요가 없습니다.

---

# Bean은 어떻게 등록될까?

Spring은 특정 어노테이션이 붙은 클래스를 자동으로 Bean으로 등록합니다.

대표적인 어노테이션은 다음과 같습니다.

| 어노테이션             | 역할                  |
| ----------------- | ------------------- |
| `@Component`      | 일반 컴포넌트             |
| `@Service`        | 비즈니스 로직             |
| `@Repository`     | 데이터 접근 계층           |
| `@Controller`     | MVC Controller      |
| `@RestController` | REST API Controller |
| `@Configuration`  | 설정 클래스              |

이들 대부분은 내부적으로 `@Component`를 기반으로 동작합니다.

---

# Component Scan

Spring Boot는 실행 시 자동으로 Bean을 찾아 등록합니다.

이를 **Component Scan**이라고 합니다.

```java
@SpringBootApplication
public class DemoApplication {

}
```

`@SpringBootApplication` 내부에는 다음과 같은 어노테이션이 포함되어 있습니다.

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

그중 `@ComponentScan`이 지정된 패키지를 탐색하여 Bean을 등록합니다.

```text
com.example

├── controller
├── service
├── repository
└── config
```

위 패키지 아래의 Bean들은 모두 자동으로 등록됩니다.

---

# DI(Dependency Injection)란?

Bean이 등록되었다고 해서 자동으로 사용할 수 있는 것은 아닙니다.

필요한 객체를 다른 객체에 연결하는 과정이 필요합니다.

이를 **DI(Dependency Injection)** 라고 합니다.

예를 들어 Controller는 Service가 필요합니다.

```java
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

}
```

여기서 `UserService`를 생성한 코드는 없습니다.

Spring이 이미 생성해 둔 Bean을 찾아 자동으로 주입합니다.

흐름은 다음과 같습니다.

```text
Spring Container

↓

UserService Bean 생성

↓

UserController 생성

↓

UserService 주입
```

---

# 생성자 주입을 사용하는 이유

Spring에서는 생성자 주입(Constructor Injection)을 가장 권장합니다.

```java
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

}
```

생성자 주입의 장점은 다음과 같습니다.

* 의존성이 반드시 주입된다.
* `final` 키워드를 사용할 수 있다.
* 테스트 코드 작성이 쉽다.
* 순환 참조를 조기에 발견할 수 있다.

현재는 `@Autowired`를 필드에 직접 사용하는 방식보다 생성자 주입을 사용하는 것이 일반적인 개발 방식입니다.

---

# 객체 생성부터 주입까지의 전체 흐름

Spring Boot가 실행되면 객체는 다음 순서로 관리됩니다.

```text
SpringApplication.run()

↓

ApplicationContext 생성

↓

Component Scan

↓

Bean 생성

↓

Bean 저장

↓

의존성 주입(DI)

↓

애플리케이션 실행
```

---

# Bean의 생명주기

Bean은 생성부터 소멸까지 Spring Container가 관리합니다.

```text
Bean 생성

↓

의존성 주입

↓

초기화

↓

사용

↓

소멸
```

필요하다면 초기화와 종료 시점에 원하는 작업을 수행할 수도 있습니다.

```java
@PostConstruct
public void init() {
    System.out.println("초기화");
}

@PreDestroy
public void destroy() {
    System.out.println("종료");
}
```

---

# IoC, DI, Bean의 관계

세 가지 개념은 각각 독립적인 것이 아니라 하나의 흐름으로 연결됩니다.

```text
IoC
│
├── 객체 생성 권한을 Spring에게 위임한다.
│
▼
Bean
│
├── Spring이 생성한 객체
│
▼
DI
│
└── 생성된 Bean을 필요한 곳에 주입한다.
```

즉,

* **IoC**는 객체 생성과 관리의 책임을 Spring으로 넘기는 개념입니다.
* **Bean**은 Spring이 생성하여 관리하는 객체입니다.
* **DI**는 Bean을 필요한 곳에 자동으로 연결하는 방식입니다.

---

# 마무리

Spring Boot에서 개발자는 더 이상 객체를 직접 생성하고 연결하는 데 많은 시간을 쓰지 않습니다.

대신 Spring Container가 객체를 생성하고, Bean으로 관리하며, 필요한 곳에 자동으로 주입합니다. 이러한 구조 덕분에 애플리케이션은 결합도가 낮아지고, 테스트가 쉬워지며, 유지보수성과 확장성이 크게 향상됩니다.

Spring을 처음 학습할 때는 **IoC**, **Bean**, **DI**를 각각 별개의 개념으로 외우기보다 **"Spring이 객체를 생성하고(Bean), 관리하며(IoC), 필요한 곳에 연결해준다(DI)"**라는 하나의 흐름으로 이해하면 훨씬 쉽게 전체 구조를 파악할 수 있습니다.

