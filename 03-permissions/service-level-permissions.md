# 권한 목록(서비스 마스터, 매니저)

## 서비스 마스터

- 시스템 전체 최상위 권한
- 대회 생성/오픈/종료/삭제 가능
- 서비스 매니저 추가/수정/삭제 가능
- 대회 운영자/운영매니저 직접 관리 가능
- 모든 참가 팀/문제/제출/채점 서버 설정 관리 가능
- 감사 로그 및 운영 데이터 전체 조회 가능

## 서비스 매니저 개념

- 서비스 마스터가 부여하는 전역 운영 권한
- 고정 역할이 아니라 permission grant 묶음 기반
- `global` 또는 `contest_scoped` 범위로 부여 가능
- 같은 계정이 전역 권한과 특정 대회 권한을 함께 가질 수 있음

## 서비스 매니저 권한 카탈로그

구현 시 canonical permission code는 [백엔드 권한 카탈로그](../07-architecture/permission-catalog.md)를 기준으로 한다.

서비스 매니저 권한 중 특정 대회 리소스에 적용되는 권한은 대회 권한 목록과 같은 `contest.*` prefix를 사용한다.
서비스 매니저에게 같은 permission을 `global` scope로 부여하면 여러 대회에 걸쳐 사용할 수 있고, `contest_scoped` scope로 부여하면 특정 대회 안에서만 사용할 수 있다.

### 대회 생성/개최

- `contest.create`
- `contest.open`
- `contest.close`
- `contest.delete`

### 대회 기본 정보

- `contest.view`
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

### 대회 권한 인력 관리

- `contest.operator.view`
- `contest.operator.invite`
- `contest.operator.update`
- `contest.operator.remove`
- `contest.manager.view`
- `contest.manager.invite`
- `contest.manager.permission_update`
- `contest.manager.remove`

### 참가 팀 관리

- `contest.participant.view`
- `contest.participant.create`
- `contest.participant.update`
- `contest.participant.remove`
- `contest.participant.bulk_create`
- `contest.participant.mail_resend`

### 문제/채점 데이터 관리

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

### 제출/스코어보드/인프라

- `contest.submission.view`
- `contest.submission.source.view`
- `contest.submission.cancel`
- `contest.submission.export`
- `contest.scoreboard.view`
- `contest.scoreboard.freeze`
- `contest.scoreboard.unfreeze`
- `contest.scoreboard.setting`
- `contest.scoreboard.score_adjust`
- `contest.judge_queue.view`
- `judge.queue.view`
- `judge.job.retry`
- `judge.job.cancel`
- `judge.server.view`
- `judge.server.scale`
- `judge.server.config_update`
- `judge.cluster.policy_update`

### 대회 공지 / 질문 게시판

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

### 서비스 운영

- `mail.template.view`
- `mail.template.update`
- `audit_log.view`
- `service.notice.view`
- `service.notice.view_private`
- `service.notice.manage`
- `service.notice.emergency_publish`

## 적용 원칙

- 서비스 마스터는 모든 권한 검사를 우회하는 최상위 권한이다.
- 서비스 매니저는 부여받은 권한만 사용할 수 있다.
- 권한 없는 기능은 UI와 API에서 모두 차단해야 한다.
