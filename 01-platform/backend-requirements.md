# 백엔드 요구사항

## 역할

백엔드는 단순 제출 API가 아니라 플랫폼 중심 서버여야 한다.
공개 웹/API 계층과 비공개 채점 계층 사이의 제어 지점 역할도 수행해야 한다.
백엔드 API 서버 구현은 FastAPI를 기준으로 한다.

## 핵심 책임

- 인증과 권한 처리
- 서비스 마스터/서비스 매니저 관리
- 대회 생성과 상태 관리
- 대회 운영자/운영매니저 관리
- 참가 팀 관리
- 문제/예제/리소스/테스트케이스 관리
- 제출 생성 및 조회
- 채점 요청 생성 및 결과 반영
- 공지사항/질문 게시판 관리
- 서비스 공지 게시판 관리
- 감사 로그 조회
- 운영자 대시보드와 공개 현황용 데이터 제공

## API 범주

- Auth
- Users / Me
- Roles / Permissions
- Service Managers
- Contests
- Contest Staff
- Contest Participants
- Problems
- Problem Resources
- Problem Examples
- Testcases / Generators
- Submissions / Results
- Judge Queue / Judge Servers / Cluster Settings
- Service Notices
- Audit Logs

## 설계 원칙

- 권한 체크는 엄격해야 한다.
- permission 기반 RBAC와 scope 개념을 가져야 한다.
- 대회 단위 데이터 경계가 명확해야 한다.
- 동일 대회 내 역할 충돌 규칙을 강제해야 한다.
- 제출 저장과 채점 요청 생성 사이에는 유실 방지 장치가 필요하다.
- 참가팀은 팀 이름, 팀장 이름/메일, 팀원 이름/메일로 등록한다.
- 참가자 로그인 식별자는 등록된 참가자 이메일이며, OTP와 세션도 이메일 단위로 관리한다.
- 대회별 채점 제도 유형을 명시적으로 지원해야 한다.
- AC/WA 대회와 점수제 대회의 점수 계산 규칙을 분리해서 처리해야 한다.
- 대회별 점수 계산 정책을 설정값으로 저장하고, 기본값과 대회별 override를 구분해야 한다.
- 점수제 대회에서는 문제별 서브태스크 배점 합계가 `100점`인지 검증할 수 있어야 한다.
- 중요한 운영 작업은 모두 감사 로그에 남겨야 한다.
- 백엔드 구현은 기능 단위로 API 테스트, application service 테스트, domain rule 테스트를 중간중간 작성하면서 진행해야 한다.
- 권한, 상태 전이, transaction, idempotency가 있는 기능은 정상 케이스와 실패 케이스를 함께 테스트해야 한다.

## 인증 구현 기준

- 참가자 로그인은 등록된 참가자 이메일, 이메일 기반 OTP, JWT를 사용한다.
- 권한자 로그인은 이메일과 비밀번호 기반 JWT를 사용한다.
- access token은 짧은 만료 시간을 갖고, refresh token은 서버 DB에 해시 형태로 저장한다.
- refresh token은 기기 세션과 연결하여 다른 기기 로그인 시 기존 세션을 무효화할 수 있어야 한다.
- OTP, 비밀번호 재설정, 초대 토큰처럼 일회성 비밀값은 원문 저장을 금지한다.

## 메일 발송 기준

- SMTP를 직접 사용한다.
- API 서버는 메일 요청을 DB 기반 mail queue에 적재한다.
- 별도 mail worker가 queue를 소비해 SMTP로 발송한다.
- OTP, 권한자 초대, 비밀번호 설정/재설정 메일은 같은 queue 처리 방식을 사용한다.

## 권한 모델 초안

권한 모델은 아직 최종 확정 전이다.
현재 초안은 고정 역할과 permission grant를 함께 사용하는 방식이다.

- `service_master`는 최상위 고정 역할로 두고 permission row 없이 모든 권한 검사를 통과한다.
- `contest_operator`는 특정 대회에 대한 고정 역할로 두고 해당 대회의 모든 대회 권한을 가진 것으로 계산한다.
- `service_manager`와 `contest_manager`는 명시적인 permission grant 묶음으로 권한을 부여한다.
- permission grant에는 적용 scope가 있어야 하며, 전역 또는 특정 대회 범위를 표현할 수 있어야 한다.
- 이 초안은 추후 권한 정책 논의 결과에 따라 조정될 수 있다.

## 관련 문서

- [접속 플로우](./access-flow.md)
- [배포 및 런타임 구성](./deployment-and-runtime.md)
- [Docker Compose 런타임 기준](./docker-compose-runtime.md)
- [채점 프로토콜](./judge-protocol.md)
- [서비스 권한 목록](../03-permissions/service-level-permissions.md)
- [대회 권한 목록](../03-permissions/contest-level-permissions.md)
- [DDD 초안](../07-architecture/ddd.md)
- [백엔드 권한 카탈로그](../07-architecture/permission-catalog.md)
- [백엔드 API 오류 응답 기준](../07-architecture/api-error-contract.md)
- [백엔드 MVP DB 스키마 초안](../07-architecture/backend-db-schema-mvp.md)
