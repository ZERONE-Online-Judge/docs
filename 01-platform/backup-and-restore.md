# 백업과 복구 기준

이 문서는 초기 운영에서 사용할 백업/복구 기준을 정의한다.

## 백업 대상

필수:

- PostgreSQL database
- PostgreSQL WAL archive
- MinIO object data
- MinIO bucket metadata
- 배포 env secret 별도 보관본

백업 제외:

- Redis cache
- 일시적인 judge work directory
- Docker build cache

Redis에 장기 상태를 저장하게 되면 백업 대상에 추가한다.

## PostgreSQL 백업

기본 전략:

- 매일 1회 `pg_dump` full backup
- WAL archiving 활성화
- 백업 파일은 Backend VM과 다른 저장소에 보관
- 대회 당일에는 시작 전 수동 백업, 종료 후 수동 백업 추가

권장 보관:

| 백업 | 보관 기간 |
| --- | --- |
| 일일 full dump | 30일 |
| 주간 full dump | 6개월 |
| 대회 시작 전/종료 후 수동 백업 | 영구 또는 운영 정책상 장기 보관 |
| WAL archive | 최소 7일 |

복구 목표:

- 일반 장애는 최신 일일 dump 기준 복구
- 대회 당일 장애는 WAL archive 기반 point-in-time recovery를 목표로 한다.

## MinIO 백업

MinIO에는 아래 파일이 저장된다.

- 문제 이미지와 리소스
- 테스트케이스
- 제출 코드
- 채점 산출물

기본 전략:

- `mc mirror`로 별도 디스크 또는 백업 VM에 주기 복제
- 가능하면 MinIO bucket versioning을 활성화
- 대회 시작 전/종료 후 수동 mirror 실행

예시:

```bash
mc mirror --overwrite zerone/zerone backup/zerone
```

## env secret 보관

`.env` 원본은 git에 넣지 않는다.

보관 대상:

- `backend.env`
- `db.env`
- `minio.env`
- `judge-agent.env`

운영자는 암호화된 password manager 또는 별도 보안 저장소에 보관한다.
`*.env.example`은 실제 secret 없이 설정 스키마에서 자동 생성한다.

## 복구 리허설

대회 전 최소 1회는 아래를 리허설한다.

1. 새 VM 또는 임시 디렉터리에 PostgreSQL dump 복구
2. MinIO mirror 데이터 연결
3. FastAPI가 복구 DB와 MinIO object를 읽는지 확인
4. 샘플 제출의 source code와 testcase 접근 확인
5. judge-agent가 내부 API를 통해 테스트 job을 수행하는지 확인

복구 리허설을 하지 않은 백업은 실제 복구 가능성이 검증되지 않은 것으로 본다.

