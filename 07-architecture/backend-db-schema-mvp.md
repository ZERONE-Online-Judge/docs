# 백엔드 MVP DB 스키마 초안

이 문서는 초기 백엔드 구현에서 바로 migration으로 옮길 수 있는 수준의 테이블명, 컬럼명, 주요 제약을 정리한다.
세부 타입은 PostgreSQL 기준이다.

## 공통 규칙

- 기본 PK는 UUID를 사용한다.
- 모든 운영 테이블은 `created_at timestamptz not null`, `updated_at timestamptz not null`을 가진다.
- soft delete가 필요한 테이블은 `deleted_at timestamptz null`을 둔다.
- enum은 PostgreSQL enum 또는 check constraint로 시작할 수 있다. 코드에서는 Python enum으로 고정한다.
- 외부에 노출되는 id도 UUID를 그대로 사용한다. 별도 short code가 필요한 경우 명시 컬럼을 추가한다.
- JSON 컬럼은 `jsonb`를 사용한다.

## Enum

| enum | values |
| --- | --- |
| `staff_account_status` | `invited`, `active`, `disabled` |
| `grant_status` | `active`, `revoked` |
| `scope_type` | `global`, `contest` |
| `contest_status` | `draft`, `scheduled`, `open`, `running`, `ended`, `finalized`, `archived`, `deleted` |
| `staff_assignment_role` | `operator`, `manager` |
| `assignment_status` | `active`, `removed` |
| `team_status` | `invited`, `active`, `disqualified`, `disabled` |
| `team_member_role` | `leader`, `member` |
| `otp_status` | `issued`, `verified`, `expired`, `blocked` |
| `team_session_status` | `active`, `revoked`, `expired`, `replaced` |
| `scoring_policy_type` | `ac_wa`, `score` |
| `problem_status` | `draft`, `published`, `hidden`, `locked`, `deleted` |
| `submission_status` | `waiting`, `preparing`, `judging`, `accepted`, `presentation_error`, `wrong_answer`, `time_limit_exceeded`, `memory_limit_exceeded`, `output_limit_exceeded`, `runtime_error`, `compile_error`, `system_error`, `canceled` |
| `judge_job_status` | `pending`, `assigned`, `running`, `succeeded`, `failed`, `canceled`, `expired` |
| `judge_attempt_status` | `assigned`, `running`, `succeeded`, `failed`, `canceled`, `expired` |
| `judge_node_status` | `active`, `draining`, `disabled`, `lost` |
| `notice_status` | `draft`, `published`, `hidden`, `deleted` |
| `visibility` | `public`, `private` |
| `mail_queue_status` | `pending`, `sending`, `sent`, `failed`, `canceled` |

## Identity And Access

### `staff_accounts`

| column | type | constraint |
| --- | --- | --- |
| `staff_account_id` | uuid | pk |
| `email` | citext | not null, unique |
| `display_name` | text | not null |
| `account_status` | staff_account_status | not null |
| `is_service_master` | boolean | not null default false |
| `password_hash` | text | null until setup |
| `invited_at` | timestamptz | null |
| `activated_at` | timestamptz | null |
| `disabled_at` | timestamptz | null |
| `last_login_at` | timestamptz | null |
| `password_changed_at` | timestamptz | null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

Index:

- unique `staff_accounts_email_key (email)`
- partial index `staff_accounts_service_master_active_idx` where `is_service_master = true and account_status = 'active'`

### `staff_sessions`

| column | type | constraint |
| --- | --- | --- |
| `staff_session_id` | uuid | pk |
| `staff_account_id` | uuid | fk staff_accounts |
| `refresh_token_hash` | text | not null |
| `device_fingerprint` | text | null |
| `ip_address` | inet | null |
| `user_agent` | text | null |
| `issued_at` | timestamptz | not null |
| `expires_at` | timestamptz | not null |
| `revoked_at` | timestamptz | null |
| `last_seen_at` | timestamptz | null |

### `permission_grants`

| column | type | constraint |
| --- | --- | --- |
| `permission_grant_id` | uuid | pk |
| `staff_account_id` | uuid | fk staff_accounts |
| `permission_code` | text | not null |
| `scope_type` | scope_type | not null |
| `scope_contest_id` | uuid | nullable fk contests |
| `grant_status` | grant_status | not null |
| `granted_by_staff_account_id` | uuid | fk staff_accounts |
| `granted_at` | timestamptz | not null |
| `revoked_at` | timestamptz | null |

Constraint:

- `scope_type = 'global'`이면 `scope_contest_id is null`
- `scope_type = 'contest'`이면 `scope_contest_id is not null`
- unique active grant: `(staff_account_id, permission_code, scope_type, scope_contest_id)` where `grant_status = 'active'`

### `staff_secret_tokens`

초대, 비밀번호 설정, 비밀번호 재설정 토큰을 통합 저장한다.

| column | type | constraint |
| --- | --- | --- |
| `staff_secret_token_id` | uuid | pk |
| `staff_account_id` | uuid | fk staff_accounts |
| `token_type` | text | `invitation`, `password_setup`, `password_reset` |
| `token_hash` | text | not null |
| `expires_at` | timestamptz | not null |
| `used_at` | timestamptz | null |
| `created_at` | timestamptz | not null |

## Contest Administration

### `contests`

| column | type | constraint |
| --- | --- | --- |
| `contest_id` | uuid | pk |
| `title` | text | not null |
| `organization_name` | text | null |
| `overview` | text | null |
| `status` | contest_status | not null |
| `open_at` | timestamptz | null |
| `start_at` | timestamptz | null |
| `end_at` | timestamptz | null |
| `freeze_at` | timestamptz | null |
| `scoring_policy_type` | scoring_policy_type | not null |
| `score_calculation_policy` | jsonb | not null default `{}` |
| `duplicate_rank_allowed` | boolean | not null default true |
| `tie_break_policy` | jsonb | not null default `{}` |
| `problem_public_after_end` | boolean | not null default false |
| `scoreboard_public_after_end` | boolean | not null default false |
| `submission_public_after_end` | boolean | not null default false |
| `final_scoreboard_published_at` | timestamptz | null |
| `created_by_staff_account_id` | uuid | fk staff_accounts |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |
| `deleted_at` | timestamptz | null |

### `contest_divisions`

한 대회 안의 참가 유형을 저장한다.
참가 유형은 대회 이름과 일정을 공유하지만 문제, 제출, 스코어보드, 채점 이력은 완전히 분리한다.
참가 유형이 2개 이상이면 참가팀은 반드시 하나의 유형에 배정되어야 하며, 한 참가팀이 여러 유형에 중복 배정될 수 없다.
구현 관점에서는 `contest_id`가 같더라도 `contest_division_id`가 다르면 사실상 다른 대회처럼 취급한다.

| column | type | constraint |
| --- | --- | --- |
| `contest_division_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `division_code` | text | not null |
| `division_name` | text | not null |
| `description` | text | null |
| `display_order` | integer | not null default 1 |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |
| `deleted_at` | timestamptz | null |

Constraint:

- unique `(contest_id, lower(division_code))` where `deleted_at is null`
- unique `(contest_id, lower(division_name))` where `deleted_at is null`

### `contest_staff_assignments`

| column | type | constraint |
| --- | --- | --- |
| `contest_staff_assignment_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `staff_account_id` | uuid | fk staff_accounts |
| `assignment_role` | staff_assignment_role | not null |
| `permission_overrides` | jsonb | not null default `[]` |
| `assignment_status` | assignment_status | not null |
| `assigned_by_staff_account_id` | uuid | fk staff_accounts |
| `assigned_at` | timestamptz | not null |
| `removed_at` | timestamptz | null |

Constraint:

- unique active assignment: `(contest_id, staff_account_id)` where `assignment_status = 'active'`

## Team Participation

### `participant_teams`

| column | type | constraint |
| --- | --- | --- |
| `participant_team_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `contest_division_id` | uuid | fk contest_divisions |
| `team_name` | text | not null |
| `team_status` | team_status | not null |
| `team_size` | integer | not null, `team_size >= 1` |
| `first_login_at` | timestamptz | null |
| `last_login_at` | timestamptz | null |
| `last_otp_verified_at` | timestamptz | null |
| `disqualified_at` | timestamptz | null |
| `disqualified_reason` | text | null |
| `created_by_staff_account_id` | uuid | fk staff_accounts |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |
| `deleted_at` | timestamptz | null |

Index:

- unique active team name per contest: `(contest_id, lower(team_name))` where `deleted_at is null`
- 참가팀은 정확히 하나의 `contest_division_id`를 가진다.

### `team_members`

| column | type | constraint |
| --- | --- | --- |
| `team_member_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `participant_team_id` | uuid | fk participant_teams |
| `member_role` | team_member_role | not null |
| `member_name` | text | not null |
| `email` | citext | not null |
| `member_status` | text | not null default `active` |
| `max_concurrent_sessions` | integer | not null default 1 |
| `otp_rate_limit_key` | text | null |
| `last_login_at` | timestamptz | null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

Constraint:

- unique `(contest_id, email)`
- exactly one `leader` per participant team is enforced by application validation or partial unique index.
- 로그인, OTP, 세션, 동시 접속 제한은 `team_members.email` 단위로 처리한다.

### `otp_challenges`

| column | type | constraint |
| --- | --- | --- |
| `otp_challenge_id` | uuid | pk |
| `team_member_id` | uuid | fk team_members |
| `otp_hash` | text | not null |
| `expires_at` | timestamptz | not null |
| `attempt_count` | integer | not null default 0 |
| `max_attempt_count` | integer | not null |
| `request_rate_limit_key` | text | not null |
| `ip_rate_limit_key` | text | null |
| `challenge_status` | otp_status | not null |
| `issued_at` | timestamptz | not null |
| `verified_at` | timestamptz | null |
| `blocked_at` | timestamptz | null |

### `team_sessions`

| column | type | constraint |
| --- | --- | --- |
| `team_session_id` | uuid | pk |
| `participant_team_id` | uuid | fk participant_teams |
| `team_member_id` | uuid | fk team_members |
| `refresh_token_hash` | text | not null |
| `session_status` | team_session_status | not null |
| `device_fingerprint` | text | null |
| `ip_address` | inet | null |
| `user_agent` | text | null |
| `issued_at` | timestamptz | not null |
| `expires_at` | timestamptz | not null |
| `last_seen_at` | timestamptz | null |
| `replaced_by_session_id` | uuid | null |

## Problem Management

### `problems`

| column | type | constraint |
| --- | --- | --- |
| `problem_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `contest_division_id` | uuid | fk contest_divisions |
| `problem_code` | text | not null |
| `title` | text | not null |
| `statement_content_id` | uuid | nullable fk rich_text_contents |
| `input_description` | text | null |
| `output_description` | text | null |
| `constraints_text` | text | null |
| `time_limit_ms` | integer | not null |
| `memory_limit_mb` | integer | not null |
| `display_order` | integer | not null |
| `scoring_policy_type` | scoring_policy_type | not null |
| `max_score` | integer | not null default 100 |
| `subtask_count` | integer | not null default 0 |
| `visibility_policy` | visibility | not null default `private` |
| `problem_status` | problem_status | not null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |
| `deleted_at` | timestamptz | null |

Constraint:

- unique `(contest_id, contest_division_id, problem_code)` where `deleted_at is null`
- unique `(contest_id, contest_division_id, display_order)` where `deleted_at is null`

### `problem_assets`

| column | type | constraint |
| --- | --- | --- |
| `asset_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `problem_id` | uuid | fk problems |
| `original_filename` | text | not null |
| `storage_key` | text | not null unique |
| `mime_type` | text | not null |
| `file_size` | bigint | not null |
| `sha256` | text | not null |
| `uploaded_by_staff_account_id` | uuid | fk staff_accounts |
| `asset_status` | text | not null default `active` |
| `created_at` | timestamptz | not null |
| `deleted_at` | timestamptz | null |

### `testcase_sets`

| column | type | constraint |
| --- | --- | --- |
| `testcase_set_id` | uuid | pk |
| `problem_id` | uuid | fk problems |
| `version` | integer | not null |
| `is_active` | boolean | not null default false |
| `created_by_staff_account_id` | uuid | fk staff_accounts |
| `created_at` | timestamptz | not null |

### `testcases`

| column | type | constraint |
| --- | --- | --- |
| `testcase_id` | uuid | pk |
| `testcase_set_id` | uuid | fk testcase_sets |
| `display_order` | integer | not null |
| `input_storage_key` | text | not null |
| `output_storage_key` | text | not null |
| `input_sha256` | text | not null |
| `output_sha256` | text | not null |
| `time_limit_ms_override` | integer | null |
| `memory_limit_mb_override` | integer | null |
| `created_at` | timestamptz | not null |

## Submission And Judge

### `submissions`

| column | type | constraint |
| --- | --- | --- |
| `submission_id` | uuid | pk |
| `contest_id` | uuid | fk contests |
| `contest_division_id` | uuid | fk contest_divisions |
| `problem_id` | uuid | fk problems |
| `participant_team_id` | uuid | fk participant_teams |
| `language` | text | not null |
| `source_code_storage_key` | text | not null |
| `source_code_sha256` | text | not null |
| `submitted_at` | timestamptz | not null |
| `current_status` | submission_status | not null |
| `status_updated_at` | timestamptz | not null |
| `compile_message` | text | null |
| `runtime_summary` | jsonb | not null default `{}` |
| `awarded_score` | numeric(6,2) | null |
| `score_applied_at` | timestamptz | null |
| `score_achieved_submission_at` | timestamptz | null |
| `is_finalized` | boolean | not null default false |
| `final_result_at` | timestamptz | null |

Index:

- `(contest_id, submitted_at, submission_id)`
- `(contest_id, contest_division_id, submitted_at, submission_id)`
- `(participant_team_id, problem_id, submitted_at desc)`

### `judge_jobs`

| column | type | constraint |
| --- | --- | --- |
| `judge_job_id` | uuid | pk |
| `submission_id` | uuid | fk submissions |
| `contest_id` | uuid | fk contests |
| `contest_division_id` | uuid | fk contest_divisions |
| `problem_id` | uuid | fk problems |
| `participant_team_id` | uuid | fk participant_teams |
| `job_status` | judge_job_status | not null |
| `queue_position` | bigint | not null |
| `priority` | integer | not null default 0 |
| `leased_node_id` | uuid | nullable fk judge_nodes |
| `lease_token_hash` | text | null |
| `leased_at` | timestamptz | null |
| `lease_expires_at` | timestamptz | null |
| `retry_count` | integer | not null default 0 |
| `last_error_code` | text | null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

Index:

- pending FIFO: `(priority desc, queue_position asc)` where `job_status = 'pending'`
- lease expiry: `(lease_expires_at)` where `job_status in ('assigned', 'running')`

### `judge_job_attempts`

| column | type | constraint |
| --- | --- | --- |
| `judge_job_attempt_id` | uuid | pk |
| `judge_job_id` | uuid | fk judge_jobs |
| `judge_node_id` | uuid | fk judge_nodes |
| `attempt_no` | integer | not null |
| `attempt_status` | judge_attempt_status | not null |
| `lease_token_hash` | text | not null |
| `assigned_at` | timestamptz | not null |
| `started_at` | timestamptz | null |
| `finished_at` | timestamptz | null |
| `result_payload` | jsonb | null |
| `error_code` | text | null |
| `error_message` | text | null |

Constraint:

- unique `(judge_job_id, attempt_no)`
- result apply must be idempotent by `judge_job_attempt_id`

### `judge_nodes`

| column | type | constraint |
| --- | --- | --- |
| `judge_node_id` | uuid | pk |
| `node_name` | text | not null unique |
| `node_secret_hash` | text | not null |
| `tailscale_node_identity` | text | null |
| `tailscale_ip` | inet | null |
| `node_status` | judge_node_status | not null |
| `total_slots` | integer | not null |
| `free_slots` | integer | not null |
| `running_job_count` | integer | not null default 0 |
| `region` | text | null |
| `host_group` | text | null |
| `last_heartbeat_at` | timestamptz | null |
| `last_seen_version` | text | null |
| `schedulable` | boolean | not null default true |
| `registered_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

## Content, Notice, Audit

### `rich_text_contents`

| column | type | constraint |
| --- | --- | --- |
| `content_id` | uuid | pk |
| `owner_type` | text | not null |
| `owner_id` | uuid | not null |
| `markdown_source` | text | not null |
| `allowed_markdown_policy` | text | not null |
| `latex_enabled` | boolean | not null default true |
| `katex_render_enabled` | boolean | not null default true |
| `asset_references` | jsonb | not null default `[]` |
| `sanitized_render_cache` | text | null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

### `service_notices`

| column | type | constraint |
| --- | --- | --- |
| `service_notice_id` | uuid | pk |
| `title` | text | not null |
| `body_content_id` | uuid | fk rich_text_contents |
| `summary` | text | null |
| `notice_type` | text | not null |
| `notice_status` | notice_status | not null |
| `visibility` | visibility | not null |
| `pinned` | boolean | not null default false |
| `emergency` | boolean | not null default false |
| `published_at` | timestamptz | null |
| `publish_start_at` | timestamptz | null |
| `publish_end_at` | timestamptz | null |
| `created_by_staff_account_id` | uuid | fk staff_accounts |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |
| `hidden_at` | timestamptz | null |

### `audit_log_entries`

| column | type | constraint |
| --- | --- | --- |
| `audit_log_entry_id` | uuid | pk |
| `actor_type` | text | not null |
| `actor_id` | uuid | null |
| `action_code` | text | not null |
| `target_type` | text | not null |
| `target_id` | uuid | null |
| `contest_id` | uuid | nullable fk contests |
| `summary` | text | not null |
| `before_snapshot` | jsonb | null |
| `after_snapshot` | jsonb | null |
| `occurred_at` | timestamptz | not null |
| `ip_address` | inet | null |

Index:

- `(contest_id, occurred_at desc)`
- `(actor_id, occurred_at desc)`
- `(target_type, target_id, occurred_at desc)`

## Mail

### `mail_queue`

API 서버는 메일을 직접 즉시 발송하지 않고 이 테이블에 적재한다.
`mail-worker`가 pending row를 claim해서 SMTP로 발송한다.

| column | type | constraint |
| --- | --- | --- |
| `mail_queue_id` | uuid | pk |
| `mail_type` | text | not null |
| `recipient_email` | citext | not null |
| `subject` | text | not null |
| `body_text` | text | null |
| `body_html` | text | null |
| `template_key` | text | null |
| `template_variables` | jsonb | not null default `{}` |
| `queue_status` | mail_queue_status | not null |
| `attempt_count` | integer | not null default 0 |
| `max_attempt_count` | integer | not null default 5 |
| `next_attempt_at` | timestamptz | not null |
| `claimed_at` | timestamptz | null |
| `sent_at` | timestamptz | null |
| `last_error_code` | text | null |
| `last_error_message` | text | null |
| `created_at` | timestamptz | not null |
| `updated_at` | timestamptz | not null |

Index:

- pending mail claim: `(next_attempt_at, created_at)` where `queue_status = 'pending'`
- recipient history: `(recipient_email, created_at desc)`

## 구현 순서

1. `staff_accounts`, `staff_sessions`, `permission_grants`, `staff_secret_tokens`
2. `contests`, `contest_divisions`, `contest_staff_assignments`
3. `participant_teams`, `team_members`, `otp_challenges`, `team_sessions`
4. `problems`, `problem_assets`, `testcase_sets`, `testcases`
5. `submissions`, `judge_nodes`, `judge_jobs`, `judge_job_attempts`
6. `rich_text_contents`, `service_notices`, `audit_log_entries`, `mail_queue`

## 보류 항목

- contest announcement와 clarification board는 MVP 2단계에서 별도 테이블로 확장한다.
- scoreboard projection은 제출/채점 기본 흐름이 안정된 뒤 `contest_scoreboards`, `contest_scoreboard_entries`로 분리한다.
- object storage는 MinIO를 기준으로 하고, `storage_key`는 MinIO object key로 다룬다.
