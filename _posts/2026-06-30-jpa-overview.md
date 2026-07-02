---
title: "JPA 개요 - 영속성 컨텍스트부터 Spring Data JPA까지"
date: 2026-06-30 14:00:00 +0900
categories: [Spring, Spring Boot]
tags: [spring, springboot, jpa, spring-data-jpa, hibernate, repository, persistence-context]
---

# Spring Data JPA 완벽 이해 - Repository부터 CRUD까지

Spring Boot로 백엔드 애플리케이션을 개발하다 보면 `JpaRepository`를 상속받는 Repository를 자주 작성하게 됩니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

단 두 줄의 코드만으로도 다음과 같은 기능을 사용할 수 있습니다.

* 데이터 조회
* 데이터 저장
* 수정(Update)
* 삭제(Delete)
* 페이징(Pageing)
* 정렬(Sorting)

그렇다면 **메서드를 직접 구현하지 않았는데 어떻게 동작하는 걸까요?**

이번 글에서는 **Spring Data JPA가 무엇인지**, 그리고 내부적으로 어떤 방식으로 Repository를 구현하는지 알아보겠습니다.

---

# Spring Data JPA란?

먼저 JPA와 Spring Data JPA는 서로 다른 개념입니다.

* **JPA(Java Persistence API)** 는 Java 객체와 관계형 데이터베이스를 매핑하기 위한 **표준 명세(Specification)** 입니다.
* **Spring Data JPA**는 JPA를 더 쉽고 생산적으로 사용할 수 있도록 Spring에서 제공하는 프로젝트입니다.

관계를 그림으로 표현하면 다음과 같습니다.

```text
Spring Data JPA
        │
        ▼
JPA (Specification)
        │
        ▼
Hibernate (Implementation)
        │
        ▼
Database
```

즉, Spring Data JPA는 JPA를 직접 구현한 것이 아니라, JPA 구현체(대표적으로 Hibernate)를 더 편리하게 사용할 수 있도록 감싸는 추상화 계층입니다.

---

# JPA만 사용할 경우

Spring Data JPA를 사용하지 않는다면 `EntityManager`를 이용해 직접 데이터를 처리해야 합니다.

```java
@Repository
public class UserRepository {

    @PersistenceContext
    private EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }

    public void save(User user) {
        em.persist(user);
    }

}
```

데이터를 조회하거나 저장하는 메서드를 모두 직접 작성해야 합니다.

프로젝트가 커질수록 Repository 코드도 함께 증가합니다.

여기서 등장하는 `EntityManager`가 바로 JPA의 핵심 동작 원리를 이해하는 열쇠입니다.

---

# 영속성 컨텍스트(Persistence Context)란?

`EntityManager`는 Entity를 직접 DB에 반영하지 않습니다.

대신 **영속성 컨텍스트**라는 내부 저장 공간에 Entity를 담아두고 관리합니다.

```text
EntityManager
      │
      ▼
영속성 컨텍스트
      │
      ▼
Entity 관리
```

영속성 컨텍스트는 "Entity를 영구 저장하는 환경"이라는 뜻으로, 애플리케이션과 데이터베이스 사이에서 **Entity의 상태를 추적하고 캐싱하는 1차 저장소** 역할을 합니다.

Entity는 영속성 컨텍스트 안에서 다음과 같은 생명주기를 가집니다.

```text
비영속(new/transient)
      │
   em.persist()
      ▼
영속(managed)
      │
   em.detach() / clear() / close()
      ▼
준영속(detached)


영속(managed)
      │
   em.remove()
      ▼
삭제(removed)
```

| 상태 | 설명 |
| --- | --- |
| 비영속 | 객체만 생성되어 있고 영속성 컨텍스트와 무관한 상태 |
| 영속 | 영속성 컨텍스트가 관리하는 상태 (`persist`, `find`로 조회된 상태) |
| 준영속 | 영속성 컨텍스트에서 분리된 상태 |
| 삭제 | 삭제가 예약된 상태 |

---

# 1차 캐시

영속성 컨텍스트는 내부에 **1차 캐시**를 가지고 있습니다.

`em.find()`로 조회한 Entity는 1차 캐시에 저장되며, 같은 트랜잭션 안에서 동일한 Entity를 다시 조회하면 DB에 쿼리를 보내지 않고 캐시에서 바로 반환합니다.

```java
User user1 = em.find(User.class, 1L); // DB 조회, 1차 캐시에 저장
User user2 = em.find(User.class, 1L); // 1차 캐시에서 반환, 쿼리 없음
```

```text
em.find(User.class, 1L)
      │
      ▼
1차 캐시에 있는가?
      │
   있음 ──────────► 캐시에서 반환
      │
   없음
      ▼
   DB 조회 → 1차 캐시에 저장 → 반환
```

같은 트랜잭션 내에서 조회한 동일 Entity는 항상 같은 인스턴스이므로(**동일성 보장**), `user1 == user2`는 `true`입니다.

---

# 더티 체킹(Dirty Checking)

영속 상태의 Entity 값을 변경하면, 별도로 `update()`나 `save()`를 호출하지 않아도 트랜잭션이 끝날 때 변경 사항이 자동으로 DB에 반영됩니다.

```java
@Transactional
public void updateName(Long id, String newName) {
    User user = em.find(User.class, id); // 영속 상태

    user.setName(newName); // 값만 변경, DB 호출 없음

} // 트랜잭션 커밋 시점에 UPDATE 쿼리 자동 실행
```

이것이 가능한 이유는 영속성 컨텍스트가 Entity를 최초 조회한 시점의 상태를 **스냅샷**으로 저장해 두기 때문입니다.

```text
트랜잭션 커밋
      │
      ▼
Flush 발생
      │
      ▼
스냅샷과 현재 Entity 비교
      │
      ▼
변경된 필드만 UPDATE 쿼리 생성
```

이 과정을 **더티 체킹**이라고 하며, Spring Data JPA의 `save()`가 영속 상태의 Entity에 대해서는 사실상 아무 일도 하지 않아도 되는 이유이기도 합니다.

---

# 쓰기 지연(Write-Behind)

`em.persist()`를 호출한다고 해서 즉시 INSERT 쿼리가 나가는 것은 아닙니다.

영속성 컨텍스트는 SQL을 내부 쓰기 지연 저장소에 쌓아두었다가, **Flush** 시점에 한 번에 DB로 전송합니다.

```text
em.persist(user)
      │
      ▼
쓰기 지연 SQL 저장소에 INSERT 쿼리 저장
      │
      ▼
Flush (커밋 직전, 또는 JPQL 실행 직전 등)
      │
      ▼
실제 DB에 SQL 전송
```

Flush는 영속성 컨텍스트의 변경 내용을 DB에 동기화하는 과정일 뿐, 영속성 컨텍스트 자체를 비우거나 트랜잭션을 커밋하는 것은 아닙니다.

---

# 지연 로딩과 즉시 로딩, 그리고 N+1 문제

연관관계가 있는 Entity를 조회할 때 연관된 Entity를 언제 가져올지는 로딩 전략으로 결정합니다.

```java
@Entity
public class Order {

    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private User user;

}
```

| 전략 | 설명 |
| --- | --- |
| `FetchType.EAGER` | 연관 Entity를 즉시 함께 조회 |
| `FetchType.LAZY` | 연관 Entity를 실제 사용하는 시점에 조회(프록시 객체로 지연) |

연관관계의 기본 전략(`@ManyToOne`은 EAGER, `@OneToMany`는 LAZY)을 그대로 두면 예상치 못한 추가 쿼리가 발생할 수 있는데, 대표적인 문제가 **N+1 문제**입니다.

```java
List<Order> orders = orderRepository.findAll(); // 쿼리 1번

for (Order order : orders) {
    order.getUser().getName(); // 지연 로딩된 User를 조회할 때마다 쿼리 N번 추가 발생
}
```

```text
Order 목록 조회 (1번)
      │
      ▼
각 Order마다 연관된 User 조회 (N번)
      │
      ▼
총 1 + N번의 쿼리 실행
```

N+1 문제는 실무에서 성능 저하의 주된 원인 중 하나이며, `fetch join`, `@EntityGraph`, 또는 batch size 설정(`hibernate.default_batch_fetch_size`) 등으로 해결합니다.

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();
```

---

# 트랜잭션과 영속성 컨텍스트

Spring에서는 기본적으로 트랜잭션 하나당 영속성 컨텍스트 하나가 생성되고 공유됩니다(`@Transactional` 범위 = 영속성 컨텍스트의 생존 범위).

```text
@Transactional 메서드 시작
      │
      ▼
영속성 컨텍스트 생성
      │
      ▼
Entity 조회/변경 (1차 캐시, 더티 체킹 동작)
      │
      ▼
메서드 종료 → Flush → 커밋
      │
      ▼
영속성 컨텍스트 종료
```

트랜잭션 범위를 벗어난 뒤(예: Controller 계층)에는 Entity가 준영속 상태가 되어 지연 로딩이 동작하지 않고 `LazyInitializationException`이 발생할 수 있습니다. 이 문제는 필요한 데이터를 트랜잭션 범위 안에서 미리 조회하거나 DTO로 변환해 반환하는 방식으로 해결하는 것이 일반적입니다.

---

# Spring Data JPA를 사용하면?

같은 기능을 Spring Data JPA로 작성하면 훨씬 간단합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

구현 클래스가 없는데도 CRUD 기능을 바로 사용할 수 있습니다.

```java
userRepository.save(user);

userRepository.findById(1L);

userRepository.findAll();

userRepository.delete(user);
```

Spring Data JPA가 실행 시점에 Repository의 구현 객체를 자동으로 생성해 주기 때문입니다.

---

# JpaRepository 구조

`JpaRepository`는 여러 인터페이스를 상속하여 다양한 기능을 제공합니다.

```text
Repository
      │
      ▼
CrudRepository
      │
      ▼
PagingAndSortingRepository
      │
      ▼
JpaRepository
```

각 인터페이스의 역할은 다음과 같습니다.

| 인터페이스                      | 주요 기능        |
| -------------------------- | ------------ |
| Repository                 | 마커 인터페이스     |
| CrudRepository             | CRUD 기능      |
| PagingAndSortingRepository | 페이징 및 정렬     |
| JpaRepository              | JPA 관련 추가 기능 |

일반적으로는 `JpaRepository` 하나만 상속하면 대부분의 기능을 사용할 수 있습니다.

---

# Entity와 Repository

예를 들어 회원 정보를 관리하는 애플리케이션을 생각해 보겠습니다.

Entity는 다음과 같이 정의합니다.

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String email;

}
```

Repository는 다음과 같이 작성합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

여기서

* `User`는 관리할 Entity
* `Long`은 기본 키(ID)의 타입

을 의미합니다.

---

# 기본 CRUD 기능

Spring Data JPA는 자주 사용하는 CRUD 메서드를 기본으로 제공합니다.

## 저장

```java
userRepository.save(user);
```

새로운 Entity라면 INSERT가 실행되고, 이미 존재하는 Entity라면 UPDATE가 수행됩니다.

---

## 단건 조회

```java
Optional<User> user = userRepository.findById(1L);
```

`Optional`을 반환하기 때문에 조회 결과가 없는 경우도 안전하게 처리할 수 있습니다.

---

## 전체 조회

```java
List<User> users = userRepository.findAll();
```

모든 데이터를 조회합니다.

---

## 삭제

```java
userRepository.delete(user);
```

또는 ID를 이용해 삭제할 수도 있습니다.

```java
userRepository.deleteById(1L);
```

---

# 메서드 이름만으로 조회하기

Spring Data JPA의 가장 큰 장점 중 하나는 **Query Method**입니다.

메서드 이름만으로 SQL을 생성합니다.

예를 들어

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

}
```

Spring Data JPA는 내부적으로 다음과 같은 JPQL을 생성합니다.

```sql
SELECT u
FROM User u
WHERE u.email = :email
```

개발자가 SQL을 직접 작성하지 않아도 됩니다.

---

# 다양한 Query Method

메서드 이름 규칙만 따르면 다양한 조건 검색도 가능합니다.

```java
findByName(String name)

findByAgeGreaterThan(int age)

findByNameContaining(String keyword)

findByEmailAndName(String email, String name)

findByCreatedAtBetween(LocalDate start, LocalDate end)
```

Spring Data JPA는 메서드 이름을 분석하여 적절한 JPQL을 생성합니다.

---

# @Query 사용하기

복잡한 조회는 `@Query`를 사용할 수 있습니다.

```java
@Query("""
       SELECT u
       FROM User u
       WHERE u.email LIKE %:keyword%
       """)
List<User> search(String keyword);
```

JPQL뿐 아니라 Native SQL도 사용할 수 있습니다.

```java
@Query(value = """
SELECT *
FROM users
WHERE email LIKE %:keyword%
""", nativeQuery = true)
List<User> searchNative(String keyword);
```

---

# 페이징과 정렬

대량의 데이터를 조회할 때는 페이징이 필수입니다.

```java
Page<User> page =
    userRepository.findAll(PageRequest.of(0, 10));
```

첫 번째 페이지에서 10개의 데이터를 조회합니다.

정렬도 쉽게 적용할 수 있습니다.

```java
Sort sort = Sort.by("name").ascending();

userRepository.findAll(sort);
```

페이징과 정렬을 함께 사용하는 것도 가능합니다.

```java
PageRequest.of(
    0,
    20,
    Sort.by("createdAt").descending()
);
```

---

# Spring Data JPA의 내부 동작

Repository를 선언하면 Spring Boot 실행 시 다음과 같은 과정이 수행됩니다.

```text
Application 시작
        │
        ▼
Repository 인터페이스 탐색
        │
        ▼
Proxy 객체 생성
        │
        ▼
Bean 등록
        │
        ▼
의존성 주입(DI)
```

실제로는 우리가 작성하지 않은 구현 클래스가 동적으로 생성됩니다.

```text
UserRepository
        │
        ▼
Spring Proxy
        │
        ▼
SimpleJpaRepository
```

대부분의 CRUD 기능은 내부적으로 `SimpleJpaRepository`가 담당합니다.

---

# Spring Data JPA의 장점

Spring Data JPA를 사용하면 다음과 같은 이점을 얻을 수 있습니다.

* 반복적인 CRUD 코드 작성이 줄어든다.
* 메서드 이름만으로 조회 쿼리를 생성할 수 있다.
* 페이징과 정렬 기능을 쉽게 사용할 수 있다.
* JPA와 자연스럽게 통합된다.
* Repository 구현 코드를 거의 작성하지 않아도 된다.
* 생산성과 유지보수성이 향상된다.

하지만 모든 조회를 Query Method로 해결할 수 있는 것은 아닙니다.

복잡한 통계성 조회나 여러 테이블을 조인하는 쿼리는 `@Query`, QueryDSL, 또는 Native SQL을 함께 사용하는 것이 일반적입니다.

---

# 마무리

Spring Data JPA는 JPA를 대체하는 기술이 아니라 **JPA를 더 쉽고 편리하게 사용할 수 있도록 도와주는 Spring의 추상화 계층**입니다.

간단한 CRUD부터 페이징, 정렬, 메서드 이름 기반 쿼리 생성까지 다양한 기능을 제공하여 개발자가 반복적인 데이터 접근 코드를 작성하는 부담을 크게 줄여줍니다.

다만 Spring Data JPA를 효과적으로 활용하려면 앞서 살펴본 **영속성 컨텍스트, 1차 캐시, 더티 체킹, 지연 로딩과 N+1 문제**와 같은 JPA의 기본 개념을 함께 이해하는 것이 중요합니다. Spring Data JPA의 `save()`, `findById()` 같은 편리한 메서드들도 결국 이 영속성 컨텍스트 위에서 동작하기 때문에, 내부 동작 원리를 알아야 예상치 못한 쿼리나 `LazyInitializationException` 같은 문제를 마주쳤을 때 원인을 정확히 진단할 수 있습니다.

