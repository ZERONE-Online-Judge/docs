# API 문서

현재 구현된 FastAPI 백엔드 기준 API 문서다.

Base URL:

```text
http://127.0.0.1:8012/api
```

운영 배포에서는 Nginx 뒤의 `/api` prefix를 유지한다.

## 공통 규칙

### 인증

| 대상 | 방식 | Header |
| --- | --- | --- |
| 공개 API | 없음 | 없음 |
| 일반 로그인 API | signed JWT general access token | `Authorization: Bearer {general_access_token}` |
| 참가자 API | signed JWT participant access token | `Authorization: Bearer {participant_access_token}` |
| 운영자 API | signed JWT staff access token | `Authorization: Bearer {staff_access_token}` |
| 서비스 관리자 API | signed JWT service master staff access token | `Authorization: Bearer {staff_access_token}` |
| 내부 채점기 API | body의 `node_secret` | 내부망에서만 노출 예정 |

access/refresh token은 `HS256` 서명 JWT로 발급하고, 서버 DB에는 token hash와 만료/폐기 상태를 저장한다. 따라서 JWT 검증과 DB 세션 검증을 모두 통과해야 유효하다.

### 정상 응답 포맷

단건:

```json
{
  "data": {},
  "request_id": "req_example"
}
```

목록:

```json
{
  "data": [],
  "page": {
    "limit": 20,
    "next_cursor": null
  },
  "request_id": "req_example"
}
```

### 에러 응답 포맷

```json
{
  "error": {
    "code": "not_found",
    "message": "Resource not found.",
    "request_id": "req_example",
    "details": {}
  }
}
```

공통 에러 코드:

| HTTP | code | 의미 |
| --- | --- | --- |
| 401 | `authentication_required` | Bearer token 없음/만료/무효 |
| 401 | `invalid_credentials` | 로그인 정보, OTP, refresh token 무효 |
| 403 | `permission_denied` | 서비스 관리자 권한 필요 |
| 403 | `scope_denied` | 해당 대회 운영 권한 없음 |
| 403 | `node_secret_invalid` | 채점 노드 secret 불일치 |
| 404 | `not_found` | 리소스 없음 또는 공개하면 안 되는 리소스 |
| 409 | `invalid_state_transition` | 현재 상태에서 요청 불가 |
| 409 | `lease_conflict` | judge job lease token 불일치 |
| 429 | `otp_request_rate_limited` | OTP 재요청 제한 시간 이내 |
| 422 | `validation_error` | 요청 body 값 검증 실패 |

FastAPI 기본 validation error는 아직 공통 `error` wrapper로 정규화되지 않을 수 있다. 구현 시 API 테스트를 같이 추가해 정규화한다.

### 주요 enum

```text
contest.status = draft | scheduled | open | running | ended | finalized | archived
submission.status = waiting | preparing | judging | accepted | wrong_answer | compile_error | runtime_error | system_error
judge_job.status = pending | assigned | running | succeeded | failed
language = c99 | cpp17 | python313 | java8
```

## Public API

| Method | Path | Query/Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| GET | `/health` | 없음 | 없음 | `{ "status": "ok" }` | 없음 |
| GET | `/public/home` | 없음 | 없음 | `hero`, `active_contest_count`, `emergency_notice` | 없음 |
| GET | `/public/contests` | 없음 | 없음 | 공개 대회 목록 | 없음 |
| GET | `/public/contests/{contest_id}` | `contest_id` | 없음 | `contest`, `divisions[]` | `404 not_found` |
| GET | `/public/service-notices` | 없음 | 없음 | 공지 목록 | 없음 |
| GET | `/public/service-notices/{notice_id}` | `notice_id` | 없음 | 서비스 공지 | `404 not_found` |
| GET | `/public/judge-status` | 없음 | 없음 | 노드 수, 실행 job 수, 큐 깊이 | 없음 |
| GET | `/public/rules` | 없음 | 없음 | 규정 섹션 목록 | 없음 |

예시:

```http
GET /api/public/contests
```

```json
{
  "data": [
    {
      "contest_id": "uuid",
      "title": "Zerone Spring Invitational",
      "status": "running",
      "start_at": "2026-05-04T06:44:11Z",
      "end_at": "2026-05-04T10:44:11Z",
      "freeze_at": "2026-05-04T09:44:11Z"
    }
  ],
  "page": { "limit": 20, "next_cursor": null },
  "request_id": "req_example"
}
```

## Staff Auth API

| Method | Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- |
| POST | `/auth/staff/login` | `email`, `password` | `access_token`, `refresh_token`, staff 정보, redirect | `401 invalid_credentials` |
| GET | `/auth/staff/me` | 없음 | staff account | `401 authentication_required` |
| POST | `/auth/staff/logout` | `refresh_token?` | `{ "revoked": true/false }` | 없음 |
| POST | `/auth/staff/refresh` | `refresh_token` | 새 `access_token`, `refresh_token` | `401 invalid_credentials` |

요청:

```json
{
  "email": "admin@example.com",
  "password": "configured-password"
}
```

정상 응답:

```json
{
  "data": {
    "access_token": "token",
    "refresh_token": "token",
    "token_type": "bearer",
    "default_redirect": "/admin"
  },
  "request_id": "req_example"
}
```

운영 최초 계정은 `BOOTSTRAP_SERVICE_MASTER_EMAIL`, `BOOTSTRAP_SERVICE_MASTER_PASSWORD`, `BOOTSTRAP_SERVICE_MASTER_NAME` 환경변수로 생성한다.
개발 fixture 계정은 `ENABLE_DEMO_SEED=true`인 테스트/로컬 환경에서만 생성한다.

## General Login API

일반 로그인은 참가자와 대회 운영자가 공통으로 사용하는 이메일 OTP 로그인이다. 서비스 관리자 비밀번호 로그인과 세션을 분리한다.

| Method | Path | Auth | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| POST | `/auth/general/otp/request` | 없음 | `email` | `{ sent, delivery, cooldown_seconds }`, 개발 우회 시 `demo_otp` 포함 | `404 not_found`, `429 otp_request_rate_limited` |
| POST | `/auth/general/otp/verify` | 없음 | `email`, `otp_code` | general `access_token`, `refresh_token`, `account`, `participant_contests`, `operator_contests`, `operator_session?` | `401 invalid_credentials` |
| GET | `/auth/general/me` | general | 없음 | `account`, `participant_contests`, `operator_contests`, `operator_session?` | `401 authentication_required` |
| POST | `/auth/general/refresh` | 없음 | `refresh_token` | 새 `access_token`, `refresh_token`, 로그인 권한 목록 | `401 invalid_credentials` |
| POST | `/auth/general/logout` | general | `refresh_token?` | `{ "revoked": true/false }` | 없음 |
| POST | `/auth/general/contests/{contest_id}/participant-session` | general | 없음 | 참가자 `access_token`, `team`, `member`, `division` | `401 authentication_required`, `403 scope_denied` |

정상 응답 예:

```json
{
  "data": {
    "access_token": "general-token",
    "refresh_token": "general-refresh-token",
    "account": { "email": "test1@zoj.com", "display_name": "Test User" },
    "participant_contests": [
      {
        "contest": { "contest_id": "uuid", "title": "Zerone Spring Invitational" },
        "team": { "participant_team_id": "uuid", "team_name": "Team Async" },
        "member": { "name": "Test User", "email": "test1@zoj.com" },
        "division": { "division_id": "uuid", "name": "COSS" }
      }
    ],
    "operator_contests": [
      {
        "contest": { "contest_id": "uuid", "title": "Zerone Spring Invitational" },
        "scopes": ["contest.*"]
      }
    ],
    "operator_session": {
      "access_token": "staff-token",
      "refresh_token": "staff-refresh-token",
      "staff": { "email": "operator@zoj.com", "display_name": "Operator", "is_service_master": false },
      "default_redirect": "/operator"
    }
  },
  "request_id": "req_example"
}
```

참가자가 대회에 입장할 때는 general token으로 `/auth/general/contests/{contest_id}/participant-session`을 호출해 기존 참가자 API용 participant token을 발급받는다. 대회 운영자는 `operator_session`의 staff token으로 기존 `/operator/*` API를 호출한다.

## Participant API

### 로그인/세션

| Method | Path | Auth | Body/Query | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| POST | `/contests/{contest_id}/participant-login/otp/request` | 없음 | body `email` | `{ sent, delivery, cooldown_seconds }`, 개발 우회 시 `demo_otp` 포함 | `404 not_found`, `429 otp_request_rate_limited` |
| POST | `/contests/{contest_id}/participant-login/otp/verify` | 없음 | body `email`, `otp_code` | `access_token`, `team`, `member`, `division`, `workspace_path` | `401 invalid_credentials` |
| GET | `/contests/{contest_id}/participant-session/me` | participant | 없음 | `team`, `member`, `division` | `401 authentication_required` |

OTP 요청:

```json
{
  "email": "test2@zoj.com"
}
```

같은 이메일 기준 OTP 재요청은 기본 10초 쿨다운이 있다. 제한 시간 내 재요청 시:

```json
{
  "error": {
    "code": "otp_request_rate_limited",
    "message": "Please wait 7 seconds before requesting another verification code.",
    "request_id": "req_example",
    "details": {
      "retry_after_seconds": 7
    }
  }
}
```

OTP 검증:

```json
{
  "email": "test2@zoj.com",
  "otp_code": ""
}
```

### 워크스페이스/문제

| Method | Path | Auth | Body/Query | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| GET | `/contests/{contest_id}/workspace` | participant 또는 종료 후 문제 공개 | 없음 | `contest`, `division`, `divisions[]`, `problems[]`, `emergency_notice` | `404 not_found` |
| GET | `/contests/{contest_id}/divisions/{division_id}/workspace` | participant 또는 종료 후 문제 공개 | 없음 | `contest`, `division`, `problems[]`, `emergency_notice` | `404 not_found` |
| GET | `/contests/{contest_id}/problems` | participant 또는 종료 후 문제 공개 | 없음 | 참가자 division 또는 공개 기본 division 문제 목록 | `404 not_found` |
| GET | `/contests/{contest_id}/divisions/{division_id}/problems` | participant 또는 종료 후 문제 공개 | 없음 | division 문제 목록 | `404 not_found` |
| GET | `/contests/{contest_id}/problems/{problem_id}` | participant 또는 종료 후 문제 공개 | 없음 | problem | `404 not_found` |

비로그인 사용자는 대회 전과 대회 중에 문제 API를 볼 수 없다. 참가자는 본인 팀의 참가 유형 문제만 조회할 수 있으며, 다른 division 문제는 `404 not_found`로 숨긴다. 종료 후 공개는 `problem_public_after_end`가 켜진 경우에만 허용한다.

### 제출/채점 상태

| Method | Path | Auth | Body/Query | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| POST | `/contests/{contest_id}/problems/{problem_id}/submissions` | participant | `language`, `source_code` | submission | `401`, `404`, `409`, `422` |
| GET | `/contests/{contest_id}/submissions` | participant 또는 종료 후 제출 현황 공개 | 없음 | 자기 팀 제출 목록 또는 공개 제출 목록 | `404 not_found` |
| GET | `/contests/{contest_id}/submissions/{submission_id}` | participant | 없음 | 자기 팀 제출 상세 | `401`, `404` |
| GET | `/contests/{contest_id}/submissions/{submission_id}/status:wait` | participant | 없음 | long polling 후 submission | `401`, `404` |

비로그인 사용자는 대회 전과 대회 중에 제출 현황을 볼 수 없다. 종료 후 공개는 `submission_public_after_end`가 켜진 경우에만 허용하며, 공개 제출 목록에는 소스코드를 포함하지 않는다.

제출 요청:

```json
{
  "language": "python313",
  "source_code": "print(42)"
}
```

제출 정상 응답:

```json
{
  "data": {
    "submission_id": "uuid",
    "contest_id": "uuid",
    "division_id": "uuid",
    "problem_id": "uuid",
    "participant_team_id": "uuid",
    "team_member_id": "uuid",
    "language": "python313",
    "source_code": "print(42)",
    "status": "waiting",
    "awarded_score": null,
    "compile_message": null
  },
  "request_id": "req_example"
}
```

참고:

- 대회 상태가 `running`이 아니거나 `end_at` 이후면 `409 invalid_state_transition`.
- 참가자 팀의 division과 다른 문제에 제출하면 `404 not_found`.
- 지원 언어는 `c99`, `cpp17`, `python313`, `java8`.
- 수동 재채점 API는 없다.

### 스코어보드

| Method | Path | Auth | Body/Query | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- | --- |
| GET | `/contests/{contest_id}/scoreboard` | participant 또는 종료 후 스코어보드 공개 | 없음 | 참가자 division 또는 공개 기본 division 스코어보드 | `404 not_found` |
| GET | `/contests/{contest_id}/divisions/{division_id}/scoreboard` | participant 또는 종료 후 스코어보드 공개 | 없음 | division 공개 스코어보드 | `404 not_found` |

비로그인 사용자는 대회 전과 대회 중에 스코어보드를 볼 수 없다. 참가자는 본인 팀의 참가 유형 스코어보드만 조회할 수 있으며, 다른 division 스코어보드는 `404 not_found`로 숨긴다. 종료 후 공개는 `scoreboard_public_after_end`가 켜진 경우에만 허용한다.

스코어보드 row:

```json
{
  "rank": 1,
  "team_id": "uuid",
  "team_name": "Team Async",
  "division_id": "uuid",
  "division": "Advanced",
  "solved": 1,
  "score": 100,
  "submission_count": 3,
  "last_improved_at": "2026-05-04T07:44:11Z",
  "problem_scores": [
    {
      "problem_id": "uuid",
      "problem_code": "A",
      "score": 100,
      "max_score": 100,
      "best_submission_id": "uuid",
      "best_submitted_at": "2026-05-04T07:44:11Z",
      "best_status": "accepted"
    }
  ]
}
```

점수 정책:

- 팀-문제별 최고 점수만 반영.
- 같은 점수면 기존 제출 유지.
- `compile_error`는 점수 반영에서 제외.
- 공개 스코어보드는 freeze 중이면 freeze 시점 제출까지만 반영.

## Service Admin API

모든 API는 service master token이 필요하다.
운영자 token은 `403 permission_denied`.

| Method | Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- |
| GET | `/admin/dashboard` | 없음 | contest/job/mail/node count | `401`, `403` |
| GET | `/admin/contests` | 없음 | 전체 대회 목록 | `401`, `403` |
| POST | `/admin/contests` | `organization_name`, `status`, `start_at`, `operator_email?`, `title?`, `overview?` | contest, operator assignment mail queued when `operator_email` exists | `401`, `403`, `422` |
| POST | `/admin/contests/{contest_id}/divisions` | `code`, `name`, `description?`, `display_order?` | division | `401`, `403`, `404`, `422` |
| POST | `/admin/contests/{contest_id}/operators` | `email`, `display_name?` | staff account with contest scope, assignment mail queued | `401`, `403`, `404`, `422` |
| GET | `/admin/service-managers` | 없음 | staff account 목록 | `401`, `403` |
| GET | `/admin/judge/dashboard` | 없음 | judge nodes, queue | `401`, `403` |
| GET | `/admin/mail-queue` | 없음 | mail queue 목록 | `401`, `403` |

대회 생성 요청:

```json
{
  "organization_name": "Zerone",
  "status": "open",
  "start_at": "2026-05-20T00:00:00+09:00",
  "operator_email": "operator@example.com"
}
```

제공하지 않는 API:

```text
POST /api/admin/contests/{contest_id}/submissions/{submission_id}/rejudge
```

서비스 마스터도 수동 재채점은 할 수 없다. 현재 이 주소는 `404 Not Found`가 정상이다.

## Contest Operator API

모든 API는 해당 `contest_id` scope가 있는 staff token 또는 service master token이 필요하다.

### 운영 대시보드/참가팀

| Method | Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- |
| GET | `/operator/contests` | 없음 | 로그인한 운영자에게 할당된 대회 목록, 서비스 마스터는 전체 대회 | `401` |
| GET | `/operator/contests/{contest_id}/dashboard` | 없음 | contest, divisions, count, pending_jobs | `401`, `403`, `404` |
| GET | `/operator/contests/{contest_id}/divisions` | 없음 | division 목록 | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/divisions` | `name`, `description?` | division | `401`, `403`, `404`, `422` |
| PATCH | `/operator/contests/{contest_id}/divisions/{division_id}` | `name?`, `description?` | division | `401`, `403`, `404`, `422` |
| PATCH | `/operator/contests/{contest_id}/settings` | contest settings partial payload | contest, settings update mail queued to contest operators | `401`, `403`, `404`, `422` |
| GET | `/operator/contests/{contest_id}/participants` | 없음 | 참가팀 목록 | `401`, `403` |
| POST | `/operator/contests/{contest_id}/participants` | team payload | participant team | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/participants:bulk-create` | `teams[]` | `created[]`, `errors[]` | `401`, `403`, `422` |
| PATCH | `/operator/contests/{contest_id}/participants/{participant_team_id}` | participant team partial payload | participant team | `401`, `403`, `404`, `422` |
| POST | `/operator/contests/{contest_id}/participants/{participant_team_id}/members` | member payload | team member | `401`, `403`, `404`, `422` |
| PATCH | `/operator/contests/{contest_id}/participants/{participant_team_id}/members/{team_member_id}` | member partial payload | team member | `401`, `403`, `404`, `422` |
| POST | `/operator/contests/{contest_id}/participants/{participant_team_id}/members/{team_member_id}/sessions:revoke` | 없음 | team member | `401`, `403`, `404` |

대회 설정 수정 요청:

```json
{
  "status": "ended",
  "start_at": "2026-05-05T13:00:00+09:00",
  "end_at": "2026-05-05T17:00:00+09:00",
  "freeze_at": "2026-05-05T16:00:00+09:00",
  "problem_public_after_end": true,
  "scoreboard_public_after_end": true,
  "submission_public_after_end": false,
  "emergency_notice": "Final review in progress"
}
```

검증:

- `start_at`은 `end_at`보다 앞서야 한다.
- `freeze_at`은 `start_at`과 `end_at` 사이여야 한다.
- 종료 후 공개 설정은 문제, 스코어보드, 제출 현황을 각각 따로 저장한다.

참가 유형 생성/수정 요청:

```json
{
  "name": "Advanced",
  "description": "심화 유형"
}
```

참가 유형 정책:

- 운영 화면에서는 이름과 설명만 관리한다.
- `name`은 같은 대회 안에서 중복될 수 없다.
- 목록과 선택지는 이름순으로 정렬한다.

참가팀 등록 요청:

```json
{
  "team_name": "Team Async",
  "division_id": "uuid",
  "leader": {
    "name": "Test Two",
    "email": "test2@zoj.com"
  },
  "members": [
    {
      "name": "Park Member",
      "email": "test2-member@zoj.com"
    }
  ]
}
```

참가팀 일괄 등록 요청:

```json
{
  "teams": [
    {
      "team_name": "Team Async",
      "division_id": "uuid",
      "leader": {
        "name": "Test Two",
        "email": "test2@zoj.com"
      },
      "members": [
        {
          "name": "Park Member",
          "email": "test2-member@zoj.com"
        }
      ]
    }
  ]
}
```

참가팀 일괄 등록 정상 응답:

```json
{
  "created": [
    {
      "participant_team_id": "uuid",
      "team_name": "Team Async",
      "division_id": "uuid",
      "members": []
    }
  ],
  "errors": [
    {
      "row": 2,
      "team_name": "Team Duplicate",
      "message": "participant email already registered"
    }
  ]
}
```

프론트 일괄 등록 파일 형식:

- Excel에서 CSV 또는 탭 구분 텍스트로 저장한 파일을 입력받는다.
- 지원 헤더: `team_name, division, leader_name, leader_email, member1_name, member1_email...`
- 한국어 헤더도 지원: `팀명, 참가유형, 팀장이름, 팀장메일, 팀원1이름, 팀원1메일...`
- `division`은 참가 유형 이름, 코드, ID 중 하나로 매칭한다.

참가팀 수정 요청:

```json
{
  "team_name": "Team Async Updated",
  "division_id": "uuid",
  "status": "disabled"
}
```

팀원 추가/수정 요청:

```json
{
  "name": "Park Member",
  "email": "test2-member@zoj.com",
  "role": "member"
}
```

팀원 정책:

- 이메일은 대회 안에서 중복될 수 없다.
- 팀원 삭제 API는 제공하지 않는다.
- `sessions:revoke`는 해당 이메일 참가자의 활성 세션을 종료하고 `active_sessions`를 0으로 만든다.
- 팀장/팀원 역할 변경은 운영 정책이 더 필요하므로 현재는 추가/이름/이메일/세션 종료만 제공한다.

정책:

- 참가팀 삭제 API는 제공하지 않는다.
- 상태는 `invited`, `active`, `disabled`, `disqualified` 중 하나다.
- 팀 단위 참가 유형은 하나만 유지한다.
- 팀원 이메일 편집은 유니크 제약과 세션 정책 때문에 별도 멤버 관리 API에서 다룬다.

정책:

- 참가팀은 운영자가 등록한다.
- 팀은 하나의 `division_id`만 가진다.
- 참가 유형이 2개 이상인 대회에서는 `division_id`가 필수다.
- 한 참가팀은 정확히 하나의 `division_id`만 가질 수 있고 중복 배정은 불가하다.
- `division_id`가 다르면 문제, 제출, 스코어보드, 채점 이력은 사실상 다른 대회처럼 분리된다.
- 세션과 OTP는 팀 단위가 아니라 참가자 이메일별로 관리한다.

### 운영 제출/채점 이력/스코어보드

| Method | Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- |
| GET | `/operator/contests/{contest_id}/submissions` | 없음 | 전체 제출 목록 | `401`, `403` |
| GET | `/operator/contests/{contest_id}/judge-history` | 없음 | judge job 목록 | `401`, `403` |
| GET | `/operator/contests/{contest_id}/scoreboard/internal` | 없음 | 전체 division 최신 스코어보드 | `401`, `403`, `404` |
| GET | `/operator/contests/{contest_id}/divisions/{division_id}/scoreboard/internal` | 없음 | division 최신 스코어보드 | `401`, `403`, `404` |

운영자 내부 스코어보드는 freeze 여부와 관계없이 최신 점수를 본다.
재채점 버튼/API는 제공하지 않는다.

### 문제/파일/테스트케이스

| Method | Path | Body | 정상 응답 data | 주요 에러 |
| --- | --- | --- | --- | --- |
| GET | `/operator/contests/{contest_id}/problems` | 없음 | 문제 목록 | `401`, `403` |
| POST | `/operator/contests/{contest_id}/problems` | problem payload | problem | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/problems/import-polygon` | multipart zip payload | problem, assets, script, package build result | `401`, `403`, `422 polygon_import_failed` |
| PATCH | `/operator/contests/{contest_id}/problems/{problem_id}` | partial problem payload | problem | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/storage/presign-upload` | `category`, `filename`, `content_type?` | upload URL | `401`, `403`, `404` |
| GET | `/operator/contests/{contest_id}/problems/{problem_id}/assets` | 없음 | asset 목록 | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/problems/{problem_id}/assets` | asset payload | asset | `401`, `403`, `404` |
| GET | `/operator/contests/{contest_id}/problems/{problem_id}/testcase-sets` | 없음 | testcase set 목록 | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/problems/{problem_id}/testcase-sets` | `is_active` | testcase set | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/problems/{problem_id}/testcase-sets/{testcase_set_id}/testcases` | testcase payload | testcase | `401`, `403`, `404` |
| POST | `/operator/contests/{contest_id}/problems/{problem_id}/verified-testcase-sets` | `.in/.out` case 목록 | verified active testcase set | `401`, `403`, `404`, `422 testcase_verification_failed` |

문제 생성 요청:

```json
{
  "division_id": "uuid",
  "problem_code": "A",
  "title": "A+B Reloaded",
  "statement": "문제 설명",
  "time_limit_ms": 1000,
  "memory_limit_mb": 512,
  "display_order": 1,
  "max_score": 100
}
```

문제 수정 요청:

```json
{
  "title": "New Title",
  "statement": "Updated statement",
  "time_limit_ms": 2000,
  "memory_limit_mb": 512,
  "display_order": 2,
  "max_score": 100
}
```

모든 필드는 optional이다.

Polygon zip 가져오기 요청:

`multipart/form-data`

| Field | Type | Required | 설명 |
| --- | --- | --- | --- |
| `file` | file | yes | Polygon export zip. 내부에 `problem.xml` 필요 |
| `division_id` | string | yes | 문제를 만들 참가 유형 |
| `display_order` | number | no | 표시 순서. 생략 시 `1` |
| `max_score` | number | no | 기본 `100` |
| `build_tests` | boolean | no | 기본 `true`. 가져오기 직후 package build 실행 |

정상 응답:

```json
{
  "problem": { "problem_id": "uuid", "problem_code": "A", "title": "A+B" },
  "assets": [{ "original_filename": "generator.cpp", "storage_key": "contests/.../package-files/generator/generator.cpp" }],
  "script_text": "manual 01\ngenerator random 100 1\n",
  "package_build": {
    "status": "completed",
    "generated_count": 2,
    "testcase_set": { "testcase_set_id": "uuid", "version": 1, "is_active": true }
  },
  "summary": {
    "asset_count": 6,
    "example_count": 2,
    "script_line_count": 76
  }
}
```

`package_build.status`가 `failed`면 문제와 패키지 파일은 등록된 상태이며, `package_build.error`에 컴파일/validator/checker 실패 이유가 들어간다.

presigned upload 요청:

```json
{
  "category": "testcases",
  "filename": "input-1.txt",
  "content_type": "text/plain"
}
```

검증 기반 testcase set 생성 요청:

```json
{
  "cases": [
    {
      "display_order": 1,
      "input_storage_key": "contests/{contest_id}/problems/{problem_id}/testcases/001.in",
      "output_storage_key": "contests/{contest_id}/problems/{problem_id}/testcases/001.out",
      "input_sha256": "64-char-hex",
      "output_sha256": "64-char-hex"
    }
  ]
}
```

검증 파일 역할:

- `package-resource`: `testlib.h`
- `validator`: `validator.cpp`
- `checker`: `checker.cpp`

검증 순서:

1. `testlib.h`, `validator.cpp`, `checker.cpp`를 컴파일 준비한다.
2. 각 `.in` 파일을 validator stdin으로 넣어 검증한다.
3. 각 `.out` 파일을 checker의 official answer와 participant output 양쪽에 넣어 self-check한다.
4. 모든 케이스가 통과하면 새 active testcase set을 만들고 `.in/.out` storage key를 등록한다.

정상 응답:

```json
{
  "data": {
    "method": "PUT",
    "storage_key": "contests/{contest_id}/testcases/input-1.txt",
    "upload_url": "http://...",
    "content_type": "text/plain"
  },
  "request_id": "req_example"
}
```

asset 등록 요청:

```json
{
  "original_filename": "statement.pdf",
  "storage_key": "contests/{contest_id}/assets/statement.pdf",
  "mime_type": "application/pdf",
  "file_size": 1024,
  "sha256": "64-char-hex"
}
```

testcase set 생성 요청:

```json
{
  "is_active": true
}
```

testcase 등록 요청:

```json
{
  "display_order": 1,
  "input_storage_key": "contests/{contest_id}/testcases/input-1.txt",
  "output_storage_key": "contests/{contest_id}/testcases/output-1.txt",
  "input_sha256": "64-char-hex",
  "output_sha256": "64-char-hex",
  "time_limit_ms_override": null,
  "memory_limit_mb_override": null
}
```

## Internal Judge API

내부 채점기 API는 공개 인터넷에 노출하지 않는다.
Backend VM과 Judge VM 사이 내부 IP/Tailscale/사설망에서만 호출한다.

### Node register

```http
POST /api/internal/judge/nodes/register
```

요청:

```json
{
  "node_name": "judge-vm-1",
  "node_secret": "node-secret",
  "total_slots": 10,
  "agent_version": "0.1.0"
}
```

정상 응답:

```json
{
  "data": {
    "judge_node_id": "uuid",
    "heartbeat_interval_seconds": 5
  },
  "request_id": "req_example"
}
```

에러:

| HTTP | code | 조건 |
| --- | --- | --- |
| 403 | `node_secret_invalid` | 같은 node name으로 등록된 기존 secret과 불일치 |

### Heartbeat

```http
POST /api/internal/judge/nodes/{node_id}/heartbeat
```

요청:

```json
{
  "node_secret": "node-secret",
  "total_slots": 10,
  "free_slots": 7,
  "running_job_count": 3
}
```

정상 응답 data는 judge node다.

에러:

| HTTP | code | 조건 |
| --- | --- | --- |
| 403 | `node_secret_invalid` | secret 불일치 |
| 404 | `not_found` | node 없음 |

### Claim

```http
POST /api/internal/judge/nodes/{node_id}/assignments:claim
```

요청:

```json
{
  "node_secret": "node-secret",
  "max_count": 10
}
```

정상 응답:

```json
{
  "data": {
    "jobs": [
      {
        "judge_job_id": "uuid",
        "submission_id": "uuid",
        "contest_id": "uuid",
        "division_id": "uuid",
        "status": "running",
        "queue_position": 1,
        "assigned_node_id": "uuid",
        "lease_token": "uuid",
        "submission": {},
        "testcase_set": {},
        "testcases": [
          {
            "testcase_id": "uuid",
            "input_url": "http://...",
            "output_url": "http://..."
          }
        ]
      }
    ]
  },
  "request_id": "req_example"
}
```

에러:

| HTTP | code | 조건 |
| --- | --- | --- |
| 403 | `node_secret_invalid` | secret 불일치 |
| 404 | `not_found` | node 없음 |

### Result report

```http
POST /api/internal/judge/jobs/{job_id}/result
```

요청:

```json
{
  "node_secret": "node-secret",
  "lease_token": "lease-token",
  "final_status": "accepted",
  "awarded_score": 100,
  "compile_message": null
}
```

정상 응답:

```json
{
  "data": {
    "accepted": true,
    "submission": {},
    "job": {}
  },
  "request_id": "req_example"
}
```

에러:

| HTTP | code | 조건 |
| --- | --- | --- |
| 403 | `node_secret_invalid` | secret 불일치 |
| 404 | `not_found` | job 또는 submission 없음 |
| 409 | `lease_conflict` | lease token 불일치 |

## API 테스트 작성 기준

기능을 추가하거나 API 계약을 바꿀 때는 같은 PR/작업 단위에서 테스트를 추가한다.

필수 테스트:

- 정상 응답 포맷: `data`, `request_id`
- 목록 응답 포맷: `data`, `page`, `request_id`
- 인증 실패: `401 authentication_required`
- 권한 실패: `403 permission_denied` 또는 `403 scope_denied`
- 리소스 없음/비공개: `404 not_found`
- 상태 충돌: `409 invalid_state_transition`
- 내부 judge secret 실패: `403 node_secret_invalid`
- 내부 judge lease 실패: `409 lease_conflict`

현재 검증 명령:

```bash
cd backend
.venv/bin/python -m pytest tests
```

채점 에이전트:

```bash
PYTHONPATH=judge_agent backend/.venv/bin/python -m pytest judge_agent/tests
```
| POST | `/auth/staff/otp/request` | `email` | `cooldown_seconds` 포함 staff OTP 발송 | `404` 미등록 staff email, `429 otp_request_rate_limited` |
| POST | `/auth/staff/otp/verify` | `email`, `otp_code` | staff session | `401` 잘못된 code/만료 |
