# 권한 목록(대회 운영자, 대회 운영매니저)

## 대회 운영자

- 특정 대회의 전체 운영 권한
- 해당 대회의 설정, 참가 팀, 문제, 스코어보드, 공지, 질문, 리소스를 관리할 수 있다.
- 해당 대회의 대회 운영매니저를 초대/권한 부여/추방할 수 있다.

## 대회 운영매니저

- 대회 운영자가 부여하는 대회 단위 세부 권한
- 고정 역할이 아니라 체크리스트 기반 permission 묶음
- 부여받은 권한만 사용할 수 있어야 한다

## 대회 운영매니저 권한 카탈로그

구현 시 canonical permission code는 [백엔드 권한 카탈로그](../07-architecture/permission-catalog.md)를 기준으로 한다.

### 대회 기본 정보

- `contest.view`
- `contest.update_overview`
- `contest.update_organization`
- `contest.update_rule`
- `contest.update_schedule`

### 대회 참가 설정

- `contest.participant_policy.view`
- `contest.participant_policy.update`

### 대회 채점 / 점수 계산 설정

- `contest.scoring_policy.view`
- `contest.scoring_policy.update`
- `contest.score_calculation_policy.view`
- `contest.score_calculation_policy.update`

### 참가 팀 관리

- `contest.participant.view`
- `contest.participant.create`
- `contest.participant.update`
- `contest.participant.remove`

### 문제 관리

- `contest.problem.view`
- `contest.problem.create`
- `contest.problem.update`
- `contest.problem.delete`
- `contest.problem.reorder`

### 문제 리소스 / 테스트케이스

- `contest.problem.resource.view`
- `contest.problem.resource.manage`
- `contest.testcase.view`
- `contest.testcase.manage`
- `contest.generator.view`
- `contest.generator.manage`

### 스코어보드 / 팀 현황

- `contest.scoreboard.view`
- `contest.scoreboard.freeze`
- `contest.scoreboard.unfreeze`
- `contest.scoreboard.setting`
- `contest.scoreboard.score_adjust`
- `contest.team_status.view`
- `contest.team_status.realtime_view`

### 공지 / 질문 게시판

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

### 제출 / 운영 인력

- `contest.submission.view`
- `contest.submission.source.view`
- `contest.submission.cancel`
- `contest.submission.export`
- `contest.judge_queue.view`
- `contest.manager.invite`
- `contest.manager.permission_update`
- `contest.manager.remove`

## 역할 충돌 규칙

- 같은 대회에서 운영자/운영매니저는 참가 팀이 될 수 없다.
- 다른 대회에서는 다른 역할을 가질 수 있다.
- 운영자는 해당 대회의 모든 권한을 가진 것으로 처리한다.
