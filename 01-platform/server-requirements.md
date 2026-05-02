# 서버 요구사항

## 서버 계층

- `web/api`
  - 공개 인터넷에 노출되는 계층
  - 사용자 요청, 인증, 운영 API 담당
- `judge-dispatcher`
  - 비공개 내부 계층
  - FIFO 큐, lease, heartbeat, 재할당 담당
- `judge-agent`
  - 각 채점 서버에서 동작
  - 실제 컴파일/실행/채점 담당

## 네트워크 원칙

- 사용자는 `web/api`에만 직접 접속해야 한다.
- 채점 서버는 공개 인터넷에 직접 노출하지 않는다.
- `web/api`, `judge-dispatcher`, `judge-agent` 사이 통신은 Tailscale 사설망 기준으로 설계한다.
- Nginx는 정적 웹 배포와 API reverse proxy의 공개 진입점 역할을 맡는다.
- FastAPI, PostgreSQL, Redis, judge 계층은 외부 공개망에 직접 노출하지 않는다.

## 채점 실행 요구사항

- 각 제출은 Docker 샌드박스에서 실행해야 한다.
- 외부 네트워크 접근은 차단해야 한다.
- 호스트 접근은 차단해야 한다.
- 시간/메모리/출력 제한을 강제해야 한다.
- 지원 언어는 C99, C++17, Python 3.13, Java 8 이다.

## 큐 및 스케줄링 요구사항

- FIFO 기준은 `submitted_at ASC, submission_id ASC`
- heartbeat 기반으로 node 상태를 수집해야 한다.
- lease 만료 시 자동 재할당이 가능해야 한다.
- 중복 실행/중복 반영을 막는 idempotency 장치가 필요하다.
- 초기 구현은 DB 기반 채점 큐를 기준으로 한다.
- 제출 저장과 채점 job 생성은 같은 DB transaction 안에서 처리한다.
- dispatcher는 pending job을 DB에서 lease 방식으로 가져가고, Redis는 실시간 알림/캐시/대시보드 보조 용도로 우선 사용한다.

### 채점 큐 데이터 기준

- `submissions`: 제출 원본과 제출 시각, 제출자, 문제, 언어 정보를 저장한다.
- `judge_jobs`: 채점해야 할 작업의 현재 상태와 FIFO 순서를 저장한다.
- `judge_job_attempts`: 각 실행 시도와 assigned node, 시작/종료 시각, 실패 사유를 저장한다.
- `judge_nodes`: judge-agent의 heartbeat, slot, 가용 상태를 저장한다.
- 결과 반영은 `attempt_id` 또는 이에 준하는 idempotency key를 기준으로 중복 반영을 막는다.

## 운영성 요구사항

- 채점 서버 수를 조정할 수 있어야 한다.
- 서버별 slot 수를 조정할 수 있어야 한다.
- 장애 node를 자동으로 비할당 처리할 수 있어야 한다.
- 공개 현황 페이지와 내부 운영 대시보드가 필요하다.
- 고정 서비스 배포는 Docker Compose를 기준으로 시작한다.
- 제출별 Docker 샌드박스 컨테이너는 Docker Compose가 아니라 `judge-agent`의 Python 코드가 직접 제어한다.

## 비기능 요구사항

- 안정성
  - 제출 유실 방지
  - 일부 node 장애 시 전체 서비스 유지
- 성능
  - 수평 확장 가능
  - 빠른 큐 적재
- 보안
  - 격리 실행
  - 최소 권한
- 운영성
  - 모니터링, 백업, 복구, 알람 체계 필요

## 관련 문서

- [배포 및 런타임 구성](./deployment-and-runtime.md)
