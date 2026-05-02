# 백엔드 API 오류 응답 기준

이 문서는 FastAPI 백엔드의 공통 실패 응답 형식을 정의한다.
각 API 문서는 성공 응답에 집중하고, 실패 응답은 이 문서의 code를 재사용한다.

## 공통 오류 응답

```json
{
  "error": {
    "code": "permission_denied",
    "message": "You do not have permission to perform this action.",
    "request_id": "req_01JABCDEF0000000000000000",
    "details": {
      "required_permission": "contest.participant.update"
    }
  }
}
```

원칙:

- `code`는 클라이언트 분기용 안정 문자열이다.
- `message`는 운영자 화면 표시용 기본 문구다. 계정 존재 여부, 토큰 원문, 내부 경로는 노출하지 않는다.
- `request_id`는 모든 응답에 포함하고 서버 로그와 연결한다.
- `details`는 선택값이며 validation field error처럼 안전한 정보만 담는다.

## HTTP Status 매핑

| status | code | 사용 조건 |
| --- | --- | --- |
| 400 | `bad_request` | 요청 의미가 잘못됐지만 field validation으로 표현하기 어려움 |
| 401 | `authentication_required` | access token 없음 |
| 401 | `invalid_token` | access token 형식 오류, 서명 오류, 만료 |
| 401 | `invalid_credentials` | 로그인 실패. 계정 존재 여부는 노출하지 않음 |
| 403 | `permission_denied` | 인증은 됐지만 권한 부족 |
| 403 | `scope_denied` | 권한은 있으나 해당 contest scope에는 적용 불가 |
| 404 | `not_found` | 리소스 없음 또는 존재 여부를 숨겨야 함 |
| 409 | `conflict` | unique constraint, 이미 처리된 상태, 중복 요청 |
| 409 | `invalid_state_transition` | 현재 상태에서 허용되지 않는 전이 |
| 409 | `role_conflict` | 같은 대회 내 운영자/매니저와 참가팀 역할 충돌 |
| 409 | `lease_conflict` | judge job lease token 불일치 또는 만료 |
| 410 | `gone` | 초대/OTP/reset token 만료 또는 이미 사용됨 |
| 413 | `payload_too_large` | 업로드 또는 제출 코드 크기 초과 |
| 415 | `unsupported_media_type` | 허용하지 않는 파일 MIME type |
| 422 | `validation_error` | request schema 또는 domain validation 실패 |
| 429 | `rate_limited` | 로그인, OTP 등 rate limit 초과 |
| 500 | `internal_error` | 예상하지 못한 서버 오류 |
| 503 | `service_unavailable` | DB, queue, judge infra 등 일시 장애 |

## Validation Error 형식

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed.",
    "request_id": "req_01JABCDEF0000000000000000",
    "details": {
      "fields": [
        {
          "path": "body.team_name",
          "code": "required",
          "message": "team_name is required."
        }
      ]
    }
  }
}
```

field error의 `code`는 아래 값을 우선 사용한다.

- `required`
- `invalid_format`
- `invalid_enum`
- `too_short`
- `too_long`
- `out_of_range`
- `duplicate`
- `not_allowed`
- `sum_mismatch`

## 주요 실패 케이스

각 실패 케이스는 해당 기능 구현 시 API 테스트로 최소 1개 이상 확인한다.
권한, 상태 전이, validation, idempotency 오류는 성공 케이스와 같은 PR 또는 같은 작업 단위에서 함께 테스트한다.

### 인증

| 상황 | status | code |
| --- | --- | --- |
| staff email/password 불일치 | 401 | `invalid_credentials` |
| 비활성 staff 계정 로그인 | 403 | `permission_denied` |
| refresh token 재사용 또는 DB hash 불일치 | 401 | `invalid_token` |
| password reset token 만료 | 410 | `gone` |
| login rate limit 초과 | 429 | `rate_limited` |

### 대회 관리

| 상황 | status | code |
| --- | --- | --- |
| 존재하지 않거나 숨겨야 하는 contest | 404 | `not_found` |
| `ended` 대회를 `running`으로 직접 변경 | 409 | `invalid_state_transition` |
| 시작 시각이 종료 시각 이후 | 422 | `validation_error` |
| 권한 scope 밖의 contest 수정 | 403 | `scope_denied` |
| 마지막 service master 비활성화 시도 | 409 | `conflict` |

### 참가팀

| 상황 | status | code |
| --- | --- | --- |
| 같은 대회 내 team login identity 중복 | 409 | `conflict` |
| 운영자/운영매니저를 참가팀으로 등록 | 409 | `role_conflict` |
| disabled/disqualified 팀의 로그인 | 403 | `permission_denied` |
| OTP 만료 또는 이미 검증됨 | 410 | `gone` |
| OTP 시도 횟수 초과 | 429 | `rate_limited` |

### 문제/테스트케이스

| 상황 | status | code |
| --- | --- | --- |
| running 대회에서 문제 본문 수정 | 409 | `invalid_state_transition` |
| 점수제 문제의 subtask 점수 합계가 100이 아님 | 422 | `validation_error` |
| 지원하지 않는 리소스 MIME type | 415 | `unsupported_media_type` |
| 테스트케이스 업로드 크기 초과 | 413 | `payload_too_large` |

### 제출/채점

| 상황 | status | code |
| --- | --- | --- |
| 대회 시작 전 문제 조회 또는 제출 | 403 | `permission_denied` |
| 종료된 대회 제출 | 409 | `invalid_state_transition` |
| 지원하지 않는 언어 | 422 | `validation_error` |
| 제출 코드 크기 초과 | 413 | `payload_too_large` |
| judge job result의 lease token 불일치 | 409 | `lease_conflict` |
| 이미 final 반영된 attempt 결과 재반영 | 409 | `conflict` |
| judge infra 일시 장애 | 503 | `service_unavailable` |

## 보안 응답 원칙

- 공개 API에서는 비공개/숨김/예약 리소스에 대해 `404 not_found`를 반환한다.
- 로그인 실패는 `invalid_credentials` 하나로 통일한다.
- 권한 없는 제출 source 조회는 리소스 존재 여부와 관계없이 `403 permission_denied`를 반환한다.
- 내부 API는 외부 Nginx route에 노출하지 않는다.
