# 채점 프로토콜

이 문서는 API 서버, judge-dispatcher, judge-agent 사이의 초기 구현 계약을 정의한다.
기본 방향은 dispatcher-assigned scheduling이다.

## 결론

채점 job의 배정 주체는 `judge-dispatcher`다.
다만 dispatcher가 agent로 직접 push하지 않고, DB에 배정 상태를 기록한 뒤 agent가 자기 node에 할당된 job만 가져간다.

이 방식의 장점:

- agent VM inbound network를 열 필요가 없다.
- dispatcher가 FIFO, slot, node 상태, lease 정책을 한 곳에서 결정한다.
- agent는 실행과 결과 보고에 집중한다.
- 장애 시 DB lease 기준으로 재할당할 수 있다.

## 구성 요소 책임

### FastAPI

- 참가자 제출을 받는다.
- `submissions`와 `judge_jobs`를 같은 DB transaction 안에서 생성한다.
- judge-agent 내부 API를 제공한다.
- 채점 결과를 idempotent하게 반영한다.
- 운영자용 큐/노드 조회 API를 제공한다.

### judge-dispatcher

- pending job을 FIFO 기준으로 조회한다.
- heartbeat가 살아 있고 schedulable인 judge node를 선택한다.
- node의 free slot을 고려해 job을 배정한다.
- `judge_jobs`와 `judge_job_attempts`에 lease를 기록한다.
- lease가 만료된 job을 회수하고 재할당한다.
- lost node를 감지한다.

### judge-agent

- 주기적으로 heartbeat를 전송한다.
- 자기 node에 배정된 job을 가져온다.
- 제출 코드, 문제 메타데이터, 테스트케이스를 내부 API 또는 storage adapter로 가져온다.
- Docker sandbox를 생성해 컴파일/실행/채점한다.
- 결과를 lease token과 함께 보고한다.

## 상태 흐름

### Submission

```text
waiting -> preparing -> judging -> final result
waiting -> canceled
preparing/judging -> system_error
```

final result:

- `accepted`
- `presentation_error`
- `wrong_answer`
- `time_limit_exceeded`
- `memory_limit_exceeded`
- `output_limit_exceeded`
- `runtime_error`
- `compile_error`
- `system_error`
- `canceled`

### JudgeJob

```text
pending -> assigned -> running -> succeeded
pending -> assigned -> running -> failed -> pending
pending -> assigned -> expired -> pending
pending/assigned/running -> canceled
```

원칙:

- `succeeded`는 submission final result 반영 성공을 뜻한다.
- 채점 결과가 WA/CE/RE여도 job 자체는 `succeeded`다.
- infra 오류나 agent 오류로 결과를 확정할 수 없을 때만 `failed`다.

## FIFO 기준

기본 정렬:

```sql
priority DESC, queue_position ASC
```

`queue_position`은 제출 생성 transaction에서 증가하는 sequence 값이다.
동일 우선순위에서는 `submitted_at ASC, submission_id ASC`와 같은 의미가 되도록 생성한다.

재채점 job은 기본 제출보다 낮은 priority 또는 별도 queue group으로 둘 수 있다.
MVP에서는 재채점도 같은 큐를 쓰되 `priority = -10`으로 둔다.

## Lease 정책

기본값:

| 항목 | 값 |
| --- | --- |
| agent heartbeat interval | 5초 |
| node lost timeout | 30초 |
| assignment lease ttl | 120초 |
| running lease extend interval | 30초 |
| max retry count | 3 |

agent는 실행 중인 job의 lease를 heartbeat와 함께 연장한다.
dispatcher는 `lease_expires_at < now()`인 `assigned`, `running` job을 만료 처리한다.

## 내부 API

내부 API는 외부 Nginx에 노출하지 않는다.
Tailscale 또는 사설망에서만 접근 가능해야 한다.

### Register Node

```text
POST /internal/judge/nodes/register
```

Request:

```json
{
  "node_name": "judge-1",
  "node_secret": "plain-secret-from-env",
  "total_slots": 10,
  "agent_version": "0.1.0",
  "tailscale_node_identity": "ts-node-id",
  "tailscale_ip": "100.64.0.10"
}
```

Response:

```json
{
  "judge_node_id": "uuid",
  "heartbeat_interval_seconds": 5,
  "lease_extend_interval_seconds": 30
}
```

### Heartbeat

```text
POST /internal/judge/nodes/{judge_node_id}/heartbeat
```

Request:

```json
{
  "node_secret": "plain-secret-from-env",
  "total_slots": 10,
  "free_slots": 7,
  "running_job_count": 3,
  "running_attempt_ids": ["uuid"],
  "agent_version": "0.1.0"
}
```

Response:

```json
{
  "node_status": "active",
  "schedulable": true,
  "server_time": "2026-05-02T12:00:00Z"
}
```

### Fetch Assigned Jobs

```text
POST /internal/judge/nodes/{judge_node_id}/assignments:claim
```

Request:

```json
{
  "node_secret": "plain-secret-from-env",
  "max_count": 3
}
```

Response:

```json
{
  "jobs": [
    {
      "judge_job_id": "uuid",
      "judge_job_attempt_id": "uuid",
      "lease_token": "plain-lease-token",
      "submission_id": "uuid",
      "contest_id": "uuid",
      "problem_id": "uuid",
      "language": "cpp17",
      "source_code_storage_key": "submissions/...",
      "testcase_set_id": "uuid",
      "time_limit_ms": 1000,
      "memory_limit_mb": 512
    }
  ]
}
```

claim 호출은 이미 dispatcher가 해당 node에 배정한 job만 반환한다.
반환된 job은 `judge_jobs.job_status = 'running'`, `judge_job_attempts.attempt_status = 'running'`으로 전이한다.

### Report Result

```text
POST /internal/judge/jobs/{judge_job_id}/result
```

Request:

```json
{
  "node_secret": "plain-secret-from-env",
  "judge_job_attempt_id": "uuid",
  "lease_token": "plain-lease-token",
  "final_status": "accepted",
  "awarded_score": 100,
  "compile_message": null,
  "runtime_summary": {
    "max_time_ms": 124,
    "max_memory_kb": 20480
  },
  "testcase_results": [
    {
      "testcase_id": "uuid",
      "status": "accepted",
      "time_ms": 12,
      "memory_kb": 20480,
      "output_truncated": false
    }
  ],
  "finished_at": "2026-05-02T12:00:10Z"
}
```

Response:

```json
{
  "accepted": true,
  "submission_status": "accepted",
  "job_status": "succeeded"
}
```

Idempotency:

- 서버는 `judge_job_attempt_id` 기준으로 결과 반영 여부를 확인한다.
- 이미 같은 attempt 결과가 반영된 경우 같은 response를 반환할 수 있다.
- lease token이 다르거나 만료된 경우 `409 lease_conflict`를 반환한다.

### Report Agent Failure

```text
POST /internal/judge/jobs/{judge_job_id}/failure
```

Request:

```json
{
  "node_secret": "plain-secret-from-env",
  "judge_job_attempt_id": "uuid",
  "lease_token": "plain-lease-token",
  "error_code": "sandbox_start_failed",
  "error_message": "failed to create sandbox container",
  "retryable": true
}
```

retryable이면 dispatcher가 max retry count 안에서 다시 `pending`으로 되돌린다.
retry 불가능하거나 max retry를 초과하면 submission은 `system_error`로 확정한다.

## Dispatcher Algorithm

반복 주기 기본값은 1초다.

```text
1. lost node 감지
2. expired lease job 회수
3. schedulable active node의 free_slots 계산
4. pending job을 FIFO로 조회
5. 각 job에 node를 선택
6. judge_job_attempt 생성
7. judge_jobs를 assigned 상태로 갱신하고 lease_token_hash 저장
```

DB 갱신은 `SELECT ... FOR UPDATE SKIP LOCKED`를 사용한다.
dispatcher는 여러 개를 띄울 수 있게 설계할 수 있지만, MVP에서는 단일 dispatcher만 실행한다.

## 테스트 기준

채점 프로토콜은 구현 중 아래 테스트를 기능별로 추가한다.

- job 생성 transaction 테스트
- dispatcher FIFO 배정 테스트
- node slot 부족 시 미배정 테스트
- heartbeat timeout과 lease expiry 테스트
- claim API가 자기 node의 assigned job만 반환하는 테스트
- result report idempotency 테스트
- lease token 불일치 실패 테스트
- retry 초과 후 `system_error` 확정 테스트

## Node Selection

MVP 기본값:

1. `node_status = active`
2. `schedulable = true`
3. `last_heartbeat_at >= now() - node_lost_timeout`
4. `free_slots > 0`
5. `running_job_count`가 가장 적은 node 우선
6. 동률이면 `last_heartbeat_at`이 최신인 node 우선

관리자가 node를 disable하면 신규 job은 배정하지 않는다.
`draining` node는 기존 running job만 마무리하고 신규 배정하지 않는다.

## Sandbox 결과 코드

agent는 sandbox 내부 결과를 아래 final status로 변환한다.

| 상황 | final_status |
| --- | --- |
| 모든 테스트 통과 | `accepted` |
| 출력 형식 오류 | `presentation_error` |
| 출력 불일치 | `wrong_answer` |
| wall/cpu time 초과 | `time_limit_exceeded` |
| memory limit 초과 | `memory_limit_exceeded` |
| output limit 초과 | `output_limit_exceeded` |
| non-zero exit 또는 signal | `runtime_error` |
| compile 실패 | `compile_error` |
| sandbox/infra 오류 | `system_error` |

테스트케이스 진행 중 final failure가 나오면 이후 테스트케이스는 실행하지 않는다.

## 보안

- node secret은 원문 저장하지 않고 hash로 저장한다.
- lease token도 원문 저장하지 않고 hash로 저장한다.
- internal API는 public reverse proxy에 등록하지 않는다.
- judge-agent가 접근하는 source/testcase URL은 내부망 전용 또는 짧은 TTL signed URL만 허용한다.
- 제출 sandbox 컨테이너는 network disabled, read-only root filesystem, 제한된 temp mount를 기본으로 한다.
