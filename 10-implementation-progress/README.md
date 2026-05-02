# 구현 진행 상황

이 문서는 프론트엔드와 백엔드 구현 중 진행상황, 필요사항, 검증 상태를 추적한다.

## 범위

현재 목표는 실제 운영 제품의 뼈대를 따르는 MVP 데모다.

- 프론트엔드: 실제 서비스처럼 보이는 데모 UI
- 백엔드: FastAPI 기반 API 골격과 in-memory MVP 데이터 흐름
- 채점: dispatcher/agent 실제 샌드박스 전 단계의 API/상태 모델 골격
- 배포: Docker Compose 기준 파일과 env example 생성 기준

## 진행 체크리스트

| 영역 | 항목 | 상태 | 메모 |
| --- | --- | --- | --- |
| 문서 | 확정 정책 요약 반영 | 완료 | `docs/09-visual-summary/decided-policies.md` |
| 문서 | 구현 진행상황 문서 생성 | 완료 | 이 문서 |
| 백엔드 | FastAPI 프로젝트 구조 | 완료 | `backend/app` |
| 백엔드 | 공통 응답/에러 포맷 | 완료 | request_id 포함 |
| 백엔드 | 공개/참가자/관리자/운영자 API | 완료 | 데모 데이터 기반 |
| 백엔드 | 참가자 이메일 OTP 로그인 모델 | 완료 | mail queue 적재 포함 |
| 백엔드 | 대회 참가 유형 division 모델 | 완료 | 팀/문제/제출/스코어보드 유형 분리 |
| 백엔드 | 제출 생성 및 judge job 생성 | 완료 | in-memory transaction 유사 처리 |
| 백엔드 | long polling 제출 상태 API | 완료 | 데모 상태 갱신 |
| 백엔드 | 내부 judge API | 완료 | register/heartbeat/claim/result |
| 백엔드 | mail queue/worker 골격 | 완료 | SMTP 실제 발송은 env 필요 |
| 프론트 | Vite React 프로젝트 구조 | 완료 | `frontend/src` |
| 프론트 | 공개/참가자/관리자/운영자 화면 | 완료 | 데모 라우팅 |
| 프론트 | 참가 유형 분리 UI | 완료 | 로그인 후 유형 자동 배정, 운영자 유형 필터 |
| 프론트 | 실제같은 운영 UI 디자인 | 완료 | 데이터 밀도 높은 대시보드 |
| 배포 | Compose 파일 | 완료 | backend/judge-agent 분리 |
| 테스트 | 백엔드 API 테스트 | 완료 | `4 passed` |
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
- `backend/.venv/bin/python -m pytest tests`: 4 passed
- `npm --prefix frontend run build`: 통과
- `curl http://127.0.0.1:8000/api/public/home`: 200 OK
- `curl http://127.0.0.1:8001/api/public/contests/{contest_id}`: divisions 포함 200 OK
- `curl POST http://127.0.0.1:8001/api/contests/{contest_id}/participant-login/otp/verify`: division/workspace_path 포함 200 OK
- `curl -I http://127.0.0.1:5173/`: 200 OK

## 실행 중인 로컬 서버

- Backend API: `http://127.0.0.1:8000/api`
- Backend API with latest division changes: `http://127.0.0.1:8001/api`
- Frontend: `http://127.0.0.1:5173/`
