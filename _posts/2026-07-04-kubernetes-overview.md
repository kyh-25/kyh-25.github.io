---
title: "Kubernetes 개요 - Compose로는 부족해지는 순간"
date: 2026-07-04 10:00:00 +0900
categories: [DevOps, Kubernetes]
tags: [kubernetes, k8s, orchestration, container]
level: beginner
series:
  name: Kubernetes 입문
  order: 1
math: false
mermaid: false
---

> 📚 **이 글을 읽기 전에**
> - [Docker Compose 개요]({% post_url 2026-07-02-docker-compose %}) — 여러 컨테이너를 하나의 파일로 정의하고 실행하는 방법을 다룹니다. 이 글은 Compose가 감당하지 못하는 지점에서 출발합니다.
{: .prompt-info }

## Compose로 충분했던 것, 충분하지 않은 것

지난 글까지는 컨테이너 여러 개를 `docker-compose.yml` 하나로 정의하고 `docker compose up`으로 실행했습니다. 로컬 개발 환경에서는 이걸로 충분합니다.

하지만 서비스를 실제로 운영한다고 생각해보겠습니다.

- 서버 한 대가 다운되면? Compose는 애초에 **한 대의 호스트** 안에서만 동작하는 도구라, 그 호스트가 죽으면 손쓸 방법이 없습니다.
- 트래픽이 몰려서 앱 컨테이너를 순간적으로 늘려야 한다면? `docker compose up --scale app=5`로 늘릴 수는 있지만, 여전히 같은 호스트 안에서만 늘어납니다. 호스트 한 대의 CPU/메모리 한계를 넘어설 수 없습니다.
- 컨테이너 하나가 죽으면? Compose는 기본적으로 재시작 정책(`restart: always`)을 넘어서는 복구 로직이 없습니다.
- 새 버전을 배포하는 동안 트래픽이 끊기지 않게 하려면? Compose에는 이런 무중단 배포 개념이 없습니다.

정리하면 Compose는 "한 대의 서버 안에서 여러 컨테이너를 편하게 관리하는 도구"이고, **여러 대의 서버에 걸쳐 컨테이너를 분산 배치하고, 장애를 자동으로 복구하고, 트래픽에 따라 자동으로 늘리고 줄이는 것**은 Compose의 역할 밖입니다. 이 역할을 하는 도구가 **Kubernetes**입니다.

---

## Kubernetes란?

**Kubernetes**(줄여서 **K8s**)는 여러 대의 서버를 하나의 **클러스터(Cluster)**로 묶고, 그 위에서 컨테이너들을 자동으로 배치·복구·확장해주는 오케스트레이션 시스템입니다.

```text
Docker Compose
      │
   한 대의 서버 안에서 여러 컨테이너 관리
      │
      ▼
Kubernetes
      │
   여러 대의 서버(클러스터)에 걸쳐 컨테이너 관리
```

핵심은 "컨테이너를 어느 서버에 띄울지"를 사람이 정하지 않는다는 점입니다. 사람은 "이런 컨테이너가 몇 개 떠 있어야 한다"는 **원하는 상태(Desired State)**만 선언하고, Kubernetes가 그 상태를 계속 유지하도록 알아서 서버를 골라 배치하고, 문제가 생기면 스스로 복구합니다.

---

## 클러스터의 구조 — Control Plane과 Node

Kubernetes 클러스터는 역할이 다른 두 종류의 서버로 구성됩니다.

```text
Cluster
  │
  ├── Control Plane (두뇌)
  │     ├── API Server    — 모든 요청의 진입점
  │     ├── Scheduler     — 어느 Node에 컨테이너를 배치할지 결정
  │     ├── Controller Manager — 원하는 상태와 실제 상태를 계속 비교/조정
  │     └── etcd          — 클러스터의 모든 상태를 저장하는 저장소
  │
  └── Worker Node (일꾼) × N
        └── 실제 컨테이너가 여기서 실행됨
```

`kubectl`(쿠버네티스 명령줄 도구)로 무언가를 요청하면, 그 요청은 항상 Control Plane의 API Server를 거칩니다. API Server는 요청받은 "원하는 상태"를 etcd에 저장하고, Scheduler는 그 컨테이너를 어느 Worker Node에서 실행할지 결정하고, Controller Manager는 실제로 그 상태가 유지되고 있는지 끊임없이 확인합니다.

> ❓ **생각해보기**
> "끊임없이 확인하고 맞춘다"는 표현에 주목해보세요. `docker compose up`은 명령어를 한 번 실행하면 끝나는 일회성 동작인데, Kubernetes는 왜 계속 상태를 확인해야 할까요? Worker Node 하나가 갑자기 다운되는 상황을 생각하면서 먼저 정리해보세요.
{: .prompt-warning }

---

## 컨테이너가 아니라 Pod

Kubernetes는 컨테이너를 직접 다루지 않습니다. 대신 **Pod**라는 단위로 다룹니다.

**Pod**는 Kubernetes에서 배포할 수 있는 최소 단위로, 컨테이너 하나 또는 여러 개를 묶은 것입니다. 같은 Pod 안의 컨테이너들은 네트워크(같은 IP, `localhost`로 서로 통신)와 스토리지를 공유합니다.

```text
Pod
 ├── Container (app)
 └── Container (sidecar, 선택 사항)

같은 Pod 안 컨테이너는
IP를 공유 (localhost로 통신 가능)
```

대부분의 경우 Pod 하나에는 컨테이너 하나만 들어갑니다(예: Spring Boot 앱 컨테이너 하나). 로그 수집기처럼 메인 컨테이너를 보조하는 컨테이너를 같이 묶어야 할 때만 Pod 안에 컨테이너를 여러 개 두는 **사이드카 패턴**을 씁니다. 이전 글에서 다룬 볼륨의 사이드카 패턴과 같은 개념이 여기서도 등장하는 셈입니다.

Pod는 언제든지 사라지고 새로 만들어질 수 있는 존재로 취급됩니다. Node가 다운되거나, 컨테이너가 죽거나, 새 버전을 배포하면 기존 Pod는 삭제되고 새 Pod가 다른 곳에 생성됩니다. 그때마다 Pod의 IP는 바뀝니다.

---

## Deployment — 원하는 상태를 선언하기

"이 이미지로 Pod를 3개 유지해줘"라는 선언을 담는 오브젝트가 **Deployment**입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

`replicas: 3`이 바로 원하는 상태입니다. Pod 하나가 죽으면 Controller Manager가 이를 감지하고 즉시 새 Pod를 만들어 replicas 수를 3으로 되돌립니다. 이것이 앞서 나온 "끊임없이 확인하고 맞춘다"의 실체이고, Kubernetes를 쓰는 가장 큰 이유인 **자가 치유(Self-healing)**입니다.

```bash
kubectl apply -f deployment.yaml
```

이 명령어로 이 선언을 클러스터에 적용합니다. `docker compose up`처럼 한 번 실행하고 끝나는 것이 아니라, 이후로도 Kubernetes는 이 선언된 상태를 계속 지키려고 합니다.

> 🛠 **직접 해보기**
> 로컬에 minikube나 kind로 간단한 클러스터를 하나 띄우고, 위 Deployment(이미지는 원하는 걸로 바꿔도 됩니다)를 `kubectl apply -f`로 적용해보세요. `kubectl get pods`로 Pod 3개가 뜨는지 확인한 뒤, `kubectl delete pod <pod 이름>`으로 그중 하나를 강제로 지워보고 다시 `kubectl get pods`를 실행해보세요. 무슨 일이 일어나는지 확인해보세요.
{: .prompt-tip }

---

## Service — Pod의 IP가 바뀌어도 안정적으로 접근하기

Pod는 다시 만들어질 때마다 IP가 바뀝니다. 그렇다면 다른 Pod나 외부에서는 이 Pod에 어떻게 접근해야 할까요?

이 문제를 해결하는 것이 **Service**입니다. Service는 라벨(`labels`)이 일치하는 Pod들을 묶어 하나의 안정적인 이름과 IP로 노출시키고, 그 뒤에서 여러 Pod로 트래픽을 분산합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

`selector.app: my-app`이 Deployment의 `labels.app: my-app`과 일치하는 Pod들을 자동으로 찾아 연결합니다. 다른 Pod에서는 개별 Pod의 IP가 아니라 `my-app`이라는 Service 이름으로 접근하면, 그 뒤에 떠 있는 Pod 중 하나로 요청이 전달됩니다.

이전 글에서 Compose가 서비스 이름을 DNS처럼 등록해 컨테이너 간 통신을 가능하게 했던 것과 같은 발상입니다. 다만 Compose의 서비스 이름은 컨테이너 하나에 대응했다면, Kubernetes의 Service는 **여러 개의 교체 가능한 Pod 앞에 놓인 로드밸런서**에 가깝습니다.

---

## 볼륨은 Kubernetes에서 어떻게 관리되는가

이전 글에서 컨테이너가 휘발성이라 볼륨이 필요하다고 했습니다. Kubernetes에서는 이 문제가 한 단계 더 복잡해집니다. Pod가 죽으면 **어느 Node에 다시 뜰지 알 수 없기** 때문입니다. Docker의 Bind Mount처럼 "이 서버의 이 디스크 경로"를 그대로 쓰는 방식은, Pod가 다른 Node로 옮겨가는 순간 그 데이터에 접근할 방법이 없어집니다.

```text
Docker: 컨테이너가 죽어도 보통 같은 호스트에서 다시 뜸
      │
      ▼
Kubernetes: Pod가 죽으면 어느 Node에 다시 뜰지 알 수 없음
      │
      ▼
특정 Node의 로컬 디스크에 의존하면 안 됨
```

그래서 Kubernetes는 저장소를 **PersistentVolume(PV)**과 **PersistentVolumeClaim(PVC)**로 나눠서 다룹니다.

- **PersistentVolume(PV)**: 특정 Node의 로컬 디스크가 아니라, 클러스터 전체에서 네트워크로 접근 가능한 저장소(클라우드 디스크, NFS 등)를 클러스터 차원의 자원으로 등록해둔 것입니다.
- **PersistentVolumeClaim(PVC)**: Pod가 "이런 용량/속성의 저장소가 필요하다"고 요청하는 것입니다. Pod는 PV를 직접 참조하지 않고 PVC를 통해 요청하면, Kubernetes가 조건에 맞는 PV를 찾아 연결(바인딩)해줍니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

이렇게 만든 PVC를 Pod의 `volumes`에서 참조하면, 그 Pod가 어느 Node에 뜨든 네트워크를 통해 같은 저장소에 연결됩니다.

```text
Pod가 PVC로 저장소를 요청
      │
      ▼
Kubernetes가 조건에 맞는 PV를 찾아 바인딩
      │
      ▼
Pod가 어느 Node에 뜨든 PV는 네트워크로 그대로 연결됨
```

> ❓ **생각해보기**
> 지금까지 본 Deployment의 Pod들은 서로 교체 가능한 익명 존재였습니다(어떤 Pod가 요청을 받아도 상관없음). 그런데 데이터베이스처럼 각 인스턴스가 자기만의 데이터를 가지고 있어야 하는 경우에도 Deployment로 충분할까요? 각 Pod가 고유한 이름과 전용 볼륨을 계속 유지해야 한다는 점을 생각하며 정리해보세요.
{: .prompt-warning }

데이터베이스처럼 각 인스턴스가 고유해야 하는 경우엔 Deployment 대신 **StatefulSet**을 사용합니다. StatefulSet은 Pod마다 고정된 이름(`db-0`, `db-1`, `db-2`)을 부여하고, 각 Pod에 전용 PVC를 만들어 Pod가 재생성되어도 항상 같은 볼륨에 다시 연결되도록 보장합니다. Deployment가 "교체 가능한 여러 개"를 다룬다면, StatefulSet은 "각자 고유한 여러 개"를 다루는 셈입니다.

---

## 여러 DB 인스턴스는 어떻게 일관성을 유지하는가

Pod마다 전용 볼륨을 준다고 해서 데이터 일관성 문제가 저절로 풀리는 것은 아닙니다. 여러 DB 인스턴스가 각자 다른 데이터를 들고 있으면 당연히 어긋날 수 있고, 반대로 하나에만 띄우면 여러 대로 나눈 의미가 없습니다. 이 문제는 **Kubernetes가 아니라 DB 소프트웨어 자체의 복제(Replication) 기능**이 해결합니다.

일반적인 구조는 Pod 하나를 **Primary(쓰기 전담)**로, 나머지를 **Replica(읽기 전담)**로 두는 것입니다.

```text
db-0 (Primary) ← 쓰기 요청은 여기로만
      │
      ├── 변경 내용을 계속 스트리밍
      ▼
db-1 (Replica)   db-2 (Replica)   ← 읽기 요청만 분산 처리
```

앱 컨테이너처럼 아무 Pod나 요청을 받아도 되는 구조가 아니라, 쓰기 권한은 Primary 하나에만 있고 나머지는 그 변경 내역을 실시간으로 받아 반영하는 구조입니다. 그래서 "동시에 여러 곳에 떠서 데이터가 어긋나는" 상황이 아니라, 애초에 쓰기 가능한 곳이 하나뿐입니다. Kubernetes는 이 Primary/Replica Pod들이 각자 안정적인 이름과 볼륨을 유지하도록 자리만 깔아줄 뿐, 무엇이 Primary인지·어떻게 동기화할지는 PostgreSQL의 스트리밍 리플리케이션, MySQL의 Group Replication처럼 DB 자체의 기능이 담당합니다.

그렇다면 여러 대에 띄우는 이유는 쓰기 부하 분산이 아니라 **장애 대비**(Primary가 죽으면 Replica 중 하나가 새 Primary로 승격)와 **읽기 부하 분산**입니다. 여러 곳에서 동시에 쓰기를 받는 진짜 멀티 라이터 구조는 CockroachDB 같은 별도로 설계된 분산 데이터베이스의 영역이고, 일반적인 PostgreSQL/MySQL 클러스터링은 해당하지 않습니다.

---

## Pod가 다른 Node로 옮겨가면 데이터도 복사되는가

Pod 하나가 다른 Node로 재배치되는 것과, StatefulSet을 늘려 새 Replica를 만드는 것은 데이터 이동 관점에서 서로 다릅니다.

- **같은 Pod가 다른 Node에서 재시작될 때**: PersistentVolume은 애초에 네트워크로 접근하는 저장소(클라우드 디스크 등)이므로, 데이터를 복사하는 게 아니라 새 Node에서 **같은 네트워크 볼륨에 다시 연결(재부착)**할 뿐입니다. 외장 하드를 이 컴퓨터에서 뽑아 저 컴퓨터에 꽂는 것과 비슷해서 데이터 자체는 이동하지 않습니다. (다만 클라우드 디스크는 보통 같은 가용 영역 안에서만 재부착할 수 있어서, Kubernetes 스케줄러가 이 제약을 고려해 Node를 고릅니다.)
- **StatefulSet을 3개에서 4개로 늘려 새 Replica를 만들 때**: 이건 실제로 기존 데이터를 처음부터 복사해서 새 Replica를 "시딩"해야 합니다. 데이터가 크면 이 초기 동기화 자체가 무겁고 시간이 걸리는 작업이 맞습니다. 다만 이 복사는 Kubernetes가 임의로 수행하는 것이 아니라, DB 복제 도구가 명시적으로 수행하는 예상된 작업입니다.

---

## 실무에서는 DB를 K8s에 직접 올리는 것이 일반적일까

여기까지 보면 StatefulSet과 PersistentVolume만 있으면 DB도 당연히 K8s 위에 직접 운영하면 될 것 같지만, 실무에서는 **DB를 클러스터 바깥에 따로 두는 경우가 여전히 흔합니다.**

지금까지 본 것처럼 K8s 위에서 DB를 제대로 운영하려면 StatefulSet, PersistentVolume, Primary/Replica 복제 설정까지 직접 신경 써야 하고, 백업이나 장애 복구(Failover) 같은 DB 특유의 운영 부담은 Kubernetes가 대신 해결해주지 않습니다. 그래서 많은 팀은 AWS RDS, Google Cloud SQL 같은 **관리형 DB 서비스**를 클러스터 바깥에 두고, K8s 위의 앱만 네트워크로 그 DB에 접속하는 구조를 택합니다. K8s는 "언제든 죽고 다시 만들어져도 상관없는" 워크로드에 잘 맞게 설계되어 있어서, "절대 데이터가 어긋나면 안 되는" DB와는 철학이 살짝 다르기 때문입니다.

물론 CloudNativePG 같은 **Operator**(Primary 승격, 백업, 장애 복구 같은 DB 운영 노하우를 Kubernetes 컨트롤러로 코드화한 도구)를 쓰면 K8s 안에서도 프로덕션 DB를 충분히 운영할 수 있고, 실제로 그렇게 하는 회사도 많습니다. 다만 이는 StatefulSet만으로 되는 것이 아니라 전용 Operator를 갖췄을 때 선택하는 심화 경로에 가깝고, 로컬 개발이나 소규모 환경에서는 K8s 안에 DB를 그냥 띄우는 것도 흔합니다.

---

## Compose와 Kubernetes 비교

| 구분 | Docker Compose | Kubernetes |
| --- | --- | --- |
| 관리 범위 | 서버 한 대 | 여러 대(클러스터) |
| 실행 단위 | 컨테이너 | Pod (컨테이너 묶음) |
| 상태 관리 | 명령 실행 시점에만 적용 | 원하는 상태를 계속 유지(Self-healing) |
| 장애 대응 | 재시작 정책 수준 | Pod/Node 장애 자동 복구 |
| 서비스 간 통신 | 서비스 이름 DNS (컨테이너 단위) | Service (Pod 여러 개 대상 로드밸런싱) |
| 데이터 저장 | Named Volume (호스트 한 대에 종속) | PersistentVolume (네트워크 저장소, Node에 무관) |
| 적합한 환경 | 로컬 개발, 소규모 운영 | 프로덕션, 대규모 운영 |

---

## 마무리

Kubernetes는 컨테이너 하나하나를 직접 관리하는 도구가 아니라, 여러 대의 서버를 하나의 클러스터로 묶고 그 위에서 "원하는 상태"를 선언하면 알아서 유지해주는 시스템입니다. Pod는 언제든 사라지고 다시 만들어질 수 있는 존재이고, Deployment는 그 개수를 계속 유지시켜주며, Service는 계속 바뀌는 Pod의 IP 대신 안정적인 접근 지점을 제공합니다. 저장소도 마찬가지로, Pod가 어느 Node에 뜨든 문제없이 접근할 수 있도록 PersistentVolume이 특정 Node가 아닌 클러스터 차원에서 관리됩니다.

Compose가 "한 대의 서버 안에서 여러 컨테이너를 편하게"였다면, Kubernetes는 "여러 대의 서버에 걸쳐 컨테이너가 항상 원하는 상태로 떠 있게"에 가깝습니다. 이번 글에서는 개요와 핵심 오브젝트(Pod, Deployment, Service, PersistentVolume)만 다뤘고, 실제로 다루다 보면 ConfigMap/Secret로 설정을 분리하거나 Ingress로 외부 트래픽을 라우팅하는 것처럼 더 다뤄야 할 개념들이 이어집니다.
