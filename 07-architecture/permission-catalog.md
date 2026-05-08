# 백엔드 권한 카탈로그

이 문서는 백엔드 구현에서 사용할 canonical permission code를 정리한다.
기존 문서에 섞여 있는 같은 의미의 다른 권한명은 이 문서 기준으로 정리한다.

## 명명 원칙

- permission code는 소문자 snake case segment와 dot separator를 사용한다.
- 리소스 prefix는 실제 도메인 소유권보다 API 권한 판정 단위를 우선한다.
- 같은 동작은 서비스 매니저와 대회 운영매니저가 같은 permission code를 사용하고, 적용 범위는 `scope_type`으로 구분한다.
- `service_master`는 permission row 없이 모든 권한 검사를 통과한다.
- `contest_operator`는 특정 대회 scope에서 모든 `contest.*` 권한을 가진 것으로 계산한다.
- permission 저장값은 이 문서의 canonical code만 허용한다.

## Scope 모델

| scope_type | 의미 | scope_id |
| --- | --- | --- |
| `global` | 서비스 전체 | `null` |
| `contest` | 특정 대회 | `contest_id` |

`global` scope의 `contest.*` 권한은 모든 대회에 적용된다.
`contest` scope의 `contest.*` 권한은 지정 대회에만 적용된다.
`service.*`, `mail.*`, `audit_log.*`, `judge.*` 권한은 기본적으로 `global` scope만 허용한다.

## Canonical Permission Codes

### Contest Lifecycle

- `contest.view`
- `contest.create`
- `contest.open`
- `contest.start`
- `contest.close`
- `contest.finalize`
- `contest.archive`
- `contest.delete`

### Contest Settings

- `contest.update_basic`
- `contest.update_schedule`
- `contest.update_organization`
- `contest.update_overview`
- `contest.update_rule`
- `contest.update_visibility`
- `contest.participant_policy.view`
- `contest.participant_policy.update`
- `contest.scoring_policy.view`
- `contest.scoring_policy.update`
- `contest.score_calculation_policy.view`
- `contest.score_calculation_policy.update`

### Contest Staff

- `contest.operator.view`
- `contest.operator.invite`
- `contest.operator.update`
- `contest.operator.remove`
- `contest.manager.view`
- `contest.manager.invite`
- `contest.manager.permission_update`
- `contest.manager.remove`

### Participants

- `contest.participant.view`
- `contest.participant.create`
- `contest.participant.update`
- `contest.participant.remove`
- `contest.participant.bulk_create`
- `contest.participant.mail_resend`
- `contest.team_status.view`
- `contest.team_status.realtime_view`

### Problems And Judge Data

- `contest.problem.view`
- `contest.problem.create`
- `contest.problem.update`
- `contest.problem.reorder`
- `contest.problem.delete`
- `contest.problem.resource.view`
- `contest.problem.resource.manage`
- `contest.testcase.view`
- `contest.testcase.manage`
- `contest.generator.view`
- `contest.generator.manage`

### Submissions And Scoreboard

- `contest.submission.view`
- `contest.submission.source.view`
- `contest.submission.cancel`
- `contest.submission.export`
- `contest.scoreboard.view`
- `contest.scoreboard.freeze`
- `contest.scoreboard.unfreeze`
- `contest.scoreboard.setting`
- `contest.scoreboard.score_adjust`
- `contest.scoreboard.finalize`
- `contest.judge_queue.view`

### Contest Notice And Board

- `contest.notice.view`
- `contest.notice.create`
- `contest.notice.update`
- `contest.notice.delete`
- `contest.notice.emergency_publish`
- `contest.board.question.view`
- `contest.board.question.update_visibility`
- `contest.board.answer.create`
- `contest.board.answer.update`
- `contest.board.answer.delete`

### Judge Infrastructure

- `judge.queue.view`
- `judge.job.cancel`
- `judge.server.view`
- `judge.server.scale`
- `judge.server.config_update`
- `judge.cluster.policy_update`

### Service Operation

- `service.notice.view`
- `service.notice.view_private`
- `service.notice.manage`
- `service.notice.emergency_publish`
- `mail.template.view`
- `mail.template.update`
- `audit_log.view`

## 중복 권한명 정리

| 기존 권한명 | canonical 권한명 | 비고 |
| --- | --- | --- |
| `contest.info.view` | `contest.view` | 대회 기본 조회 권한으로 통합 |
| `contest.info.update_overview` | `contest.update_overview` | prefix 통일 |
| `contest.info.update_organization` | `contest.update_organization` | prefix 통일 |
| `contest.info.update_rule` | `contest.update_rule` | 기존 서비스 권한 목록에는 없던 항목 추가 |
| `contest.info.update_schedule` | `contest.update_schedule` | prefix 통일 |
| `contest.participant_setting.view` | `contest.participant_policy.view` | API path의 participant-policy와 맞춤 |
| `contest.participant_setting.update` | `contest.participant_policy.update` | API path의 participant-policy와 맞춤 |
| `submission.rejudge` | 사용하지 않음 | 수동 재채점은 서비스 마스터 포함 모든 권한에서 제공하지 않음 |
| `submission.cancel` | `contest.submission.cancel` 또는 `judge.job.cancel` | 제출 취소와 인프라 job 취소를 분리 |
| `contest.judge_queue.view` | `contest.judge_queue.view` | 특정 대회 큐 조회 |
| `judge.queue.view` | `judge.queue.view` | 전역 인프라 큐 조회 |

## API 권한 매핑 기준

| API pattern | required permission |
| --- | --- |
| `GET /admin/contests` | `contest.view` |
| `POST /admin/contests` | `contest.create` |
| `PATCH /admin/contests/{contest_id}` | 변경 필드별 `contest.update_*` |
| `POST /admin/contests/{contest_id}/open` | `contest.open` |
| `POST /admin/contests/{contest_id}/start` | `contest.start` |
| `POST /admin/contests/{contest_id}/close` | `contest.close` |
| `POST /admin/contests/{contest_id}/finalize` | `contest.finalize` |
| `DELETE /admin/contests/{contest_id}` | `contest.delete` |
| `GET /operator/contests/{contest_id}/participants` | `contest.participant.view` |
| `POST /operator/contests/{contest_id}/participants` | `contest.participant.create` |
| `PATCH /operator/contests/{contest_id}/participants/{participant_team_id}` | `contest.participant.update` |
| `DELETE /operator/contests/{contest_id}/participants/{participant_team_id}` | `contest.participant.remove` |
| `GET /operator/contests/{contest_id}/submissions` | `contest.submission.view` |
| `GET /operator/contests/{contest_id}/submissions/{submission_id}/source` | `contest.submission.source.view` |
| `POST /operator/contests/{contest_id}/score-adjustments` | `contest.scoreboard.score_adjust` |
| `POST /operator/contests/{contest_id}/scoreboard/finalize` | `contest.scoreboard.finalize` |
| `GET /admin/judge/dashboard` | `judge.queue.view` or `judge.server.view` |
| `PATCH /admin/judge/nodes/{node_id}/slots` | `judge.server.scale` |
| `POST /admin/judge/jobs/{job_id}/retry` | 제공하지 않음 |
| `POST /admin/judge/jobs/{job_id}/cancel` | `judge.job.cancel` |

## 구현 메모

- 권한 비교는 문자열 직접 비교 대신 enum 또는 DB seed table 기반으로 한다.
- 권한 seed는 migration에서 관리하고, 없는 권한 code가 grant에 들어오면 `422 validation_error`로 거부한다.
- 관리자 UI 메뉴 노출과 API 권한 판정은 같은 permission evaluation 결과를 사용한다.
- 서비스 매니저가 contest scope 권한을 여러 대회에 받을 수 있으므로 `permission_grants`는 row per permission per scope로 저장한다.
