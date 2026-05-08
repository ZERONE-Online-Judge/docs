# 구현 진행 상황

이 문서는 프론트엔드와 백엔드 구현 중 진행상황, 필요사항, 검증 상태를 추적한다.

## 범위

현재 목표는 실제 운영 제품의 뼈대를 따르는 MVP 구현이다.

- 프론트엔드: 실제 서비스 톤의 공개/참가자/운영자 UI
- 백엔드: FastAPI + SQLAlchemy 기반 API 골격과 MVP 데이터 흐름
- 채점: dispatcher/agent 실제 샌드박스 전 단계의 API/상태 모델 골격
- 배포: Docker Compose 기준 파일과 env example 생성 기준

## 진행 체크리스트

| 영역 | 항목 | 상태 | 메모 |
| --- | --- | --- | --- |
| 문서 | 확정 정책 요약 반영 | 완료 | `docs/09-visual-summary/decided-policies.md` |
| 문서 | 구현 진행상황 문서 생성 | 완료 | 이 문서 |
| 문서 | API 주소별 계약 문서 | 완료 | `docs/11-api/README.md` |
| 백엔드 | FastAPI 프로젝트 구조 | 완료 | `backend/app` |
| 백엔드 | SQLAlchemy DB 기반 | 완료 | SQLite local 기본값, PostgreSQL URL 지원 |
| 백엔드 | Alembic migration 골격 | 완료 | `backend/migrations/versions/0001_initial_core.py` |
| 백엔드 | 권한자 DB 세션 로그인 | 완료 | PBKDF2 password hash, access/refresh token 저장 |
| 백엔드 | JWT 기반 세션 토큰 | 완료 | HS256 signed JWT 발급 + DB token hash/revoke 검증 병행 |
| 백엔드 | 운영자/서비스 관리자 이메일 OTP 로그인 | 완료 | staff OTP request/verify, mail queue 적재 |
| 백엔드 | 일반 로그인 세션 API | 완료 | 참가자/대회 운영자 공통 이메일 OTP, 내 대회 목록, 참가자 세션 발급 |
| 백엔드 | OTP 재요청 쿨다운 | 완료 | 참가자/운영자 이메일 OTP 요청 10초 서버 제한, `429 otp_request_rate_limited` |
| 백엔드 | 관리자/운영자 API 인증 보호 | 완료 | service master, contest scope 검사 |
| 백엔드 | 공통 응답/에러 포맷 | 완료 | request_id 포함 |
| 백엔드 | 공개/참가자/관리자/운영자 API | 완료 | 데모 데이터 기반 |
| 백엔드 | 참가자 이메일 OTP 로그인 모델 | 완료 | mail queue 적재, OTP 만료, participant session 저장 |
| 백엔드 | 참가자 세션 조회 API | 완료 | Bearer token 기반 `/participant-session/me` |
| 백엔드 | 참가자 제출 API 인증 보호 | 완료 | 제출/목록/상세/status를 participant token 기준으로 제한 |
| 백엔드 | 대회 참가 유형 division 모델 | 완료 | 팀/문제/제출/스코어보드 유형 분리 |
| 백엔드 | 문제 CRUD API | 완료 | 운영자 scope 보호, 유형별 문제 생성/수정 |
| 백엔드 | 문제 리소스/테스트케이스 메타데이터 | 완료 | MinIO storage_key 기반 asset/testcase 저장 |
| 백엔드 | object storage presigned URL | 완료 | MinIO/local backend, upload/download URL 발급 |
| 백엔드 | 제출 생성 및 judge job 생성 | 완료 | DB transaction 기반 처리 |
| 백엔드 | long polling 제출 상태 API | 완료 | 데모 상태 갱신 |
| 백엔드 | 내부 judge API | 완료 | register/heartbeat/claim/result |
| 백엔드 | 내부 judge API node secret 검증 | 완료 | secret hash 저장, heartbeat/claim/result 검증 |
| 백엔드 | 스코어보드 점수 반영 정책 | 완료 | 팀-문제별 최고 점수만 반영, 같은 점수 기존 제출 유지, 컴파일 에러 제외 |
| 백엔드 | 참가자/공개 리소스 접근 제한 | 완료 | 문제/스코어보드/제출 현황은 참가자 인증 또는 종료 후 수동 공개 필요 |
| 백엔드 | 운영자 대회 설정 수정 API | 완료 | 일정/상태/프리즈/종료 후 공개/긴급공지 PATCH, 시간 검증 포함 |
| 백엔드 | 운영자 참가팀 상태 변경 API | 완료 | 팀명/참가 유형/상태 PATCH, 삭제 없음, disabled/disqualified 상태 관리 |
| 백엔드 | 운영자 팀원 관리 API | 완료 | 팀원 추가/이름·이메일 수정/이메일별 세션 종료, 삭제 없음 |
| 백엔드 | 참가 유형 정책 보강 | 완료 | 이름/설명 중심, 대회 내 이름 중복 방지, 이름순 정렬 |
| 백엔드 | 참가팀 일괄 등록 API | 완료 | CSV/TSV 엑셀 저장 형식 파싱 결과를 bulk create로 등록, 행별 실패 반환 |
| 백엔드 | 개발 fixture seed | 완료 | `ENABLE_DEMO_SEED=true`인 테스트/로컬 환경에서만 생성 |
| 백엔드 | 운영 모드 seed 분리 | 완료 | `ENABLE_DEMO_SEED=false`, `BOOTSTRAP_SERVICE_MASTER_*`, `ALLOW_EMPTY_OTP=false` 운영 설정 추가 |
| 백엔드 | SMTP mail-worker 실발송 | 완료 | mail queue pending을 SMTP로 발송하고 sent/failed 상태 갱신 |
| 백엔드 | 서비스 관리자 초기 운영 API | 완료 | 대회 생성, 대회 운영자 이메일 권한 배정 |
| 백엔드 | 운영자 접근 가능 대회 목록 API | 완료 | 공개 여부와 무관하게 operator scope 기준 조회 |
| 백엔드 | 대회 운영자 이메일 알림 | 완료 | 대회 생성/운영자 배정/대회 설정 변경 시 mail queue 적재 |
| 백엔드 | 수동 재채점 API 제거 | 완료 | 운영자/서비스 마스터 모두 재채점 불가 |
| 백엔드 | mail queue/worker 골격 | 완료 | SMTP 실제 발송은 env 필요 |
| 채점 | judge-agent backend client | 완료 | register/heartbeat/claim/report 연결 |
| 채점 | judge-agent executor | 완료 | C99/C++17/Python 3.13/Java 8 target compile/run, testcase stdout 비교 |
| 채점 | active testcase claim payload | 완료 | judge job claim에 active testcase set/testcases/download URL 포함 |
| 프론트 | Vite React 프로젝트 구조 | 완료 | `frontend/src` |
| 프론트 | 공개/참가자/관리자/운영자 화면 | 완료 | 실제 서비스 흐름 기준 라우팅 |
| 프론트 | 참가 유형 분리 UI | 완료 | 로그인 후 유형 자동 배정, 운영자 유형 필터 |
| 프론트 | 참가자 유형 접근 제한 | 완료 | 팀 로그인 시 본인 참가 유형 문제/스코어보드/제출 현황만 표시 |
| 프론트 | 비로그인 대회 리소스 차단 | 완료 | 대회 전/중 문제/스코어보드/제출 현황 접근 제한, 종료 후 운영자 공개 설정 UI |
| 프론트 | 운영자 대회 설정 화면 | 완료 | `/operator/contests/{contestId}/settings`, 일정/상태/참가 유형/프리즈/긴급공지/종료 후 공개 설정 |
| 프론트 | 운영자 내부 스코어보드 패널 | 완료 | 운영자 콘솔에서 유형별 최신 점수와 순위 확인 |
| 프론트 | 운영자 참가팀 관리 화면 | 완료 | `/operator/contests/{contestId}/participants`, 팀 등록/목록, 팀당 참가 유형 필수, 삭제 없음 정책 표시 |
| 프론트 | 참가팀 등록/수정 UX 보강 | 완료 | 팀장 필수, 팀원 다중 추가, 팀 목록 편집, CSV/TSV 일괄 등록 |
| 프론트 | 운영자 문제 관리 화면 | 완료 | `/operator/contests/{contestId}/problems`, 유형별 문제 목록/등록/리소스 관리 API 연결 |
| 프론트 | 문제 출제 편집기 고도화 | 완료 | Polygon식 섹션 편집, 입력/출력 설명, 예제 다중 관리, 노트, 본문 미리보기 |
| 프론트 | 문제 리소스/테스트케이스 관리 UI | 완료 | testlib.h/validator.cpp/checker.cpp 업로드, `.in/.out` 쌍 업로드, 검증 후 active set 생성 |
| 프론트 | 테스트케이스 파일 업로드 UI | 완료 | `.in/.out` 파일 쌍 업로드 후 storage key, sha256 자동 반영 |
| 프론트 | 출제 패키지 파일 관리 UI | 완료 | 문제별 testlib.h/validator.cpp/checker.cpp 중심 UI |
| 백엔드 | 검증 기반 테스트케이스 API | 완료 | `.in` validator 검증, `.out` checker self-check, active testcase set 생성 |
| 백엔드 | Polygon zip 가져오기 API | 완료 | zip 압축 해제, `problem.xml` 분석, 문제/예제/패키지 파일 저장, Test Script 생성, 즉시 package build |
| 도구 | Polygon export 변환 | 완료 | `tools/polygon_export_to_zoj.py`로 `problem.xml`/statement/tests/solutions/files를 ZOJ 업로드 구조로 변환 |
| 백엔드 | Polygon식 테스트 빌드 워커 | 필요 | 현재 MVP는 API 동기 실행, 장기 실행/격리는 별도 build worker/queue로 분리 필요 |
| 백엔드 | 출제 패키지 참가자 노출 차단 | 완료 | `package-files/*` 문제 리소스는 참가자 assets API에서 제외 |
| 프론트 | 로그아웃 UI | 완료 | 참가자 프로필 로그아웃, 운영자/관리자 세션 종료 동선 |
| 프론트 | 데모성 UI 제거 1차 | 완료 | 테스트 계정 prefill, 데모 fallback 문구, 임시 빠른 수정 버튼, 비로그인 가짜 제출 표시 제거 |
| 프론트 | staff 세션 토큰 연결 | 완료 | 관리자 로그인 결과를 localStorage에 저장하고 운영자/관리자 API 호출에 사용 |
| 프론트 | 일반 로그인 통합 플로우 | 완료 | 참가자/대회 운영자 공통 OTP 로그인, 내 대회 탭, 운영 페이지 진입 분리 |
| 프론트 | 관리자 로그인 에러 표시 | 완료 | 실패 시 에러 code/HTTP/request_id 노출, 저장된 staff 세션 검증 실패 시 재로그인 안내 |
| 프론트 | 서비스 관리자 대회 추가 화면 | 완료 | 오픈일, 공개상태, 주최기관, 운영자 이메일만 입력 |
| 프론트 | 운영자 전용 대회 선택 화면 | 완료 | `/operator`, 스케줄/비공개 대회도 운영자 권한으로 진입 |
| 프론트 | 실제같은 운영 UI 디자인 | 완료 | 데이터 밀도 높은 대시보드 |
| 프론트 | 백준 스타일 페이지 IA | 완료 | 메인, 대회목록, 참가/관리자 로그인, 대회홈, 문제집, 문제상세, 스코어보드, 게시판, 채점상태 |
| 프론트 | API 연결 기반 | 진행중 | 공개 API와 로그인 기반 운영자/관리자 API 연결 |
| 프론트 | 참가자 제출 API 연결 | 완료 | OTP 세션 토큰 기반 제출 생성, long polling 상태 확인 |
| 프론트 | 참가자 문제집 API 연결 | 완료 | 로그인 후 participant token으로 본인 참가 유형 문제 목록 로드 |
| 프론트 | 참가자 채점 현황 페이지 | 완료 | `/contests/{contestId}/submissions`, 로그인 시 자기 팀 제출 목록 API 연결 |
| 프론트 | 문제 상세 이동 동선 | 완료 | 참가 유형별 문제 목록만 표시, 문제 간 URL 이동, 제출 후 채점 현황 이동 |
| 프론트 | 서비스/대회 네비 분리 | 완료 | 대회 내부 전용 네비, 참가자 프로필/팀/참가 유형 표시 |
| 프론트 | 운영자/관리자 접근 네비 개선 | 완료 | staff 전용 상단바, 운영 홈/설정/참가팀/문제/관리자 홈 직접 이동 |
| 프론트 | URL 라우팅/뒤로가기 | 완료 | History API 기반 경로, 브라우저 뒤로가기, 직접 URL 접근 |
| 프론트 | 반응형 레이아웃 보강 | 완료 | 모바일 nav, 카드, 문제 상세, 표 overflow 대응 |
| 배포 | Compose 파일 | 완료 | backend/judge-agent 분리, migrate/minio-init 반영 |
| 배포 | 운영 env 초기화 | 완료 | `deploy/init-env.sh`, 실제 env gitignore, production env example 생성 |
| 배포 | JWT signing secret env | 완료 | `AUTH_TOKEN_SECRET`, `AUTH_TOKEN_ISSUER` env example 반영 |
| 배포 | 운영 compose 보강 | 완료 | API healthcheck, PostgreSQL 백업 profile, MinIO/SMTP/Postgres env 정리 |
| 배포 | judge-agent compose hardening | 완료 | docker.sock 제거, no-new-privileges, slot 자원 힌트 |
| 테스트 | 백엔드 API 테스트 | 완료 | `32 passed` |
| 테스트 | judge-agent executor 테스트 | 완료 | `5 passed` |
| 테스트 | 프론트 빌드 검증 | 완료 | Vite production build 성공 |

## 필요 결정/준비사항

| 항목 | 필요성 | 상태 |
| --- | --- | --- |
| SMTP 서버 정보 | 실제 메일 발송 | 필요 |
| 운영 도메인 | CORS, 메일 링크, Nginx 설정 | 필요 |
| MinIO access key/secret | 파일 저장소 연결 | 필요 |
| PostgreSQL 운영 비밀번호 | 운영 DB 연결 | 필요 |
| judge node별 secret | 내부 judge API 인증 | 필요 |
| 실제 대회별 참가 유형 이름/개수 | 대회 생성/운영 정책 | 대회별 결정 |

## 검증 로그

- `python3 -m compileall backend/app judge_agent/app judge_dispatcher/app`: 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 36 passed
- `PYTHONPATH=judge_agent backend/.venv/bin/python -m pytest judge_agent/tests`: 5 passed
- `PYTHONPATH=judge_agent INTERNAL_API_BASE_URL=http://127.0.0.1:8006/api JUDGE_RUN_ONCE=true backend/.venv/bin/python -m app.main`: register/claim/report 통과
- `DATABASE_URL=sqlite:////private/tmp/zerone_minio_presign_test.db backend/.venv/bin/alembic upgrade head`: 통과
- `DATABASE_URL=sqlite:////private/tmp/zerone_node_secret_test.db backend/.venv/bin/alembic upgrade head`: 통과
- `docker compose -f deploy/compose.backend.yaml config`: 통과
- `docker compose -f deploy/compose.judge-agent.yaml config`: 통과
- `npm --prefix frontend run build`: 통과
- `curl http://127.0.0.1:8000/api/public/home`: 200 OK
- `curl http://127.0.0.1:8001/api/public/contests/{contest_id}`: divisions 포함 200 OK
- `curl POST http://127.0.0.1:8001/api/contests/{contest_id}/participant-login/otp/verify`: division/workspace_path 포함 200 OK
- `curl http://127.0.0.1:8002/api/health`: DB 전환 최신 서버 200 OK
- `curl POST http://127.0.0.1:8003/api/auth/staff/login`: DB 세션 토큰 포함 200 OK
- `curl http://127.0.0.1:8004/api/admin/dashboard`: 토큰 없음 `401 authentication_required`
- `curl http://127.0.0.1:8005/api/public/contests`: 참가자 제출 인증 최신 서버 200 OK
- `curl http://127.0.0.1:8006/api/public/contests`: 문제/테스트케이스 메타데이터 최신 서버 200 OK
- `curl http://127.0.0.1:8007/api/public/contests`: active testcase claim 최신 서버 200 OK
- `curl http://127.0.0.1:8008/api/public/contests`: object storage presign 최신 서버 200 OK
- `curl http://127.0.0.1:8009/api/public/contests`: judge node secret 검증 최신 서버 200 OK
- `curl http://127.0.0.1:8010/api/public/contests`: 스코어보드 점수 정책 최신 서버 200 OK
- `curl http://127.0.0.1:8011/api/public/contests`: 이전 서버 200 OK
- `curl http://127.0.0.1:8012/api/public/contests`: 수동 재채점 API 제거 최신 서버 200 OK
- `curl http://127.0.0.1:8014/api/public/contests`: 참가자 제출 API 연결 최신 서버 200 OK
- `curl http://127.0.0.1:8014/api/contests/{contest_id}/problems`: 비로그인 문제 목록 접근 `404 not_found`
- `curl -I http://127.0.0.1:5173/`: 200 OK
- `npm --prefix frontend run build`: API 연결 기반 반영 후 통과
- `npm --prefix frontend run build`: 백준 스타일 페이지 IA 반영 후 통과
- `npm --prefix frontend run build`: 참가자 제출 API 연결 후 통과
- `npm --prefix frontend run build`: URL 라우팅/반응형 보강 후 통과
- `npm --prefix frontend run build`: 참가자 채점 현황 페이지 반영 후 통과
- `npm --prefix frontend run build`: 문제 상세 이동 동선 반영 후 통과
- `npm --prefix frontend run build`: 서비스/대회 네비 분리 후 통과
- `npm --prefix frontend run build`: 참가자 유형 접근 제한 반영 후 통과
- `npm --prefix frontend run build`: 비로그인 대회 리소스 차단/운영자 공개 설정 UI 반영 후 통과
- `npm --prefix frontend run build`: 운영자 대회 설정 화면 반영 후 통과
- `npm --prefix frontend run build`: 운영자 참가팀 관리 화면 반영 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 참가자/공개 리소스 접근 제한 반영 후 19 passed
- `backend/.venv/bin/python -m pytest backend/tests`: 운영자 대회 설정 수정 API 반영 후 20 passed
- `python3 -m compileall backend/app`: 운영자 대회 설정 수정 API 반영 후 통과
- `npm --prefix frontend run build`: 운영자 공개 설정 PATCH API 연결 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 운영자 참가팀 상태 변경 API 반영 후 21 passed
- `python3 -m compileall backend/app`: 운영자 참가팀 상태 변경 API 반영 후 통과
- `npm --prefix frontend run build`: 참가팀 상태 변경 버튼 연결 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 운영자 팀원 관리 API 반영 후 22 passed
- `python3 -m compileall backend/app`: 운영자 팀원 관리 API 반영 후 통과
- `npm --prefix frontend run build`: 팀원 추가/세션 종료 UI 연결 후 통과
- `npm --prefix frontend run build`: 운영자 문제 관리 화면/API 연결 후 통과
- `npm --prefix frontend run build`: 문제 출제 편집기 고도화 반영 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 운영자 문제 관리 화면 반영 후 22 passed
- `curl -I http://127.0.0.1:5173/operator/problems`: 200 OK
- `npm --prefix frontend run build`: 문제 리소스/테스트케이스 관리 UI 연결 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 문제 리소스/테스트케이스 관리 UI 반영 후 22 passed
- `backend/.venv/bin/python -m pytest backend/tests`: 출제 패키지 참가자 노출 차단 반영 후 30 passed
- `npm --prefix frontend run build`: 출제 패키지 파일 관리 UI 반영 후 통과
- `npm --prefix frontend run build`: 일반 로그인 통합 플로우 반영 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 일반 로그인 통합 플로우 반영 후 36 passed
- `backend/.venv/bin/python -m pytest backend/tests`: JWT 기반 세션 토큰 반영 후 36 passed
- `npm --prefix frontend run build`: JWT 기반 세션 토큰 반영 후 통과
- `curl http://127.0.0.1:8080/api/health`: JWT 기반 세션 토큰 Docker 재빌드 후 200 OK
- `npm --prefix frontend run build`: 운영자/관리자 접근 네비 개선 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 참가 유형 이름 중복/참가팀 bulk 등록 반영 후 31 passed
- `npm --prefix frontend run build`: 참가팀 다중 등록/수정/CSV 일괄 등록 UI 반영 후 통과
- `npm --prefix frontend run build`: Polygon식 테스트 recipe/빌드 UI 반영 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: Polygon식 package build API 반영 후 32 passed
- `python3 -m compileall backend/app`: Polygon식 package build API 반영 후 통과
- `curl http://127.0.0.1:8080/api/health`: Docker Compose API 재빌드 후 200 OK
- `backend/.venv/bin/python -m pytest backend/tests`: `test*.zoj.com` 데모 계정/OTP 빈값 허용 반영 후 22 passed
- `python3 -m compileall backend/app`: 데모 계정/OTP 빈값 허용 반영 후 통과
- `npm --prefix frontend run build`: 프론트 기본 데모 계정 `test*.zoj.com` 반영 후 통과
- `npm --prefix frontend run build`: 로그아웃 UI 반영 후 통과
- `npm --prefix frontend run build`: 데모성 UI 제거 1차 반영 후 통과
- `npm --prefix frontend run build`: staff 세션 토큰 연결 후 통과
- `npm --prefix frontend run build`: 참가자 문제집 API 연결 후 통과
- `npm --prefix frontend run build`: 관리자 로그인 에러 표시/세션 검증 후 통과
- `docker compose -f deploy/compose.backend.yaml config`: 운영 env/compose 보강 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 운영 seed/SMTP 설정 분리 후 22 passed
- `npm --prefix frontend run build`: `/api` production 기본값 반영 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 초기 운영 API 추가 후 23 passed
- `npm --prefix frontend run build`: 서비스 관리자 초기 운영 화면 추가 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 운영자 대회 목록/메일 알림/staff OTP 반영 후 27 passed
- `backend/.venv/bin/python -m pytest backend/tests`: OTP 재요청 쿨다운 반영 후 통과
- `python3 -m compileall backend/app`: 운영자 대회 목록/메일 알림 반영 후 통과
- `npm --prefix frontend run build`: 운영자 대회 목록 라우팅/스케줄 대회 접근 반영 후 통과
- `npm --prefix frontend run build`: 대회 추가 폼 단순화 후 통과
- `backend/.venv/bin/python -m pytest backend/tests`: 로그아웃 UI 반영 후 22 passed
- `curl -I http://127.0.0.1:5173/contests`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo/login`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo/submissions`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo/problems/p-d`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo/problems`: 200 OK
- `curl -I http://127.0.0.1:5173/contests/demo/scoreboard`: 200 OK
- `curl -I http://127.0.0.1:5173/operator/contests/demo/settings`: 200 OK
- `curl -I http://127.0.0.1:5173/operator/contests/demo/participants`: 200 OK
- `curl http://127.0.0.1:8014/api/operator/contests/{contest_id}/participants`: 운영자 토큰으로 200 OK

## 실행 중인 로컬 서버

- Backend API: `http://127.0.0.1:8000/api`
- Backend API with latest division changes: `http://127.0.0.1:8001/api`
- Backend API with latest DB changes: `http://127.0.0.1:8002/api`
- Backend API with latest auth/session changes: `http://127.0.0.1:8003/api`
- Backend API with latest authz changes: `http://127.0.0.1:8004/api`
- Backend API with latest participant submission auth changes: `http://127.0.0.1:8005/api`
- Backend API with latest problem metadata changes: `http://127.0.0.1:8006/api`
- Backend API with latest active testcase claim changes: `http://127.0.0.1:8007/api`
- Backend API with latest object storage presign changes: `http://127.0.0.1:8008/api`
- Backend API with latest judge node secret changes: `http://127.0.0.1:8009/api`
- Backend API with latest scoreboard scoring changes: `http://127.0.0.1:8010/api`
- Backend API with removed manual rejudge API: `http://127.0.0.1:8012/api`
- Backend API with latest participant submit flow: `http://127.0.0.1:8014/api`
- Frontend: `http://127.0.0.1:5173/`
