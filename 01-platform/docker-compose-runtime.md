# Docker Compose 런타임 기준

이 문서는 초기 운영에서 사용할 Docker Compose 스택 분리 기준을 정의한다.

## 결론

Compose 파일은 두 종류로 분리한다.

```text
deploy/
├── compose.backend.yaml
├── compose.judge-agent.yaml
└── env/
    ├── backend.env.example
    ├── db.env.example
    ├── minio.env.example
    └── judge-agent.env.example
```

운영 배치:

```text
Backend VM:
- compose.backend.yaml

Judge VM 1~5:
- compose.judge-agent.yaml
```

백엔드는 단일 인스턴스로 시작한다.
judge-agent는 VM별 1개씩 실행하고, 각 agent가 `JUDGE_TOTAL_SLOTS=10` 같은 설정으로 동시 채점 수를 보고한다.

## Backend Stack

`compose.backend.yaml` 대상:

- `nginx`
- `api`
- `migrate`
- `postgres`
- `redis`
- `minio`
- `minio-init`
- `mail-worker`

초기에는 DB를 같은 Backend VM에 둔다.
대회 당일 병목이 확인되면 `postgres`, `redis`를 DB VM으로 분리하고 `DATABASE_URL`, `REDIS_URL`만 바꾼다.

예시:

```yaml
services:
  nginx:
    image: nginx:stable
    depends_on:
      - api
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ../frontend/dist:/usr/share/nginx/html:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped

  api:
    build:
      context: ../backend
    env_file:
      - ./env/backend.env
    depends_on:
      migrate:
        condition: service_completed_successfully
      minio-init:
        condition: service_completed_successfully
    expose:
      - "8000"
    restart: unless-stopped

  postgres:
    image: postgres:16
    env_file:
      - ./env/db.env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    restart: unless-stopped

  minio:
    image: minio/minio:latest
    command: ["server", "/data", "--console-address", ":9001"]
    env_file:
      - ./env/minio.env
    volumes:
      - minio_data:/data
    expose:
      - "9000"
      - "9001"
    restart: unless-stopped

  migrate:
    build:
      context: ../backend
    command: ["alembic", "upgrade", "head"]
    env_file:
      - ./env/backend.env
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"

  minio-init:
    image: minio/mc:latest
    env_file:
      - ./env/minio.env
      - ./env/backend.env
    depends_on:
      - minio
    entrypoint: ["/bin/sh", "-c"]
    command: ["until mc alias set local http://minio:9000 $${MINIO_ROOT_USER} $${MINIO_ROOT_PASSWORD}; do sleep 2; done; mc mb -p local/$${OBJECT_STORAGE_BUCKET} || true"]
    restart: "no"

  mail-worker:
    build:
      context: ../backend
    command: ["python", "-m", "app.workers.mail_worker"]
    env_file:
      - ./env/backend.env
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

## Judge Agent Stack

`compose.judge-agent.yaml` 대상:

- `judge-agent`

예시:

```yaml
services:
  judge-agent:
    build:
      context: ../judge_agent
    env_file:
      - ./env/judge-agent.env
    volumes:
      - judge_work:/var/lib/zerone-judge
    security_opt:
      - no-new-privileges:true
    pids_limit: 1024
    mem_limit: 20g
    cpus: 10
    restart: unless-stopped

volumes:
  judge_work:
```

judge-agent는 Docker socket을 mount하지 않는다.
현재 executor는 agent 컨테이너 내부 subprocess로 컴파일/실행한다.
따라서 judge-agent stack은 반드시 별도 judge VM에서 실행하고, VM 자체를 채점 전용 격리 경계로 둔다.

## Slot 기준

초기 설정:

```text
JUDGE_TOTAL_SLOTS=10
JUDGE_POLL_INTERVAL_SECONDS=1
JUDGE_DEFAULT_TIME_LIMIT_SECONDS=3
```

권장 운영:

- VM 4대면 총 40 slots
- VM 5대면 총 50 slots
- 바로 VM당 10 slots로 시작한다.
- 각 judge VM 권장 자원은 `10~12 vCPU`, `20GB RAM`이다.
- CPU 기준은 E5-2698 v4 기반 Proxmox host다.

slot은 동시에 실행되는 sandbox 컨테이너 수다.
컴파일 단계와 실행 단계를 모두 slot에 포함한다.

## env.example 생성 기준

`deploy/env/*.env.example`은 수동으로 따로 관리하지 않고, 백엔드 설정 스키마에서 자동 생성한다.
구현 시 `backend/app/core/settings.py` 같은 단일 설정 정의를 기준으로 아래 명령을 제공한다.

```bash
python -m app.tools.generate_env_examples
```

생성 대상:

```text
deploy/env/backend.env.example
deploy/env/db.env.example
deploy/env/minio.env.example
deploy/env/judge-agent.env.example
```

## Env 기준

### `backend.env`

```env
APP_ENV=production
PUBLIC_BASE_URL=https://judge.example.com
DATABASE_URL=postgresql+psycopg://zerone:password@postgres:5432/zerone
REDIS_URL=redis://redis:6379/0
OBJECT_STORAGE_BACKEND=minio
OBJECT_STORAGE_ENDPOINT=http://minio:9000
OBJECT_STORAGE_BUCKET=zerone
OBJECT_STORAGE_ACCESS_KEY=change-me
OBJECT_STORAGE_SECRET_KEY=change-me
OBJECT_STORAGE_SECURE=false
OBJECT_STORAGE_PRESIGN_TTL_SECONDS=900
LOCAL_OBJECT_STORAGE_ROOT=/var/lib/zerone-local-objects
STAFF_ACCESS_TOKEN_TTL_SECONDS=900
STAFF_REFRESH_TOKEN_TTL_SECONDS=1209600
PARTICIPANT_ACCESS_TOKEN_TTL_SECONDS=21600
OTP_TTL_SECONDS=300
INTERNAL_API_ALLOWED_CIDRS=100.64.0.0/10
MAX_SOURCE_CODE_BYTES=524288
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=change-me
SMTP_PASSWORD=change-me
SMTP_FROM_EMAIL=no-reply@example.com
SMTP_USE_TLS=true
```

### `db.env`

```env
POSTGRES_DB=zerone
POSTGRES_USER=zerone
POSTGRES_PASSWORD=change-me
```

### `minio.env`

```env
MINIO_ROOT_USER=zerone-minio
MINIO_ROOT_PASSWORD=change-me
```

### `judge-agent.env`

```env
APP_ENV=production
INTERNAL_API_BASE_URL=http://backend-vm-tailscale-ip:8000/api
JUDGE_NODE_NAME=judge-1
JUDGE_NODE_SECRET=change-me-per-node
JUDGE_TOTAL_SLOTS=10
JUDGE_WORK_ROOT=/var/lib/zerone-judge
JUDGE_AGENT_VERSION=0.1.0
JUDGE_POLL_INTERVAL_SECONDS=1
JUDGE_DEFAULT_TIME_LIMIT_SECONDS=3
JUDGE_RUN_ONCE=false
```

## Network

- 외부 공개 포트는 Backend VM의 `80`, `443`만 연다.
- FastAPI `8000`, PostgreSQL `5432`, Redis `6379`는 public internet에 노출하지 않는다.
- judge-agent는 Backend VM의 internal API로 outbound 접근만 한다.
- Backend VM과 Judge VM은 Tailscale 또는 사설망으로 연결한다.

## Migration

초기 운영에서는 `migrate` one-shot service가 `alembic upgrade head`를 실행한다.
`api`와 `mail-worker`는 `migrate` 완료 후 시작한다.

```bash
docker compose -f deploy/compose.backend.yaml run --rm api alembic upgrade head
docker compose -f deploy/compose.backend.yaml up -d
```

자동 migration on startup은 운영 장애 원인을 숨길 수 있으므로 MVP 이후에도 기본값으로 두지 않는다.

## Backup

세부 기준은 [백업과 복구 기준](./backup-and-restore.md)을 따른다.

최소 백업 대상:

- PostgreSQL dump
- PostgreSQL WAL archive
- MinIO object data and metadata
- env secret은 별도 안전한 위치에 보관

Redis는 cache와 wake-up signal 용도이므로 기본 복구 대상이 아니다.
단, Redis에 장기 상태를 저장하기 시작하면 백업 정책을 다시 정한다.

추천 백업 방식:

- 매일 1회 `pg_dump` full backup을 저장한다.
- WAL archiving을 켜서 대회 중 point-in-time recovery가 가능하게 한다.
- MinIO는 bucket replication 또는 `mc mirror`로 별도 디스크/VM에 복제한다.
- 대회 당일에는 시작 전 수동 백업과 종료 후 수동 백업을 추가한다.
- 백업 파일은 운영 VM과 다른 저장소에 보관한다.

## 로그

Compose 기본 json log가 무한 증가하지 않도록 log rotation을 설정한다.

```yaml
logging:
  driver: json-file
  options:
    max-size: "100m"
    max-file: "5"
```

위 설정은 모든 service에 공통 적용한다.
