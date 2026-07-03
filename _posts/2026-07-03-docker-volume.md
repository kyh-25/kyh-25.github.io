---
title: "Docker Volume 개요 - 컨테이너 데이터를 안전하게 보존하기"
date: 2026-07-03 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, volume, bind-mount, data-persistence]
level: beginner
series:
  name: Docker 입문
  order: 3
math: false
mermaid: false
---

> 📚 **이 글을 읽기 전에**
> - [Docker 개요 - 컨테이너는 가상화가 아니다]({% post_url 2026-07-01-docker-overview %}) — "컨테이너는 기본적으로 휘발성이다" 절에서 볼륨의 필요성을 짧게 언급했습니다. 이 글은 그 개념을 제대로 다룹니다.
> - [Docker Compose 개요]({% post_url 2026-07-02-docker-compose %}) — 여러 컨테이너를 함께 다루는 방법을 다룹니다.
{: .prompt-info }

## 다시 확인하는 문제 상황

데이터베이스 컨테이너를 하나 띄우고, 데이터를 몇 건 넣고, 컨테이너를 지웠다가 같은 이미지로 다시 띄운다고 해보겠습니다.

```bash
docker run -d --name my-db -e POSTGRES_PASSWORD=1234 postgres:16
# ... 데이터 입력 ...
docker rm -f my-db
docker run -d --name my-db -e POSTGRES_PASSWORD=1234 postgres:16
# 방금 넣었던 데이터가 전부 사라져 있습니다.
```

컨테이너 이름도 같고 이미지도 같은데, 데이터는 없습니다. 왜 이런 일이 생기는지부터 짚고 가겠습니다.

---

## 컨테이너는 왜 휘발성인가 — 쓰기 가능 레이어

컨테이너를 실행하면, 이미지의 읽기 전용 레이어들 위에 **그 컨테이너만을 위한 쓰기 가능한 레이어**가 하나 추가됩니다.

```text
쓰기 가능 레이어 (컨테이너 전용, 여기에만 기록)
──────────────────────────────────────────
Layer 3 (이미지, 읽기 전용)
Layer 2 (이미지, 읽기 전용)
Layer 1 (이미지, 읽기 전용)
```

컨테이너 안에서 파일을 새로 만들거나 수정하면 전부 이 쓰기 가능 레이어에 기록됩니다. 아래에 있는 이미지 레이어들은 절대 바뀌지 않습니다(같은 이미지로 컨테이너를 여러 개 띄워도 서로 영향이 없는 이유이기도 합니다).

문제는 `docker rm`으로 컨테이너를 지우면 이 쓰기 가능 레이어까지 통째로 함께 사라진다는 것입니다. 데이터베이스가 저장한 데이터도 결국 이 레이어 안의 파일이므로, 컨테이너와 운명을 같이합니다.

```text
docker rm my-db
      │
      ▼
쓰기 가능 레이어 삭제
      │
      ▼
그 안에 있던 데이터도 함께 삭제
```

---

## 볼륨(Volume)이 하는 일

**볼륨**은 컨테이너의 쓰기 가능 레이어 바깥에, **컨테이너의 생명주기와 무관하게 독립적으로 존재하는 저장 공간**입니다. 컨테이너 안의 특정 디렉터리를 이 볼륨에 연결(마운트)해두면, 그 디렉터리에 쓰는 파일은 컨테이너의 쓰기 가능 레이어가 아니라 볼륨에 저장됩니다.

```text
Container
      │
   /var/lib/postgresql/data  ← 이 경로만 볼륨에 연결
      │
      ▼
Volume (컨테이너 바깥, 독립적으로 존재)
```

컨테이너를 지워도 볼륨은 그대로 남아 있고, 새 컨테이너를 만들면서 같은 볼륨을 다시 연결하면 이전 데이터를 그대로 이어받습니다.

> ❓ **생각해보기**
> 볼륨은 "컨테이너 바깥에 독립적으로 존재"한다고 했습니다. 그렇다면 컨테이너 A와 컨테이너 B가 같은 볼륨을 동시에 마운트하는 것도 가능할까요? 가능하다면 어떤 상황에서 유용할지 먼저 생각해보세요.
{: .prompt-warning }

---

## 여러 컨테이너가 같은 볼륨을 써도 되는가

기술적으로는 가능합니다. `docker run -v my-data:/path`를 컨테이너 A, B에 똑같이 지정하면 둘 다 같은 볼륨에 연결됩니다.

다만 **독립된 DB 서버 컨테이너 여러 개**가 같은 데이터 디렉터리를 공유하는 것은 오히려 위험합니다. PostgreSQL 같은 DB 엔진은 자신이 데이터 디렉터리의 유일한 소유자라고 가정하고 동작하며(내부적으로 잠금 파일로 이를 강제합니다), 독립된 두 프로세스가 같은 파일을 동시에 쓰면 데이터가 깨지거나 두 번째 인스턴스가 시작에 실패합니다.

같은 볼륨을 여러 컨테이너가 공유하는 것이 실제로 유용한 경우는 따로 있습니다.

- 여러 앱 컨테이너가 **같은 정적 파일이나 업로드 파일**을 함께 읽고 쓸 때
- 한 컨테이너가 로그를 쓰고, 다른 컨테이너(로그 수집기)가 그 볼륨을 읽기 전용으로 붙여서 가져가는 **사이드카 패턴**

핵심 기준은 "그 데이터를 관리하는 프로세스가 혼자 소유해야 하는 엔진(DB 서버)인가, 아니면 여러 프로세스가 나눠서 읽고 써도 되는 공유 데이터인가"입니다. DB를 여러 개 두고 싶다면 볼륨을 공유하는 것이 아니라, 각자 자기 볼륨을 가진 상태에서 DB 자체의 복제(Replication) 기능으로 데이터를 동기화하는 것이 올바른 방법입니다.

---

## 볼륨의 두 가지 방식 — Named Volume과 Bind Mount

Docker에서 컨테이너 바깥에 데이터를 저장하는 방법은 크게 **Named Volume**과 **Bind Mount** 두 가지입니다. `-v` 옵션에 무엇을 어떻게 쓰느냐에 따라 이 둘 중 하나, 혹은 이름을 지정하지 않은 변형(익명 볼륨)이 결정됩니다. 콜론(`:`)으로 구분된 조각이 몇 개인지, 그리고 소스가 경로 형태인지 이름 형태인지가 기준입니다.

| 표현 | 소스 지정 | 의미 |
| --- | --- | --- |
| `-v /container/path` | 없음 (경로 1개) | **익명 볼륨(Anonymous Volume)** — Docker가 임의 이름의 볼륨을 만들어 그 경로에 연결 |
| `-v /host/path:/container/path` | `/`로 시작하는 경로 | **Bind Mount** — 호스트의 그 경로를 그대로 연결 |
| `-v my-data:/container/path` | 슬래시 없는 이름 | **Named Volume** — 그 이름의 볼륨을 연결 (없으면 자동 생성) |

### Named Volume

Docker가 직접 만들고 관리하는 저장 공간입니다. 실제 파일이 호스트의 어느 경로에 저장되는지는 Docker가 알아서 관리하고, 사용자는 이름으로만 다룹니다.

```bash
docker volume create my-data

docker run -d --name my-db \
  -v my-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=1234 \
  postgres:16
```

`-v my-data:/var/lib/postgresql/data`는 위 표의 세 번째 경우로, "이름이 `my-data`인 볼륨을 컨테이너의 `/var/lib/postgresql/data` 경로에 연결하라"는 뜻입니다. 뒤쪽 경로는 항상 **컨테이너 내부** 경로이며, 마침 PostgreSQL 이미지가 실제 데이터를 저장하는 경로이기 때문에 이 볼륨에 기록되는 내용이 곧 DB 데이터가 됩니다.

### Bind Mount

호스트 파일시스템의 **특정 경로를 직접** 컨테이너 안에 연결하는 방식입니다. Docker가 관리하는 별도의 저장 공간이 아니라, 이미 존재하는 호스트의 실제 디렉터리를 그대로 씁니다.

```bash
docker run -d --name my-app \
  -v /Users/my-app/src:/app/src \
  my-app:1.0
```

호스트의 `src` 폴더를 컨테이너의 `/app/src`에 그대로 연결했기 때문에, 호스트에서 코드를 수정하면 컨테이너 안에도 즉시 반영됩니다. 그래서 로컬 개발 중 코드 변경을 바로 컨테이너에 반영하고 싶을 때 자주 사용합니다.

| 구분 | Named Volume | Bind Mount |
| --- | --- | --- |
| 저장 위치 | Docker가 관리 (경로를 몰라도 됨) | 호스트의 지정된 경로 (경로를 직접 알아야 함) |
| 주 용도 | 데이터베이스 등 영속 데이터 보존 | 로컬 개발 중 소스 코드 실시간 반영 |
| 이식성 | 이미지/환경이 바뀌어도 이름만 알면 재사용 가능 | 호스트 경로 구조에 의존적 |

---

## 자주 쓰는 볼륨 명령어

```bash
docker volume create my-data     # 볼륨 생성
docker volume ls                  # 볼륨 목록
docker volume inspect my-data     # 볼륨 상세 정보(실제 저장 경로 등)
docker volume rm my-data          # 볼륨 삭제
```

> 🛠 **직접 해보기**
> `docker volume create my-data`로 볼륨을 만들고, `-v my-data:/var/lib/postgresql/data` 옵션으로 PostgreSQL 컨테이너를 띄워보세요. 데이터를 몇 건 넣은 뒤 `docker rm -f`로 컨테이너를 지우고, 같은 볼륨으로 다시 컨테이너를 띄워 데이터가 남아 있는지 확인해보세요.
{: .prompt-tip }

---

## Compose에서 볼륨 사용하기

지난 글의 `docker-compose.yml`에 `db` 서비스가 있었습니다. 여기에 Named Volume을 추가하면 이렇게 됩니다.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: 1234
    volumes:
      - db-data:/var/lib/postgresql/data   # 컨테이너 경로 ← 볼륨 이름

  app:
    image: my-app:1.0
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/postgres
    depends_on:
      - db

volumes:
  db-data:   # 최상위에 사용할 볼륨 이름을 선언해야 합니다
```

`services.db.volumes`에서 사용한 `db-data`는 파일 최상위의 `volumes` 항목에 미리 선언되어 있어야 합니다. 이렇게 하면 `docker compose down`으로 컨테이너와 네트워크를 정리해도 `db-data` 볼륨은 남아 있고, `docker compose up`을 다시 실행하면 이전 데이터를 그대로 이어받습니다. 볼륨까지 완전히 지우고 싶다면 `docker compose down -v`를 사용합니다.

> 🛠 **직접 해보기**
> 지난 글의 `docker-compose.yml`에 위 `volumes` 설정을 추가한 뒤 `docker compose down`으로 컨테이너를 지우고, 다시 `docker compose up -d`로 띄워보세요. 데이터베이스에 넣었던 데이터가 남아 있는지 확인해보세요.
{: .prompt-tip }

---

## 마무리

컨테이너가 휘발성인 이유는 컨테이너 전용 쓰기 가능 레이어가 컨테이너와 함께 삭제되기 때문입니다. 볼륨은 이 레이어 바깥, 컨테이너의 생명주기와 무관한 별도의 저장 공간을 만들어 이 문제를 해결합니다.

Docker가 직접 관리하는 **Named Volume**은 데이터베이스처럼 영속적으로 보존해야 하는 데이터에 적합하고, 호스트 경로를 그대로 연결하는 **Bind Mount**는 로컬 개발 중 소스 코드를 실시간으로 반영할 때 적합합니다.

지금까지 이미지·컨테이너의 기본 개념, 여러 컨테이너를 함께 관리하는 Compose, 데이터를 보존하는 볼륨까지 살펴봤습니다. 이 세 가지만 이해하고 있으면 로컬 개발 환경에서 Docker를 다루는 데 필요한 기본기는 대부분 갖춘 셈입니다.

호스트 한 대를 넘어 여러 대의 서버에 컨테이너를 자동으로 배치·복구·확장하고 싶다면, 그다음은 **[Kubernetes]({% post_url 2026-07-04-kubernetes-overview %})**의 영역입니다.
