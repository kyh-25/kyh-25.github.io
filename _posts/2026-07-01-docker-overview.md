---
title: "Docker 개요 - 컨테이너는 가상화가 아니다"
date: 2026-07-01 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, virtualization, devops]
level: beginner
series:
  name: Docker 입문
  order: 1
math: false
mermaid: false
---

## Docker는 가상화가 아니다

Docker를 처음 접하면 자연스럽게 "가상 머신(Virtual Machine, VM)의 가벼운 버전" 정도로 이해하게 됩니다. 실제로 컨테이너(Container)와 VM은 하는 일이 비슷해 보입니다. 둘 다 "격리된 실행 환경"을 만들어주고, 둘 다 "여기서는 이 앱만 실행된다"는 느낌을 줍니다.

하지만 두 기술은 **격리를 만들어내는 방식**이 근본적으로 다릅니다. 이 차이를 이해하지 못하면 "왜 컨테이너는 이렇게 빨리 뜨지?", "왜 컨테이너 안에서 커널 버전을 바꿀 수 없지?" 같은 질문에 답할 수 없습니다.

---

## 전통적인 가상화(VM) 방식

VM은 하이퍼바이저(Hypervisor)라는 소프트웨어 위에서 **완전한 운영체제(Guest OS) 전체**를 새로 띄웁니다.

```text
Host Hardware
      │
      ▼
Hypervisor
      │
   ┌──┴──┬──────┐
   ▼     ▼      ▼
Guest OS  Guest OS  Guest OS
(Linux)   (Windows) (Linux)
   │        │         │
   App      App       App
```

각 VM은 자신만의 커널(Kernel), 자신만의 부팅 과정, 자신만의 디바이스 드라이버를 전부 가지고 있습니다. 그래서 VM 하나를 켜는 데는 수십 초에서 몇 분이 걸리고, 디스크 용량도 GB 단위로 필요합니다.

VM이 이렇게 무거운 이유는 명확합니다. **하드웨어 자체를 가상화해서 그 위에 완전히 독립된 운영체제를 통째로 얹기 때문**입니다.

---

## 컨테이너 방식

반면 컨테이너는 커널을 새로 띄우지 않습니다. **호스트(Host) 운영체제의 커널을 그대로 공유**하면서, 커널이 제공하는 격리 기능만 이용해 "이 프로세스 그룹은 다른 프로세스들과 분리된 세상에 있다"고 착각하게 만듭니다.

```text
Host Hardware
      │
      ▼
Host OS Kernel (공유)
      │
      ▼
Docker Engine
      │
   ┌──┴──┬──────┐
   ▼     ▼      ▼
Container Container Container
 (App)    (App)    (App)
```

컨테이너의 실체는 사실 **격리된 리눅스 프로세스**일 뿐입니다. 새로운 커널을 부팅하는 과정이 없으니 컨테이너는 1초 안팎으로 시작되고, 이미지 용량도 VM보다 훨씬 작습니다.

---

## Windows는 격리를 못 하는가 — Windows Containers와 Docker Desktop

Docker 이미지는 리눅스용만 있는 것은 아닙니다. Windows도 자신의 커널이 제공하는 격리 기능(Job Object 등)을 이용한 **Windows Containers**를 지원하며, Windows Server Core 같은 Windows 전용 베이스 이미지로 완전히 네이티브하게 격리를 수행합니다.

다만 Docker Hub에 있는 이미지 대부분은 **리눅스 커널의 Namespace/Cgroup에 의존하는 Linux 컨테이너**입니다. Windows나 macOS는 이 기능 자체가 없는 커널이기 때문에, 그 위에서 Linux 컨테이너를 그대로 돌릴 방법이 없습니다.

---

## 격리는 어떻게 만들어지는가 — Namespace와 Cgroup

그렇다면 커널을 공유하면서도 어떻게 "서로 다른 컨테이너는 서로를 볼 수 없다"는 격리가 가능할까요? 답은 리눅스 커널이 원래 제공하던 두 가지 기능입니다.

- **네임스페이스(Namespace)**: 프로세스가 볼 수 있는 자원의 범위를 제한합니다. 예를 들어 PID 네임스페이스에 들어간 프로세스는 자기 자신이 PID 1인 것처럼 보이고, 호스트의 다른 프로세스는 아예 보이지 않습니다. 네트워크, 파일시스템 마운트, 호스트명 등도 각각 별도의 네임스페이스로 분리됩니다.
- **컨트롤 그룹(Control Group, Cgroup)**: 프로세스(그룹)가 사용할 수 있는 CPU, 메모리, 디스크 I/O 같은 자원의 **상한선**을 정합니다.

```text
격리된 것처럼 보이는 컨테이너
      │
      ├── Namespace  → "무엇을 볼 수 있는가" 제한
      │
      └── Cgroup      → "얼마나 쓸 수 있는가" 제한
```

즉 Docker는 새로운 격리 기술을 발명한 것이 아니라, 리눅스 커널에 이미 있던 Namespace/Cgroup을 **사용하기 쉽게 패키징한 도구**에 가깝습니다.

---

## 이미지(Image)와 컨테이너(Container)

Docker를 쓸 때 가장 먼저 마주치는 두 단어가 이미지와 컨테이너입니다.

- **이미지(Image)**: 애플리케이션 실행에 필요한 파일, 라이브러리, 설정을 읽기 전용으로 담아둔 템플릿입니다.
- **컨테이너(Container)**: 이미지를 실제로 실행한 상태, 즉 이미지의 **인스턴스**입니다.

```text
Image (설계도, 읽기 전용)
      │
   docker run
      ▼
Container (실행 중인 인스턴스)
```

같은 이미지로 컨테이너를 여러 개 띄울 수 있고, 각 컨테이너는 서로 영향을 주지 않습니다. 클래스(Class)와 객체(Object)의 관계와 비슷하다고 생각하면 이해가 쉽습니다.

> 🛠 **직접 해보기**
> Docker Desktop을 설치한 뒤 터미널에서 다음을 실행해보세요.
> ```bash
> docker run hello-world
> ```
> 실행 결과 메시지를 잘 읽어보면, Docker가 이미지를 로컬에서 찾고→ 없으면 원격 저장소에서 받아오고→ 컨테이너로 실행하는 과정을 순서대로 설명해줍니다. 몇 단계를 거쳤는지 확인해보세요. 막히면 다음 절의 이미지 빌드 과정을 참고하세요.
{: .prompt-tip }

---

## 이미지는 어떻게 만들어지는가 — Dockerfile

이미지는 `Dockerfile`이라는 텍스트 파일에 "어떤 순서로 무엇을 설치하고 어떤 명령을 실행할지"를 적어서 만듭니다.

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

각 줄은 다음과 같은 의미를 가집니다.

| 명령어 | 역할 |
| --- | --- |
| `FROM` | 베이스 이미지 지정 (여기서는 자바 17 런타임) |
| `WORKDIR` | 이후 명령이 실행될 작업 디렉터리 지정 |
| `COPY` | 호스트의 파일을 이미지 안으로 복사 |
| `EXPOSE` | 컨테이너가 사용할 포트를 문서화 (실제 개방은 `docker run -p`에서) |
| `ENTRYPOINT` | 컨테이너가 시작될 때 실행할 명령 |

Dockerfile을 빌드하면 이미지가 만들어집니다.

```bash
docker build -t my-app:1.0 .
```

여기서 `.`은 빌드 컨텍스트(현재 디렉터리)를 의미하고, `-t`는 이미지에 이름(태그)을 붙이는 옵션입니다.

---

## 컨테이너 실행과 기본 명령어

이미지를 만들었다면 이제 실행할 차례입니다.

```bash
docker run -d -p 8080:8080 --name my-app my-app:1.0
```

- `-d`: 백그라운드(detached)로 실행
- `-p 8080:8080`: 호스트 포트:컨테이너 포트를 연결
- `--name`: 컨테이너에 이름 부여

실행 중인 컨테이너를 관리할 때 자주 쓰는 명령어는 다음과 같습니다.

```bash
docker ps              # 실행 중인 컨테이너 목록
docker logs my-app      # 컨테이너 로그 확인
docker exec -it my-app sh  # 컨테이너 내부로 진입
docker stop my-app      # 컨테이너 정지
docker rm my-app        # 컨테이너 삭제
```

---

## 컨테이너는 기본적으로 휘발성이다

컨테이너 안에서 만든 파일은 컨테이너를 삭제하면 함께 사라집니다. 컨테이너는 "실행 중인 프로세스"에 가깝기 때문에, 데이터베이스처럼 데이터를 영구히 보존해야 하는 경우에는 **볼륨(Volume)**을 사용해 호스트 파일시스템(혹은 별도 저장 영역)에 데이터를 남겨야 합니다.

```text
Container (휘발성)
      │
      ▼
Volume (영속 저장소)
      │
      ▼
Host Filesystem
```

이 부분은 데이터베이스를 컨테이너로 운영할 때 반드시 마주치는 주제입니다.

---

## 마무리

Docker 컨테이너는 VM과 달리 커널을 새로 띄우지 않고, 리눅스가 원래 제공하던 **네임스페이스**와 **컨트롤 그룹**을 이용해 프로세스 수준에서 격리를 흉내 냅니다. 그래서 가볍고 빠르지만, 그만큼 호스트 커널을 그대로 공유한다는 제약도 함께 가집니다.

이미지는 실행 환경의 설계도이고, 컨테이너는 그 설계도로 찍어낸 실행 인스턴스입니다. Dockerfile은 이 설계도를 텍스트로 정의한 것이며, `docker build`로 이미지를 만들고 `docker run`으로 그 위에서 격리된 프로세스 하나를 띄우는 것이 Docker의 기본 흐름입니다.

다음 글에서는 여러 컨테이너를 하나의 YAML 파일로 함께 정의하고 실행하는 **[Docker Compose]({% post_url 2026-07-02-docker-compose %})**를 다뤄보겠습니다.
