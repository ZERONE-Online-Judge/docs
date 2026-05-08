# Zerone Online Judge Docs

이 폴더는 `Zerone Online Judge`의 요구사항 문서를 카테고리별로 분리한 메인 문서 폴더다.
기존 단일 문서는 `docs/99-archive/`에 보관하고, 현재 운영 기준 문서는 이 폴더 아래 개별 문서를 기준으로 관리한다.

## 문서 구조

### 00-overview

- [project-overview.md](./00-overview/project-overview.md): 프로젝트 방향, 제품 목표, 문서 운영 원칙

### 01-platform

- [access-flow.md](./01-platform/access-flow.md): 전체 접속 플로우
- [frontend-requirements.md](./01-platform/frontend-requirements.md): 프론트 요구사항
- [backend-requirements.md](./01-platform/backend-requirements.md): 백엔드 요구사항
- [server-requirements.md](./01-platform/server-requirements.md): 서버/채점 인프라 요구사항
- [deployment-and-runtime.md](./01-platform/deployment-and-runtime.md): 배포 방식, 런타임 구성, Docker Compose 운영 기준
- [judge-protocol.md](./01-platform/judge-protocol.md): dispatcher-assigned 채점 프로토콜
- [docker-compose-runtime.md](./01-platform/docker-compose-runtime.md): backend/judge-agent Compose 분리 운영 기준
- [backup-and-restore.md](./01-platform/backup-and-restore.md): PostgreSQL/MinIO 백업과 복구 기준

### 02-public-pages

- [main-page-flow.md](./02-public-pages/main-page-flow.md): 메인 페이지 플로우
- [contest-list-page-flow.md](./02-public-pages/contest-list-page-flow.md): 대회 목록 페이지 플로우
- [contest-participation-login-flow.md](./02-public-pages/contest-participation-login-flow.md): 대회 참가 로그인 플로우

### 03-permissions

- [service-level-permissions.md](./03-permissions/service-level-permissions.md): 서비스 마스터/서비스 매니저 권한 목록
- [contest-level-permissions.md](./03-permissions/contest-level-permissions.md): 대회 운영자/대회 운영매니저 권한 목록

### 04-contest

- [contest-components.md](./04-contest/contest-components.md): 대회 구성 요소
- [contest-page-structure.md](./04-contest/contest-page-structure.md): 대회 페이지 구성
- [contest-features.md](./04-contest/contest-features.md): 대회별 기능
- [contest-scoring-systems.md](./04-contest/contest-scoring-systems.md): 대회 채점 제도 구분
- [contest-rules.md](./04-contest/contest-rules.md): 대회 규정
- [contest-operations-flow.md](./04-contest/contest-operations-flow.md): 대회 운영 플로우
- [contest-board-operations.md](./04-contest/contest-board-operations.md): 대회 게시판 운영
- [contest-announcement-operations.md](./04-contest/contest-announcement-operations.md): 대회 공지사항 운영

### 05-operator

- [operator-page-structure.md](./05-operator/operator-page-structure.md): 대회 운영자 페이지 구성
- [operator-participant-management.md](./05-operator/operator-participant-management.md): 참가자 추가/삭제/변경
- [operator-problem-management.md](./05-operator/operator-problem-management.md): 문제 추가 및 수정
- [operator-problem-page-details.md](./05-operator/operator-problem-page-details.md): 문제 관리 페이지 상세
- [operator-stage-policies.md](./05-operator/operator-stage-policies.md): 대회 시작/진행/종료 단계별 운영 범위와 규정

### 06-service-admin

- [service-master-page.md](./06-service-admin/service-master-page.md): 서비스 마스터 페이지
- [service-management-page.md](./06-service-admin/service-management-page.md): 서비스 관리 페이지
- [service-announcement-board.md](./06-service-admin/service-announcement-board.md): 서비스 공지 게시판

### 07-architecture

- [ddd.md](./07-architecture/ddd.md): DDD 초안
- [permission-catalog.md](./07-architecture/permission-catalog.md): 백엔드 구현용 canonical 권한 카탈로그
- [api-error-contract.md](./07-architecture/api-error-contract.md): 백엔드 공통 오류 응답과 실패 케이스
- [backend-db-schema-mvp.md](./07-architecture/backend-db-schema-mvp.md): MVP DB 테이블/컬럼/제약 초안

### 08-page-ddd

- [root-public-pages.md](./08-page-ddd/root-public-pages.md): 메인/루트 공개 페이지 DDD 플로우차트와 다이어그램
- [service-notice-pages.md](./08-page-ddd/service-notice-pages.md): 서비스 공지 게시판 공개/운영 페이지 DDD 플로우차트와 다이어그램
- [contest-rules-pages.md](./08-page-ddd/contest-rules-pages.md): 대회 규정집 공개 페이지 DDD 플로우차트와 다이어그램
- [contest-participant-pages.md](./08-page-ddd/contest-participant-pages.md): 대회 참가자 페이지 DDD 플로우차트와 다이어그램
- [contest-operator-pages.md](./08-page-ddd/contest-operator-pages.md): 대회 운영자 페이지 DDD 플로우차트와 다이어그램
- [public-judge-status-pages.md](./08-page-ddd/public-judge-status-pages.md): 공개 채점 서버 현황 페이지 DDD 플로우차트와 다이어그램
- [staff-login-pages.md](./08-page-ddd/staff-login-pages.md): 서비스 관리자/권한자 로그인 페이지와 모달 DDD 플로우차트와 다이어그램
- [service-admin-pages.md](./08-page-ddd/service-admin-pages.md): 서비스 관리자 페이지 DDD 플로우차트와 다이어그램
- [service-admin-contest-pages.md](./08-page-ddd/service-admin-contest-pages.md): 서비스 관리자 대회 관리 페이지 DDD 플로우차트와 다이어그램
- [service-manager-pages.md](./08-page-ddd/service-manager-pages.md): 서비스 매니저 관리 페이지 DDD 플로우차트와 다이어그램
- [service-judge-pages.md](./08-page-ddd/service-judge-pages.md): 서비스 관리자 채점 인프라 페이지 DDD 플로우차트와 다이어그램
- [service-mail-template-pages.md](./08-page-ddd/service-mail-template-pages.md): 서비스 관리자 메일 템플릿 페이지 DDD 플로우차트와 다이어그램

### 09-visual-summary

- [README.md](./09-visual-summary/README.md): 사람이 보는 구현 요약 문서 안내
- [pages-and-api.md](./09-visual-summary/pages-and-api.md): 페이지, API, 정상/에러 응답 포맷 요약
- [backend-judge-flow.md](./09-visual-summary/backend-judge-flow.md): 백엔드와 채점기 동작 방식 요약
- [operations-and-permissions.md](./09-visual-summary/operations-and-permissions.md): 대회 운영 방식과 권한명 요약
- [decided-policies.md](./09-visual-summary/decided-policies.md): 확정된 운영/채점/공개/메일/백업 정책 요약

### 10-implementation-progress

- [README.md](./10-implementation-progress/README.md): 프론트엔드/백엔드 구현 진행 상황과 검증 로그

### 11-api

- [README.md](./11-api/README.md): 구현된 백엔드 API 주소별 파라미터, 정상 응답, 에러 응답 문서

### 99-archive

- 이전 통합 문서 보관

## 관리 원칙

- 제품 요구사항은 최대한 작은 문서 단위로 분리한다.
- 화면 흐름, 권한, 운영 규정은 서로 다른 문서에서 관리한다.
- 아키텍처/DDD 문서는 제품 요구사항 문서와 분리한다.
- 페이지별 DDD 다이어그램은 `08-page-ddd`에서 관리한다.
- 새로운 주제가 생기면 가장 가까운 카테고리에 새 문서를 추가하고, `docs/README.md`에 즉시 링크를 등록한다.

## 브라우저 뷰어

정적 생성 대신 임시 백엔드가 `docs/` 폴더를 직접 읽어 문서 뷰어를 제공한다.

```bash
uvicorn backend.app.main:app --reload
```

브라우저에서 `http://127.0.0.1:8000/docs` 로 접속하면 된다.
