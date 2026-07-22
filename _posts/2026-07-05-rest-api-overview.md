---
title: "REST API 개요 - URL 네이밍이 전부가 아니다"
date: 2026-07-05 10:00:00 +0900
categories: [Spring, Spring Boot]
tags: [rest, restful, http, api-design]
level: beginner
math: false
mermaid: false
---

> 📚 **이 글을 읽기 전에**
> - [Spring Boot 요청 처리 과정 완벽 이해 - DispatcherServlet을 중심으로]({% post_url 2026-06-30-dispatcher-servlet %}) — `@RestController`, `@GetMapping` 같은 어노테이션으로 요청이 처리되는 기본 흐름을 다룹니다.
> - [Spring Security와 JWT - Stateless 인증 방식 완벽 이해]({% post_url 2026-06-30-jwt %}) — "Stateless"라는 단어가 이 글에서도 다시 등장합니다. 사실 REST의 핵심 제약 조건 중 하나입니다.
{: .prompt-info }

## "REST = URL 네이밍 규칙"이라는 오해

`@GetMapping("/users")`, `@PostMapping("/users")` 같은 코드를 작성하다 보면, REST API를 "적절히 URL 이름을 짓는 방법" 정도로 이해하기 쉽습니다. `/getUsers` 대신 `/users`를 쓰고, `/users/{id}`처럼 경로에 ID를 넣는 규칙 정도로요.

하지만 REST(Representational State Transfer)는 원래 Roy Fielding이 2000년 박사 학위 논문에서 제시한 **아키텍처 스타일**입니다. 즉 몇 가지 **제약 조건(Constraint)의 집합**이고, URL을 어떻게 짓느냐는 그 제약 조건 중 하나(Uniform Interface)의 아주 일부일 뿐입니다.

이번 글에서는 URL 네이밍보다 한 단계 위에서, REST가 실제로 요구하는 것이 무엇인지 살펴보겠습니다.

---

## REST의 제약 조건들

REST 아키텍처 스타일은 다음과 같은 제약 조건으로 구성됩니다.

| 제약 조건 | 의미 |
| --- | --- |
| Client-Server | 클라이언트와 서버의 책임을 분리 (UI ↔ 데이터/로직) |
| Stateless | 서버는 클라이언트의 상태를 저장하지 않음 |
| Cacheable | 응답이 캐시 가능한지 명시 |
| Uniform Interface | 자원을 일관된 방식으로 다룸 (URL 네이밍이 여기 속함) |
| Layered System | 클라이언트는 서버가 단일 서버인지, 여러 계층(프록시, 게이트웨이 등)을 거치는지 몰라도 됨 |
| Code-On-Demand (선택) | 서버가 실행 가능한 코드를 클라이언트에 전달 가능 (거의 안 씀) |

이 중 핵심은 **Stateless**와 **Uniform Interface**입니다.

---

## Stateless 

**Stateless(무상태)**는 서버가 클라이언트의 이전 요청 상태를 기억하지 않는다는 뜻입니다. 매 요청은 그 자체로 완결되어야 하고, 요청을 처리하는 데 필요한 모든 정보는 그 요청 안에 포함되어야 합니다.

```text
Stateful (세션 기반)
Request 1 → 서버가 로그인 상태를 세션에 저장
Request 2 → 서버가 세션을 조회해 로그인 여부 확인

Stateless (REST)
Request 1 → 요청 안에 인증 정보(JWT 등) 포함
Request 2 → 이 요청도 독립적으로 인증 정보 포함, 서버는 아무것도 기억 안 함
```

이전 JWT 글에서 "서버가 로그인 상태를 저장하지 않고, 클라이언트가 보낸 토큰만으로 매번 인증한다"고 설명했던 것이 바로 이 제약 조건입니다. REST API가 JWT 같은 토큰 기반 인증과 잘 맞는 이유도, JWT 자체가 Stateless 제약을 만족시키기 위해 설계된 방식이기 때문입니다.

---

## Uniform Interface — 자원을 URL로, 행위를 HTTP Method로

여기서부터가 흔히 "REST API 설계"라고 부르는 부분입니다. 핵심 원칙은 **URL은 자원(명사)만 나타내고, 행위(동사)는 HTTP Method가 표현한다**는 것입니다.

```text
잘못된 예              올바른 예
GET  /getUsers         GET    /users
POST /createUser       POST   /users
POST /deleteUser/1     DELETE /users/1
POST /updateUser/1     PUT    /users/1
```

`/getUsers`, `/createUser`처럼 URL에 동사를 넣으면 문제가 생깁니다. HTTP Method가 이미 "무엇을 할지"를 나타내는데, URL에도 동사가 들어가면 같은 정보가 두 곳에 중복되고, 둘이 어긋나는 경우(`GET /deleteUser`처럼)도 방지할 방법이 없습니다. 그래서 URL은 "무엇에 대한 요청인가"(자원)만 답하고, "무엇을 할 것인가"(행위)는 HTTP Method가 전담합니다.

| Method | 의미 | 예시 |
| --- | --- | --- |
| GET | 조회 | `GET /users/1` |
| POST | 생성 | `POST /users` |
| PUT | 전체 수정(덮어쓰기) | `PUT /users/1` |
| PATCH | 부분 수정 | `PATCH /users/1` |
| DELETE | 삭제 | `DELETE /users/1` |

> ❓ **생각해보기**
> `@GetMapping`, `@PostMapping`처럼 Spring이 HTTP Method별로 다른 어노테이션을 제공하는 이유를 이 관점에서 다시 생각해보세요. 만약 모든 요청을 `@RequestMapping("/users")`로만 받고 메서드 안에서 분기 처리한다면 무엇을 잃게 될까요?
{: .prompt-warning }

---

## Safe와 Idempotent — Method 선택이 곧 계약

HTTP Method는 단순히 "역할 분담" 이상의 의미를 가집니다. 각 Method는 **안전성(Safe)**과 **멱등성(Idempotent)**이라는 약속을 내포합니다.

- **Safe(안전)**: 서버의 상태를 변경하지 않는 Method. `GET`, `HEAD`, `OPTIONS`가 해당합니다. 그래서 브라우저나 중간 프록시가 `GET` 요청을 캐싱하거나 미리 요청(prefetch)해도 안전하다고 가정합니다.
- **Idempotent(멱등)**: 같은 요청을 여러 번 보내도 결과가 달라지지 않는 Method. `GET`, `PUT`, `DELETE`가 해당합니다. `PUT /users/1`로 같은 데이터를 열 번 보내도 최종 상태는 동일합니다. 반면 `POST /users`는 멱등하지 않습니다 — 같은 요청을 열 번 보내면 사용자가 열 명 생성됩니다.

```text
POST /users  (멱등하지 않음) → 재시도하면 중복 생성 위험
PUT /users/1 (멱등함)        → 재시도해도 안전
```

이 약속 때문에 네트워크 오류로 요청이 실패했을 때 "재시도해도 되는 Method인지"를 Method 자체로 판단할 수 있습니다. `PUT` 요청은 타임아웃이 나도 안심하고 재전송할 수 있지만, `POST` 요청은 재전송 전에 실제로 처리됐는지 먼저 확인해야 합니다.

---

## 상태 코드도 계약의 일부

응답의 HTTP 상태 코드도 "무엇을 했는지"를 정확히 전달하는 계약입니다. 아무 요청에나 `200 OK`만 반환하면 클라이언트는 성공/실패를 응답 바디를 파싱해야만 알 수 있게 됩니다.

| 상태 코드 | 의미 |
| --- | --- |
| 200 OK | 조회/수정 성공 |
| 201 Created | 생성 성공 (Location 헤더에 새 자원 위치 포함 권장) |
| 204 No Content | 성공했지만 반환할 데이터 없음 (주로 DELETE) |
| 400 Bad Request | 요청 형식/값이 잘못됨 |
| 401 Unauthorized | 인증 정보 없음/유효하지 않음 |
| 403 Forbidden | 인증은 됐지만 권한 없음 |
| 404 Not Found | 자원이 존재하지 않음 |
| 409 Conflict | 현재 상태와 충돌 (예: 중복 생성) |
| 500 Internal Server Error | 서버 내부 오류 |

```java
@PostMapping("/users")
public ResponseEntity<UserDto> create(@RequestBody UserCreateRequest request) {
    UserDto created = userService.create(request);
    return ResponseEntity
            .created(URI.create("/users/" + created.getId()))
            .body(created);
}
```

`ResponseEntity`를 사용하면 상태 코드와 헤더까지 명시적으로 제어할 수 있습니다. 생성에 성공했다면 `200`이 아니라 `201 Created`를 반환하는 것이 REST의 Uniform Interface를 지키는 방법입니다.

> 🛠 **직접 해보기**
> 기존에 만들어본 API(또는 새로 간단히 만든 API)의 엔드포인트들을 점검해보세요. URL에 동사가 들어간 곳은 없는지, 생성 API가 `200`이 아니라 `201`을 반환하는지, 삭제 API가 `204`를 반환하는지 확인하고 고쳐보세요.
{: .prompt-tip }

---

## 사실 대부분의 API는 "완전한 REST"가 아니다

여기까지 왔다면 한 가지는 짚고 넘어가야 합니다. 진짜 순수한 REST는 자원, 표현, HTTP Method, 상태 코드에 더해 **HATEOAS**(Hypermedia As The Engine Of Application State)까지 요구합니다. 응답 안에 "다음에 할 수 있는 행동"에 대한 링크를 함께 담아, 클라이언트가 URL 구조를 미리 알지 않아도 응답을 따라가며 API를 사용할 수 있어야 한다는 원칙입니다.

```json
{
  "id": 1,
  "name": "Kim",
  "_links": {
    "self": { "href": "/users/1" },
    "orders": { "href": "/users/1/orders" }
  }
}
```

하지만 실제 이 단계까지 구현하는 API는 많지 않습니다. Leonard Richardson은 REST 성숙도를 아래처럼 단계로 나눴습니다(Richardson Maturity Model).

```text
Level 0: 단일 엔드포인트에 모든 요청을 POST로 (RPC 스타일)
Level 1: 자원별로 URL 분리
Level 2: HTTP Method와 상태 코드까지 의미대로 사용  ← 대부분의 실무 API가 여기
Level 3: HATEOAS까지 적용 (진짜 REST)
```

우리가 지금까지 다룬 내용(자원 중심 URL + Method + 상태 코드)은 **Level 2**에 해당합니다. 이 정도만 지켜도 흔히 "RESTful API"라고 부르고, 실무에서도 이 수준이 사실상 표준입니다.

---

## 마무리

REST API는 URL 이름을 예쁘게 짓는 규칙이 아니라, Client-Server 분리·Stateless·Uniform Interface 같은 제약 조건을 지키는 **아키텍처 스타일**입니다. 그중 URL을 자원 중심으로 설계하고 HTTP Method로 행위를 표현하는 것(Uniform Interface)이 실무에서 가장 눈에 띄는 부분이라 "REST = URL 네이밍"으로 오해하기 쉽지만, Method가 내포한 안전성·멱등성 계약과 상태 코드의 의미까지 지켜야 진짜 의미의 RESTful API에 가까워집니다.

다만 실무 대부분의 API는 HATEOAS까지는 구현하지 않는 Level 2 수준에 머물러 있고, 이것만으로도 충분히 "RESTful"하다고 인정받습니다. 완벽한 REST를 추구하기보다, 지금 만드는 API가 이 제약 조건들과 얼마나 일관되게 맞아떨어지는지를 점검하는 것이 실무적으로 더 유용합니다.
