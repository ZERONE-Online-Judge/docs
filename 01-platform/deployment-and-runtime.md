# 배포 및 런타임 구성

## 목적

이 문서는 백엔드 기술 스택, 정적 웹 배포 방식, Docker Compose 적용 범위, 채점 샌드박스 실행 방식, Proxmox 운영 서버 배치 기준을 정리한다.

## 현재 전제

- 운영 환경은 Proxmox 기반 단일 물리 서버 또는 소수 VM 구성이다.
- 사용 가능한 리소스는 약 `70 vCPU`, 최대 `80 vCPU`, `240GB RAM`, 최대 `256GB RAM`이다.
- 대회 당일 예상 운영 시간은 `6~8시간`이다.
- 대회 당일 예상 동시 접속자는 `100명 이상`이다.
- 프론트엔드는 정적 빌드 결과물을 Nginx로 배포한다.
- 백엔드는 FastAPI를 기준으로 한다.
- 채점 샌드박스 실행 제어는 Python 코드에서 담당한다.

## 기술 스택 기준

### Web

- 프론트엔드는 Vite 기반 React 앱으로 개발한다.
- 운영 배포 시 정적 파일은 Nginx가 직접 서빙한다.
- Nginx는 `/api` 요청을 FastAPI로 reverse proxy 한다.
- 브라우저가 직접 접근 가능한 공개 진입점은 Nginx 하나로 모은다.

### Backend

- API 서버는 FastAPI를 사용한다.
- 운영 실행은 ASGI 서버 기반으로 구성한다.
- 인증, 권한, 대회 관리, 제출 접수, 채점 요청 생성, 운영 API를 담당한다.
- 제출 코드를 직접 실행하지 않는다.

### Database / Queue

- 주 데이터 저장소는 PostgreSQL을 기준으로 한다.
- 제출 큐, 캐시, 짧은 상태 공유에는 Redis 사용을 우선 검토한다.
- 문제 리소스, 테스트케이스, 제출 코드, 채점 산출물은 MinIO에 저장한다.
- 채점 요청은 DB 트랜잭션과 큐 적재 사이의 유실 방지 장치를 가져야 한다.
- 초기 채점 큐 원장은 PostgreSQL에 둔다.
- Redis는 실시간 알림, scoreboard 캐시, dispatcher wake-up signal 같은 보조 용도로 사용한다.

### Judge

- `judge-dispatcher`는 채점 큐, lease, heartbeat, 재할당을 담당한다.
- `judge-agent`는 각 채점 VM에서 실행된다.
- 제출별 Docker 샌드박스 컨테이너는 `judge-agent`의 Python 코드가 직접 생성, 실행, 제한, 종료, 정리한다.
- 제출별 샌드박스 컨테이너를 Docker Compose 서비스로 정의하지 않는다.

## Docker Compose 사용 원칙

Docker Compose는 운영 서버의 고정 서비스 배포와 재시작 관리에 사용한다.

Compose로 관리하는 대상:

- `nginx`
- `fastapi`
- `postgres`
- `redis`
- `judge-dispatcher`
- `judge-agent` 프로세스
- 모니터링 또는 로그 수집 보조 서비스

Compose로 관리하지 않는 대상:

- 제출 1건마다 생성되는 채점 샌드박스 컨테이너
- 테스트케이스별 임시 실행 컨테이너
- 채점 작업별 timeout, kill, cleanup 단위

이 구분은 고정 서비스와 일회성 격리 실행 단위를 분리하기 위한 것이다.
Compose는 오래 떠 있어야 하는 서비스에 사용하고, 제출별 컨테이너는 Python judge-agent가 Docker API 또는 Docker CLI로 직접 제어한다.

## 권장 VM 배치

초기 운영은 아래처럼 VM 단위 경계를 나눈다.

```text
Proxmox Host
├── VM: web
│   ├── nginx
│   └── fastapi
├── VM: db
│   ├── postgres
│   └── redis
├── VM: judge-1
│   └── judge-agent + Docker sandbox runner
└── VM: judge-n
    └── judge-agent + Docker sandbox runner
```

운영 단순성을 우선할 경우 `web`, `db`, `judge`를 더 적은 VM으로 시작할 수 있다.
다만 제출 코드 실행 영역인 judge VM은 web/api VM과 분리하는 것을 원칙으로 한다.

### 초기 리소스 배분안

```text
web VM
- 8~12 vCPU
- 16~32GB RAM

db VM
- 8~12 vCPU
- 32~64GB RAM

judge VM 전체
- 나머지 vCPU/RAM 배치
- 초기 동시 채점 slot 40~60부터 검증
```

`240GB RAM` 환경에서는 메모리보다 CPU, Docker 격리 오버헤드, DB I/O, 큐 처리량, 채점 언어별 실행 시간이 먼저 병목이 될 가능성이 크다.
Python/Java 제출이 많으면 C/C++ 제출보다 slot당 메모리와 실행 시간이 증가하므로 실제 slot 수는 부하 테스트로 조정한다.

## 네트워크 경계

- 외부 사용자는 Nginx를 통해서만 서비스에 접근한다.
- FastAPI는 외부에 직접 공개하지 않는다.
- PostgreSQL, Redis는 외부에 공개하지 않는다.
- judge VM은 공개 인터넷에서 직접 접근할 수 없게 한다.
- `web/api`, `judge-dispatcher`, `judge-agent` 사이 통신은 Tailscale 또는 사설망 기준으로 제한한다.
- 공개 채점 서버 현황 페이지는 내부 IP, Tailscale 식별자, 상세 할당 상태 같은 민감 정보를 노출하지 않는다.

## 권장 파일 구조

초기 구현 시 아래 구조를 기준으로 둔다.

```text
zerone_online_judge/
├── frontend/
│   ├── src/
│   └── dist/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── core/
│   │   ├── domains/
│   │   └── infrastructure/
│   ├── migrations/
│   └── Dockerfile
├── judge_dispatcher/
│   ├── app/
│   └── Dockerfile
├── judge_agent/
│   ├── app/
│   │   ├── main.py
│   │   ├── sandbox/
│   │   ├── runners/
│   │   └── clients/
│   ├── images/
│   │   ├── c/
│   │   ├── cpp/
│   │   ├── python/
│   │   └── java/
│   └── Dockerfile
├── deploy/
│   ├── compose.backend.yaml
│   ├── compose.judge-agent.yaml
│   ├── nginx/
│   │   └── default.conf
│   └── env/
│       ├── backend.env.example
│       ├── db.env.example
│       ├── minio.env.example
│       └── judge-agent.env.example
└── docs/
```

## Compose 파일 구성 기준

Compose는 backend stack과 judge-agent stack을 분리한다.
세부 env와 예시는 [Docker Compose 런타임 기준](./docker-compose-runtime.md)을 따른다.

```text
deploy/
├── compose.backend.yaml
└── compose.judge-agent.yaml
```

Backend VM에서는 아래처럼 실행한다.

```bash
docker compose -f deploy/compose.backend.yaml up -d
```

Judge VM에서는 아래처럼 실행한다.

```bash
docker compose -f deploy/compose.judge-agent.yaml up -d
```

운영 환경에서는 DB를 별도 VM으로 분리할 수 있다.
이 경우 `postgres`, `redis` 서비스는 web VM의 Compose에서 제거하고, `DATABASE_URL`, `REDIS_URL`을 사설망 주소로 설정한다.

## Judge agent 배포 기준

judge VM에서는 `judge-agent` 자체만 고정 서비스로 배포한다.

```yaml
services:
  judge-agent:
    build:
      context: ../judge_agent
    env_file:
      - ./env/judge-agent.env
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - judge_work:/var/lib/zerone-judge
    restart: unless-stopped

volumes:
  judge_work:
```

위 구성은 구현 난이도가 낮지만, Docker socket을 컨테이너에 마운트하면 `judge-agent`가 호스트 Docker daemon을 제어할 수 있다.
따라서 judge VM은 web/api VM과 분리하고, judge-agent 컨테이너 또는 프로세스 권한을 최소화해야 한다.

보안을 더 강화하려면 judge-agent를 컨테이너가 아니라 systemd 서비스로 실행하고, rootless Docker 또는 별도 제한 계정을 검토한다.

## Sandbox 실행 정책

제출별 샌드박스 컨테이너는 다음 제약을 가져야 한다.

- 외부 네트워크 비활성화
- 호스트 파일시스템 접근 금지
- read-only root filesystem 우선 검토
- 임시 작업 디렉터리만 제한적으로 mount
- CPU 제한
- 메모리 제한
- 프로세스 수 제한
- 출력 크기 제한
- wall time 제한
- 실행 종료 후 컨테이너와 임시 파일 정리

Python judge-agent는 제출 실행 시 다음 값을 명시적으로 결정한다.

- 언어별 실행 이미지
- 컴파일 명령
- 실행 명령
- timeout
- memory limit
- output limit
- testcase 입력/출력 전달 방식
- 채점 결과 상태 코드

## 저장소 마운트 기준

- web/api VM은 MinIO에 문제 리소스, 테스트케이스, 제출 코드, 채점 산출물을 저장한다.
- judge VM은 테스트케이스와 제출 작업 디렉터리에 접근해야 한다.
- 테스트케이스 원본 저장소는 공개 Nginx document root 아래에 두지 않는다.
- judge-agent가 필요한 파일은 내부 API가 제공하는 메타데이터와 MinIO 접근 정보로 가져간다.
- DB에는 실제 절대 경로에 의존하지 않는 storage key와 메타데이터를 저장한다.

## 운영 모드

### 단순 초기 운영

```text
web VM:
- nginx
- fastapi
- postgres
- redis
- judge-dispatcher

judge VM:
- judge-agent
- Docker sandbox runner
```

장점은 배포와 장애 대응이 단순하다는 점이다.
단점은 DB와 web/api가 같은 VM에 있어 대회 당일 I/O 병목을 분리하기 어렵다는 점이다.

### 권장 운영

```text
web VM:
- nginx
- fastapi

db VM:
- postgres
- redis

judge VM:
- judge-agent
- Docker sandbox runner
```

대회 당일 안정성을 우선할 경우 이 구성을 기준으로 한다.

## Kubernetes 도입 기준

현재 예상 규모에서는 Kubernetes를 초기 필수 조건으로 두지 않는다.

Kubernetes 도입을 검토할 조건:

- 여러 물리 서버로 judge node를 계속 늘려야 하는 경우
- 서비스별 rolling update, autoscaling, service discovery가 운영 병목이 되는 경우
- 대회 외 상시 트래픽이 커져 수평 확장이 필요해지는 경우
- 전담 운영자가 생겨 클러스터 운영 복잡도를 감당할 수 있는 경우

초기에는 Proxmox VM 경계와 Docker Compose로 운영하고, 병목이 명확해진 뒤 Kubernetes 또는 Nomad 같은 오케스트레이션 도구를 검토한다.

## 관련 문서

- [백엔드 요구사항](./backend-requirements.md)
- [서버 요구사항](./server-requirements.md)
- [프론트 요구사항](./frontend-requirements.md)
- [접속 플로우](./access-flow.md)
