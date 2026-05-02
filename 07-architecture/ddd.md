# Zerone Online Judge DDD 초안

이 문서는 [통합 요구사항 문서](../99-archive/requirements-monolith.md)에서 분리한 DDD 초안이다.
대회 규정, 참가 정책, 팀 동시 접속 정책을 기준으로 현재 시점의 bounded context와 모델 초안을 정리한다.

## 1. DDD 적용 원칙

이 프로젝트는 단순 CRUD 백오피스가 아니라 아래 성격을 동시에 가진다.

- 전역 권한 관리 시스템
- 대회 운영 플랫폼
- 팀 인증 및 세션 제어 시스템
- 제출/랭킹/스코어보드 시스템
- 분리형 채점 인프라 제어 시스템

따라서 기능별 폴더 나열보다 `Bounded Context` 중심으로 나누는 것이 적절하다.

DDD 적용 방향:

- 각 컨텍스트는 자신의 모델과 규칙을 우선 가진다.
- API, DB, 외부 인프라보다 도메인 규칙을 먼저 설계한다.
- 컨텍스트 간 통신은 명시적인 application service 또는 event/message를 통해 수행한다.
- 공통 enum이나 헬퍼를 무분별하게 공유하지 않고, 꼭 필요한 계약만 공유한다.
- 특히 `채점`, `대회 운영`, `권한`, `팀 인증`은 모델을 섞지 않는다.

## 2. 주요 Bounded Context

현재 기준으로는 아래 컨텍스트로 나누는 것이 적절하다.

### 2.1 Identity & Access Context

- 서비스 마스터
- 서비스 매니저
- 대회 운영자
- 대회 매니저
- permission / scope
- 비밀번호 로그인
- 계정 초대 / 비밀번호 설정 / 비밀번호 재설정

핵심 책임:

- 권한자 계정 lifecycle 관리
- role / permission / scope 검사
- 계정 인증, 세션, 보안 정책

### 2.2 Contest Administration Context

- 대회 생성/수정/오픈/종료
- 개최 기관, 대회 개요, 공개 정책
- 운영자/매니저 배정
- 공지사항
- 질문 게시판
- 대회 규정집

핵심 책임:

- 대회 자체 lifecycle 관리
- 대회 운영 정보 관리
- 대회 단위 인력 배정
- 대회 규정과 참가 정책 관리

### 2.3 Team Participation Context

- 참가 팀 등록
- 팀명, 로그인 식별자, 상태
- OTP 로그인
- 동시 접속 / 세션 전환 정책
- 팀 세션 제어

핵심 책임:

- 팀 참가 자격과 인증
- 팀 상태와 로그인 세션 관리
- 같은 대회 내 역할 충돌 방지

### 2.4 Problem Management Context

- 문제 본문
- 제한값
- 예제 입출력
- 문제 리소스
- 테스트케이스
- 테스트케이스 제너레이터
- 문제 공개 범위

핵심 책임:

- 문제와 채점 데이터 관리
- 대회 중 수정 금지 같은 운영 규칙 보장

### 2.5 Submission & Scoreboard Context

- 제출 생성
- 제출 상태
- 제출 조회
- 팀별 풀이 현황
- 스코어보드
- 페널티 계산
- 점수 계산
- 프리즈 / 최종 공개

핵심 책임:

- 제출 lifecycle
- 대회 채점 제도별 랭킹 계산
- AC/WA 대회의 페널티, 점수제 대회의 총점/동점 처리, 프리즈 정책 보장

### 2.6 Judge Orchestration Context

- judge job
- 전역 FIFO
- dispatcher
- judge node registry
- heartbeat
- lease
- 재시도 / 재할당
- cluster policy

핵심 책임:

- 채점 작업 분배
- Tailscale 기반 judge node 관리
- 장애 전환과 재할당

### 2.7 Audit & Monitoring Context

- 감사 로그
- 운영 대시보드
- 공개 채점 서버 현황
- 내부 모니터링 지표

핵심 책임:

- 운영 추적성
- 상태 가시화
- 변경 이력 보관

### 2.8 Service Communication Context

- 서비스 공지
- 서비스 공지 게시판
- 전역 긴급 공지
- 서비스 점검/장애/복구 안내
- 정책 변경 안내

핵심 책임:

- 특정 대회에 속하지 않는 전역 운영 공지 관리
- 공개 공지와 운영자 전용 공지 구분
- 전역 긴급 공지 노출 정책 관리
- 일반 이용자 게시판 기능을 제공하지 않는 공지 전용 모델 유지

### 2.9 Content & Asset Policy Context

- Markdown 기반 작성 콘텐츠
- LaTeX 입력 / KaTeX 렌더링
- HTML 직접 입력 금지
- XSS 방지용 sanitizer 정책
- 컨텍스트별 이미지 asset 업로드와 내부 참조

핵심 책임:

- 문제 본문, 공지, 질문/답변, 메일 템플릿 등 작성 콘텐츠의 공통 렌더링 정책 관리
- 업로드 파일의 공통 storage key, MIME 검증, 보안 정책 관리
- 실제 파일 경로 대신 내부 asset reference 기반 렌더링 보장
- asset 소유 모델은 `ProblemAsset`, `ServiceNoticeAsset`, `ContestAnnouncementAsset`, `ClarificationAsset`처럼 컨텍스트별로 분리한다.

### 2.10 Mail Communication Context

- 권한자 초대 메일
- 비밀번호 설정/재설정 메일
- 참가자 OTP 메일
- 메일 템플릿
- 테스트 발송

핵심 책임:

- 메일 템플릿 lifecycle 관리
- 템플릿 변수와 안전한 본문 렌더링 관리
- OTP, reset token 같은 비밀값 원문이 로그에 남지 않도록 보장

## 3. Context 간 관계 원칙

- `Contest Administration`은 대회 메타데이터의 소유자다.
- `Problem Management`는 대회에 연결되지만 문제/테스트케이스 규칙의 소유자다.
- `Team Participation`은 참가 자격과 팀 세션의 소유자다.
- `Submission & Scoreboard`는 제출과 랭킹 계산의 소유자다.
- `Judge Orchestration`은 실제 채점 분산과 실행 흐름의 소유자다.
- `Identity & Access`는 권한자 인증과 permission 판정의 소유자다.
- `Audit & Monitoring`은 다른 컨텍스트의 이벤트를 받아 기록하고 시각화한다.
- `Service Communication`은 특정 대회에 속하지 않는 서비스 공지 게시판의 소유자다.
- `Content & Asset Policy`는 여러 컨텍스트가 사용하는 작성 콘텐츠와 asset 보안 정책을 제공한다.
- `Mail Communication`은 권한자 초대, 비밀번호 재설정, 참가자 OTP 같은 메일 템플릿을 소유한다.

원칙적으로:

- 대회 생성/수정은 `Contest Administration`
- 팀 로그인/OTP는 `Team Participation`
- 제출 생성은 `Submission & Scoreboard`
- judge job 생성 이후 분산은 `Judge Orchestration`
- 권한 체크는 `Identity & Access`
- 작성 콘텐츠 sanitizer와 asset 참조 검증은 `Content & Asset Policy`
- 메일 템플릿 관리는 `Mail Communication`

## 4. Aggregate 초안

현재 단계에서 너무 세밀하게 고정할 필요는 없지만, 대략 아래 aggregate 단위가 적절하다.

- `StaffAccount`
- `ServiceManagerGrant`
- `Contest`
- `ContestStaffAssignment`
- `ParticipantTeam`
- `Problem`
- `Submission`
- `ScoreboardSnapshot` 또는 `ContestScoreboard`
- `JudgeJob`
- `JudgeNode`
- `AuditLogEntry`
- `ServiceNotice`
- `RichTextContent`
- `ContentAssetPolicy`
- `MailTemplate`

주의:

- `Submission`과 `JudgeJob`은 분리한다.
- `Contest`와 `Problem`도 분리한다.
- `ParticipantTeam`의 세션/OTP 관련 규칙은 `Team Participation`이 소유한다.

## 5. Context별 모델 초안

### 5.1 Identity & Access Context

`Aggregate`

- `StaffAccount`
- `ServiceManagerGrant`

`Entity`

- `StaffAccount`
- `PasswordCredential`
- `StaffInvitation`
- `PasswordResetToken`
- `PermissionGrant`

`Value Object`

- `EmailAddress`
- `PasswordHash`
- `PermissionCode`
- `PermissionScope`
- `InvitationStatus`
- `AccountStatus`

`Domain Service`

- `PermissionEvaluationService`
- `ScopeResolutionService`
- `StaffAccountProvisioningService`
- `LastServiceMasterProtectionService`

설명:

- 권한자 계정과 권한 부여는 이 컨텍스트가 소유한다.
- 서비스 매니저의 전역/대회 범위 permission 계산도 여기서 수행한다.
- 대회 운영자/매니저 assignment 자체는 대회 컨텍스트와 연결되지만, permission 해석 규칙은 여기서 담당한다.
- 마지막 서비스 마스터 계정 삭제 또는 비활성화는 금지한다.

### 5.2 Contest Administration Context

`Aggregate`

- `Contest`
- `ContestStaffAssignment`

`Entity`

- `Contest`
- `ContestAnnouncement`
- `ContestAnnouncementAsset`
- `EmergencyAnnouncement`
- `ClarificationThread`
- `ClarificationAsset`
- `ClarificationReply`
- `ContestRuleDocument`
- `ContestSchedule`
- `ContestFreezePolicy`
- `ContestVisibilityPolicy`
- `ContestStaffAssignment`

`Value Object`

- `ContestStatus`
- `ContestPeriod`
- `FreezeAt`
- `ContestScoringPolicy`
- `ScoreCalculationPolicy`
- `TieBreakPolicy`
- `OrganizationName`
- `ContestOverview`
- `AnnouncementType`
- `AnnouncementVisibility`
- `EmergencyBannerSummary`
- `ClarificationStatus`
- `ClarificationVisibility`

`Domain Service`

- `ContestPublicationService`
- `ContestFreezePolicyService`
- `ContestStaffAssignmentPolicyService`

설명:

- 대회의 상태 전이, 공개/종료, 운영 인력 배정 규칙은 이 컨텍스트가 소유한다.
- 대회별 채점 제도, 점수 계산 정책, 동점 처리 정책도 이 컨텍스트가 소유한다.
- 공지와 질문 게시판은 대회 운영 메타데이터의 일부로 본다.
- 긴급 공지는 원문 공지와 배너 요약을 분리하고, 상세 공지로 이동할 수 있어야 한다.
- 질문 게시판은 질문 상태, 공개 범위, 답변 lifecycle을 대회 단위로 관리한다.

### 5.3 Team Participation Context

`Aggregate`

- `ParticipantTeam`
- `TeamMember`
- `TeamSession`

`Entity`

- `ParticipantTeam`
- `TeamMember`
- `OtpChallenge`
- `TeamSession`

`Value Object`

- `TeamName`
- `TeamMemberName`
- `TeamMemberEmail`
- `MaxConcurrentSessions`
- `TeamStatus`
- `OtpCode`
- `OtpHash`
- `OtpExpiresAt`
- `OtpAttemptLimit`
- `OtpRateLimitKey`
- `SessionId`
- `DeviceFingerprint`
- `LastLoginAt`

`Domain Service`

- `TeamEligibilityService`
- `OtpIssuanceService`
- `OtpVerificationService`
- `ParticipantTeamStatusTransitionService`
- `SessionConcurrencyPolicyService`

설명:

- 팀 등록, 참가자 이메일, OTP, 이메일별 세션과 동시 접속 정책은 이 컨텍스트가 소유한다.
- 같은 대회에서 운영자/매니저가 참가 팀이 될 수 없는 규칙도 여기서 강제한다.
- `invited` 팀은 최초 로그인 성공 시 `active`로 전환한다.
- `disqualified`, `disabled` 팀은 문제 조회와 제출이 불가능하다.
- OTP 원문은 저장하지 않고 해시와 만료/시도 제한 정보만 저장한다.

### 5.4 Problem Management Context

`Aggregate`

- `Problem`

`Entity`

- `Problem`
- `ProblemStatement`
- `ProblemConstraint`
- `ProblemExample`
- `ProblemResource`
- `ProblemAsset`
- `ProblemTestcase`
- `ProblemGenerator`
- `ProblemVisibilityPolicy`

`Value Object`

- `ProblemCode`
- `ProblemTitle`
- `TimeLimit`
- `MemoryLimit`
- `InputDescription`
- `OutputDescription`
- `ResourceFileMeta`
- `AssetId`
- `StorageKey`
- `AssetReference`
- `MimeType`
- `Sha256Digest`
- `ProblemOrder`
- `ProblemScoringPolicy`
- `SubtaskDefinition`
- `ProblemMaxScore`

`Domain Service`

- `ProblemPublicationPolicyService`
- `ProblemEditingPolicyService`
- `ProblemAssetValidationService`

설명:

- 문제 본문, 예제, 리소스, 테스트케이스, 공개 범위, 대회 중 수정 금지 정책은 이 컨텍스트가 소유한다.
- 점수제 대회에서 문제별 서브태스크 구성과 배점 규칙도 이 컨텍스트가 소유한다.
- 한 문제에 대한 운영 규칙은 가능하면 `Problem` aggregate 안에 가깝게 둔다.
- 문제 이미지는 사용자 파일명 대신 내부 `asset_id`와 `storage_key`로 관리한다.
- 문제 본문은 실제 파일 경로가 아니라 내부 asset reference를 통해 이미지를 참조한다.
- 대회 진행 중에는 문제 본문, 테스트케이스, 배점/채점 제도 변경을 금지한다.

### 5.5 Submission & Scoreboard Context

`Aggregate`

- `Submission`
- `ContestScoreboard`

`Entity`

- `Submission`
- `SubmissionStatusHistory`
- `TeamProblemProgress`
- `ContestScoreboard`
- `ScoreboardEntry`
- `ProblemPenaltyRecord`
- `ProblemScoreRecord`
- `ManualScoreAdjustment`

`Value Object`

- `SubmissionId`
- `SourceCode`
- `Language`
- `SubmittedAt`
- `JudgementStatus`
- `AcceptedAt`
- `PenaltyTime`
- `TotalScore`
- `ScoreAchievedAt`
- `ScoreAdjustmentReason`
- `AdjustedScore`
- `ScoreBeforeAdjustment`
- `ScoreAfterAdjustment`
- `Rank`

`Domain Service`

- `SubmissionCreationService`
- `PenaltyCalculationService`
- `ScoreboardRankingService`
- `FreezeProjectionService`
- `ManualScoreAdjustmentService`

설명:

- 제출 생성, 상태 전이, 팀별 풀이 상태, 대회별 랭킹/페널티/프리즈는 이 컨텍스트가 소유한다.
- AC/WA 대회와 점수제 대회의 스코어보드 계산 규칙은 동일 aggregate 안에서도 정책으로 분리되어야 한다.
- AC/WA 대회에서는 `submitted_at` 기준 페널티 계산 규칙이 핵심 invariant다.
- 점수제 대회에서는 `총점`과 `현재 총점을 처음 달성한 시각`이 핵심 정렬 값이다.
- 대회 종료 후 수동 점수 조정은 제출 원본 판정과 분리된 `ManualScoreAdjustment`로 관리한다.
- 수동 점수 조정은 `ended` 이후에만 허용하고, `finalized` 이후에는 기본적으로 금지한다.
- 수동 점수 조정은 변경 전/후 점수, 사유, 작업자를 감사 가능하게 남겨야 한다.

### 5.6 Judge Orchestration Context

`Aggregate`

- `JudgeJob`
- `JudgeNode`

`Entity`

- `JudgeJob`
- `JobLease`
- `JudgeNode`
- `NodeHeartbeat`
- `ClusterPolicy`
- `RetryRecord`

`Value Object`

- `JudgeJobId`
- `LeaseToken`
- `NodeId`
- `TailscaleNodeIdentity`
- `NodeStatus`
- `SlotCount`
- `HeartbeatAt`
- `QueuePosition`

`Domain Service`

- `FifoDispatchService`
- `LeaseManagementService`
- `NodeAvailabilityService`
- `RetryReassignmentService`

설명:

- 이 컨텍스트는 채점 작업을 실제로 어디에 보낼지 결정한다.
- `Submission` 결과 자체가 아니라 `JudgeJob`의 분배와 재시도가 주된 관심사다.

### 5.7 Audit & Monitoring Context

`Aggregate`

- `AuditLogEntry`
- `JudgePublicSnapshot`

`Entity`

- `AuditLogEntry`
- `AuditActor`
- `AuditTarget`
- `JudgePublicSnapshot`
- `OperationsDashboardSnapshot`

`Value Object`

- `AuditAction`
- `AuditSummary`
- `ChangedField`
- `PublicJudgeNodeStatus`
- `SnapshotAt`

`Domain Service`

- `AuditRecordingService`
- `JudgePublicProjectionService`
- `OperationsDashboardProjectionService`

설명:

- 감사 로그는 단순 문자열 저장이 아니라 `누가 / 무엇을 / 언제 / 어떻게 바꿨는지`를 구조적으로 보관한다.
- 공개 채점 서버 현황과 운영 대시보드는 읽기 모델 projection으로 보는 것이 적절하다.

### 5.8 Service Communication Context

`Aggregate`

- `ServiceNotice`

`Entity`

- `ServiceNotice`
- `ServiceNoticeAsset`
- `GlobalEmergencyNotice`
- `ServiceNoticePublicationWindow`

`Value Object`

- `ServiceNoticeStatus`
- `ServiceNoticeType`
- `NoticeTitle`
- `NoticeSummary`
- `NoticeBody`
- `NoticeVisibility`
- `PinnedNotice`

`Domain Service`

- `ServiceNoticePublicationService`
- `GlobalEmergencyNoticeService`

설명:

- 서비스 공지는 특정 대회에 속하지 않는 전역 공지다.
- 긴급 공지 배너는 요약만 노출하고 상세 페이지 링크를 제공한다.
- 숨김, 비공개, 예약 대기 공지는 공개 API에서 존재 여부를 드러내지 않는다.

### 5.9 Content & Asset Policy Context

`Aggregate`

- `RichTextContentPolicy`

`Entity`

- `AssetUploadRecord`
- `RenderedContentCache`

`Value Object`

- `AssetId`
- `OriginalFilename`
- `StorageKey`
- `AssetOwner`
- `AssetPurpose`
- `MimeType`
- `FileSize`
- `Sha256Digest`
- `RichTextContent`
- `MarkdownSubsetPolicy`
- `LatexExpression`
- `KatexRenderPolicy`
- `SanitizedHtml`
- `AssetReference`

`Domain Service`

- `AssetStorageService`
- `AssetValidationService`
- `RichTextSanitizationService`
- `AssetReferenceResolver`

설명:

- HTML 직접 입력은 허용하지 않고 Markdown subset을 안전하게 렌더링한다.
- 수식은 LaTeX 입력과 KaTeX 렌더링 정책으로 다룬다.
- 이미지 참조는 실제 파일 경로가 아니라 내부 asset reference를 기준으로 해석한다.
- 실제 asset metadata와 lifecycle은 각 컨텍스트의 asset 모델이 소유한다.

### 5.10 Mail Communication Context

`Aggregate`

- `MailTemplate`

`Entity`

- `MailTemplate`
- `MailTemplateRevision`
- `MailTemplateTestSend`

`Value Object`

- `MailTemplateKey`
- `MailSubjectTemplate`
- `MailBodyTemplate`
- `MailTemplateVariable`
- `MailTemplateStatus`
- `MailRenderPreview`

`Domain Service`

- `MailTemplateRenderService`
- `MailTemplateValidationService`
- `MailTestSendService`

설명:

- 초대, 비밀번호 설정/재설정, 참가자 OTP 메일의 템플릿을 관리한다.
- 템플릿 key는 코드에서 참조하므로 운영자가 변경할 수 없게 한다.
- OTP 원문, reset token 원문 같은 비밀값은 로그와 감사 로그에 남기지 않는다.

## 6. Context 간 주요 이벤트 초안

컨텍스트를 나눴기 때문에, 주요 상태 변화는 이벤트 또는 명시적 메시지로 전달하는 것이 적절하다.

후보 이벤트:

- `ContestCreated`
- `ContestOpened`
- `ContestClosed`
- `ParticipantTeamRegistered`
- `ParticipantTeamActivated`
- `ParticipantTeamDisqualified`
- `ParticipantTeamDisabled`
- `OtpIssued`
- `OtpVerified`
- `OtpBlocked`
- `ProblemPublished`
- `ProblemAssetUploaded`
- `ProblemAssetDeleted`
- `SubmissionCreated`
- `JudgeJobEnqueued`
- `JudgeJobLeased`
- `SubmissionJudged`
- `ScoreboardRecalculated`
- `ManualScoreAdjusted`
- `EmergencyAnnouncementPublished`
- `ContestNoticePublished`
- `ClarificationQuestionCreated`
- `ClarificationAnswered`
- `ServiceNoticePublished`
- `MailTemplateUpdated`
- `ManagerPermissionsChanged`
- `JudgeNodeBecameUnavailable`

원칙:

- `SubmissionCreated` 이후 `JudgeJobEnqueued`는 분리된 단계로 본다.
- `SubmissionJudged`는 `Submission & Scoreboard`와 `Audit & Monitoring` 양쪽에 영향을 줄 수 있다.
- 감사 로그는 가능한 한 각 컨텍스트의 중요한 상태 변화를 이벤트 기반으로 수집한다.

## 7. Aggregate별 핵심 필드 초안

현재 단계에서는 DB 컬럼을 확정하는 것이 아니라, 각 aggregate가 어떤 상태를 책임지는지 드러나는 수준으로 필드를 잡는다.

### 7.1 `StaffAccount`

핵심 필드:

- `staff_account_id`
- `email`
- `display_name`
- `account_status`
- `password_hash`
- `invitation_status`
- `invited_at`
- `activated_at`
- `disabled_at`
- `last_login_at`
- `password_changed_at`
- `created_at`
- `updated_at`

설명:

- 권한자 로그인 자체를 담당하는 루트 aggregate다.
- 서비스 마스터, 서비스 매니저, 대회 운영자, 대회 매니저 공통 계정 기반으로 본다.
- 초대 상태와 계정 활성/비활성 상태를 분리해서 관리한다.
- 마지막 서비스 마스터 계정은 삭제 또는 비활성화할 수 없다.

### 7.2 `ServiceManagerGrant`

핵심 필드:

- `service_manager_grant_id`
- `staff_account_id`
- `permission_code`
- `scope_type`
- `scope_contest_ids`
- `granted_by_staff_account_id`
- `granted_at`
- `revoked_at`
- `grant_status`

설명:

- 서비스 매니저 권한은 계정 자체가 아니라 grant 묶음으로 보는 것이 적절하다.
- 전역 권한과 특정 대회 범위 권한을 같은 모델에서 처리한다.

### 7.3 `Contest`

핵심 필드:

- `contest_id`
- `title`
- `organization_name`
- `overview`
- `status`
- `open_at`
- `start_at`
- `end_at`
- `freeze_at`
- `scoring_policy_type`
- `score_calculation_policy`
- `duplicate_rank_allowed`
- `tie_break_policy`
- `problem_public_after_end`
- `scoreboard_public_after_end`
- `submission_public_after_end`
- `final_scoreboard_published_at`
- `created_by_staff_account_id`
- `created_at`
- `updated_at`

설명:

- 대회 lifecycle과 공개 정책의 루트다.
- 채점 제도, 점수 계산 정책, 동점 처리 규칙, 중복 순위 허용 여부도 이 aggregate가 가진다.
- 스코어보드 최종 공개 여부는 Contest가 소유하는 것이 자연스럽다.

### 7.3.1 `ScoreCalculationPolicy`

핵심 필드:

- `score_calculation_policy_id`
- `contest_id`
- `scoring_mode`
- `score_apply_timing`
- `score_sorting_rule`
- `tie_break_primary`
- `tie_break_secondary`
- `duplicate_rank_allowed`
- `subtask_allowed`
- `problem_max_score`
- `problem_score_sum_validation`
- `created_at`
- `updated_at`

설명:

- 대회별 점수 계산 방법을 표현한다.
- 기본값은 점수제 대회에서 총점 내림차순, 동점 시 `submitted_at` 기준이다.
- 같은 총점과 같은 `submitted_at`까지 발생하면 중복 순위를 허용한다.
- 대회 관리자가 기본값을 override할 수 있으므로 Contest의 단순 enum이 아니라 별도 정책 값으로 다룬다.

### 7.3.2 `ContestAnnouncement`

핵심 필드:

- `contest_announcement_id`
- `contest_id`
- `title`
- `body_content_id`
- `banner_summary`
- `announcement_type`
- `announcement_status`
- `visibility`
- `emergency`
- `pinned`
- `published_at`
- `created_by_staff_account_id`
- `updated_at`
- `deleted_at`
- `asset_count`

설명:

- 대회 단위 공지사항과 긴급 공지를 표현한다.
- 긴급 공지 배너는 요약 노출용이고, 공지 상세가 원문 기준이다.
- 긴급 공지는 항상 공지 상세 또는 목록으로 이동할 수 있어야 한다.
- 첨부 이미지는 `ContestAnnouncementAsset`으로 분리해 관리한다.

### 7.3.3 `ClarificationThread`

핵심 필드:

- `clarification_thread_id`
- `contest_id`
- `participant_team_id`
- `problem_id`
- `title`
- `question_content_id`
- `asset_count`
- `thread_status`
- `visibility`
- `visible_title_for_others`
- `visible_body_for_others`
- `created_at`
- `updated_at`

설명:

- 대회 질문 게시판의 질문 단위다.
- 문제 관련 질문은 문제 식별자와 연결할 수 있다.
- 참가 팀은 질문 작성 시 공개 또는 비공개를 선택할 수 있다.
- 공개 질문은 제목과 내용이 모든 사용자에게 보인다.
- 비공개 질문은 제목과 내용이 모두 비공개로 표시되고, 작성 팀 이외의 참가자는 상세에 접근할 수 없다.
- 대회 운영자와 권한 있는 대회 운영매니저는 공개/비공개 질문을 모두 볼 수 있다.
- 질문 첨부 이미지는 `ClarificationAsset`으로 분리해 관리한다.

### 7.3.4 `ClarificationReply`

핵심 필드:

- `clarification_reply_id`
- `clarification_thread_id`
- `answer_content_id`
- `visibility`
- `answered_by_staff_account_id`
- `created_at`
- `updated_at`
- `deleted_at`

설명:

- 대회 운영자 또는 권한 있는 대회 운영매니저의 답변을 표현한다.
- 댓글 작성자는 댓글마다 공개 또는 비공개를 선택할 수 있다.
- 공개 댓글은 질문 접근 권한이 있는 사용자에게 보이고, 비공개 댓글은 댓글 작성자, 질문 작성 팀, 대회 운영자와 권한 있는 대회 운영매니저만 볼 수 있다.
- 답변 작성, 수정, 삭제는 권한과 감사 로그 대상이다.

### 7.4 `ContestStaffAssignment`

핵심 필드:

- `contest_staff_assignment_id`
- `contest_id`
- `staff_account_id`
- `assignment_role`
- `permission_overrides`
- `assigned_by_staff_account_id`
- `assigned_at`
- `removed_at`
- `assignment_status`

설명:

- 대회 운영자와 대회 매니저의 대회 단위 배정을 표현한다.
- 운영자는 full access, 매니저는 permission set 기반으로 해석한다.

### 7.5 `ParticipantTeam`

핵심 필드:

- `participant_team_id`
- `contest_id`
- `team_name`
- `team_status`
- `team_size`
- `first_login_at`
- `last_login_at`
- `last_otp_verified_at`
- `disqualified_at`
- `disqualified_reason`
- `created_by_staff_account_id`
- `created_at`
- `updated_at`

설명:

- 팀 등록과 운영 상태를 소유한다.
- 팀장/팀원 정보와 이메일별 로그인 세션은 `TeamMember`, `TeamSession`으로 분리한다.
- `invited`는 초대 또는 등록 후 아직 최초 로그인하지 않은 상태다.
- 최초 로그인 성공 시 `active`로 전환한다.
- `disqualified`, `disabled`는 문제 조회와 제출을 막는다.

### 7.6 `TeamMember`

핵심 필드:

- `team_member_id`
- `contest_id`
- `participant_team_id`
- `member_role`
- `member_name`
- `email`
- `member_status`
- `max_concurrent_sessions`
- `otp_rate_limit_key`
- `last_login_at`
- `created_at`
- `updated_at`

설명:

- 대회 운영진이 등록한 팀장/팀원 정보를 표현한다.
- 참가자 로그인 식별자와 OTP 수신지는 `email`이다.
- 세션과 동시 접속 제한은 팀 단위가 아니라 `TeamMember.email` 단위로 관리한다.

### 7.6.1 `OtpChallenge`

핵심 필드:

- `otp_challenge_id`
- `team_member_id`
- `otp_hash`
- `expires_at`
- `attempt_count`
- `max_attempt_count`
- `request_rate_limit_key`
- `ip_rate_limit_key`
- `challenge_status`
- `issued_at`
- `verified_at`
- `blocked_at`

설명:

- 참가자 OTP 인증의 발급, 검증, 만료, 차단 상태를 표현한다.
- OTP 원문은 저장하지 않고 해시만 저장한다.
- 발송/성공/실패/만료/차단 이벤트는 감사 또는 보안 추적이 가능해야 한다.

### 7.7 `TeamSession`

핵심 필드:

- `team_session_id`
- `participant_team_id`
- `team_member_id`
- `session_status`
- `device_fingerprint`
- `ip_address`
- `user_agent`
- `issued_at`
- `expires_at`
- `last_seen_at`
- `replaced_by_session_id`

설명:

- 참가자 이메일별 활성 세션을 표현한다.
- 같은 브라우저의 여러 탭 허용은 session 단위로 처리한다.

### 7.8 `Problem`

핵심 필드:

- `problem_id`
- `contest_id`
- `problem_code`
- `title`
- `statement`
- `statement_content_id`
- `input_description`
- `output_description`
- `constraints_text`
- `time_limit_ms`
- `memory_limit_mb`
- `display_order`
- `scoring_policy_type`
- `max_score`
- `subtask_count`
- `visibility_policy`
- `resource_count`
- `testcase_count`
- `generator_count`
- `problem_status`
- `created_at`
- `updated_at`

설명:

- 문제 본문과 운영 규칙의 루트다.
- 점수제 대회에서는 문제별 점수 배분 규칙도 Problem이 소유한다.
- 대회 중 수정 금지 여부도 problem editing policy와 함께 관리한다.
- 문제 본문은 `RichTextContent`로 관리하고, 이미지 참조는 내부 asset reference를 사용한다.

### 7.8.1 `ProblemAsset`

핵심 필드:

- `asset_id`
- `contest_id`
- `problem_id`
- `original_filename`
- `storage_key`
- `mime_type`
- `file_size`
- `sha256`
- `uploaded_by_staff_account_id`
- `asset_status`
- `created_at`
- `deleted_at`

설명:

- 문제 본문 이미지 asset을 표현한다.
- 사용자 파일명은 표시용 메타데이터로만 보관하고 실제 조회 키로 사용하지 않는다.
- 본문에서는 실제 파일 경로 대신 `asset_id` 또는 내부 asset reference로 참조한다.

### 7.9 `Submission`

핵심 필드:

- `submission_id`
- `contest_id`
- `problem_id`
- `participant_team_id`
- `language`
- `source_code`
- `submitted_at`
- `current_status`
- `status_updated_at`
- `compile_message`
- `runtime_summary`
- `awarded_score`
- `score_applied_at`
- `score_achieved_submission_at`
- `is_finalized`
- `final_result_at`

설명:

- 제출 자체와 사용자에게 보이는 판정 상태를 소유한다.
- AC/WA 대회에서는 페널티 기준 시간이 `submitted_at`이다.
- 점수제 대회에서는 각 제출이 만든 점수 증가량, 점수 반영 시각, 동점 처리 기준 제출 시각을 함께 관리해야 한다.

### 7.10 `ContestScoreboard`

핵심 필드:

- `contest_scoreboard_id`
- `contest_id`
- `freeze_started_at`
- `is_frozen`
- `is_final_published`
- `last_recalculated_at`
- `published_snapshot_version`
- `manual_adjustment_count`

하위 entry 수준 핵심 필드:

- `participant_team_id`
- `total_score`
- `score_achieved_submission_at`
- `solved_count`
- `total_penalty`
- `last_accepted_at`
- `current_rank`
- `problem_progress_map`

설명:

- 별도 projection 또는 aggregate 형태 모두 가능하지만, 대회 단위 랭킹 규칙의 소유자로 본다.
- 점수제 대회에서는 `총점`, `문제별 점수`, `총점 달성 제출 시각`이 핵심 projection 필드다.
- 프리즈 중 공개용 view와 내부 운영용 view는 projection을 분리할 수 있다.

### 7.10.1 `ManualScoreAdjustment`

핵심 필드:

- `manual_score_adjustment_id`
- `contest_id`
- `participant_team_id`
- `problem_id`
- `before_score`
- `after_score`
- `adjustment_reason`
- `adjusted_by_staff_account_id`
- `adjusted_at`
- `applied_to_internal_scoreboard_at`
- `included_in_final_scoreboard`
- `reverted_at`

설명:

- 대회 종료 후 팀/문제별 점수 개별 조정을 표현한다.
- 제출 원본 판정과 분리해서 관리한다.
- 변경 전/후 점수, 사유, 작업자를 반드시 보존한다.
- 최종 스코어보드 반영 여부는 별도 확정 절차로 관리한다.

### 7.11 `JudgeJob`

핵심 필드:

- `judge_job_id`
- `submission_id`
- `contest_id`
- `problem_id`
- `participant_team_id`
- `job_status`
- `queue_position`
- `lease_token`
- `leased_node_id`
- `leased_at`
- `lease_expires_at`
- `retry_count`
- `last_error_code`
- `created_at`
- `updated_at`

설명:

- 제출과 분리된 채점 분산 단위다.
- 중복 실행 방지와 재할당 제어를 위해 lease 관련 필드를 가진다.

### 7.12 `JudgeNode`

핵심 필드:

- `judge_node_id`
- `node_name`
- `tailscale_node_identity`
- `tailscale_ip`
- `node_status`
- `total_slots`
- `free_slots`
- `running_job_count`
- `region`
- `host_group`
- `last_heartbeat_at`
- `last_seen_version`
- `schedulable`
- `registered_at`

설명:

- 채점 서버의 가용성과 할당 가능 여부를 표현한다.
- Tailscale 연결 상태와 scheduler 상태를 함께 본다.

### 7.13 `AuditLogEntry`

핵심 필드:

- `audit_log_entry_id`
- `actor_type`
- `actor_id`
- `action_code`
- `target_type`
- `target_id`
- `contest_id`
- `summary`
- `before_snapshot`
- `after_snapshot`
- `occurred_at`
- `ip_address`

설명:

- 민감 작업 추적을 위해 before/after 요약 또는 snapshot을 남긴다.
- contest_id를 nullable로 두면 전역 이벤트와 대회 이벤트를 함께 표현할 수 있다.

### 7.14 `JudgePublicSnapshot`

핵심 필드:

- `judge_public_snapshot_id`
- `snapshot_at`
- `active_node_count`
- `scaling_node_count`
- `total_running_jobs`
- `total_queue_depth`
- `allocation_policy`
- `node_summaries`

설명:

- 비로그인 공개 현황 페이지용 읽기 모델이다.
- 운영용 상세 내부 정보는 별도 dashboard projection으로 분리하는 것이 적절하다.

### 7.15 `ServiceNotice`

핵심 필드:

- `service_notice_id`
- `title`
- `body_content_id`
- `summary`
- `notice_type`
- `notice_status`
- `visibility`
- `pinned`
- `emergency`
- `published_at`
- `publish_start_at`
- `publish_end_at`
- `created_by_staff_account_id`
- `updated_at`
- `hidden_at`

설명:

- 서비스 단위 공지의 원문과 공개 상태를 소유한다.
- 긴급 공지는 배너 요약과 상세 원문을 분리한다.
- 예약/비공개/숨김 공지는 공개 API에서 존재 여부를 드러내지 않는다.

### 7.16 `RichTextContent`

핵심 필드:

- `content_id`
- `owner_type`
- `owner_id`
- `markdown_source`
- `allowed_markdown_policy`
- `latex_enabled`
- `katex_render_enabled`
- `asset_references`
- `sanitized_render_cache`
- `created_at`
- `updated_at`

설명:

- 문제 본문, 공지, 질문/답변, 메일 템플릿 본문처럼 사용자가 작성하는 글의 공통 원문 모델이다.
- HTML 직접 입력은 허용하지 않고 sanitizer를 통과한 렌더링 결과만 제공한다.
- 이미지 참조는 `asset_id` 기반 내부 참조로만 처리한다.

### 7.17 `ContentAssetPolicy`

핵심 필드:

- `owner_type`
- `asset_purpose`
- `allowed_mime_types`
- `max_file_size`
- `storage_prefix_rule`
- `filename_policy`
- `path_exposure_policy`
- `asset_reference_rule`

설명:

- 여러 컨텍스트에서 사용하는 업로드 asset의 공통 정책 모델이다.
- 기본 이미지 허용 확장자는 `jpg`, `jpeg`, `png`, `webp`이고, 파일당 최대 크기는 `512MB`다.
- 외부 URL 이미지는 허용하지 않는다.
- 실제 asset metadata와 lifecycle은 `ProblemAsset`, `ServiceNoticeAsset`, `ContestAnnouncementAsset`, `ClarificationAsset`처럼 컨텍스트별 모델이 소유한다.

### 7.18 `MailTemplate`

핵심 필드:

- `mail_template_id`
- `template_key`
- `display_name`
- `subject_template`
- `body_content_id`
- `variables`
- `template_status`
- `updated_by_staff_account_id`
- `updated_at`

설명:

- 권한자 초대, 비밀번호 설정/재설정, 참가자 OTP 메일의 템플릿을 표현한다.
- `template_key`는 코드에서 참조하므로 운영자가 변경할 수 없다.
- 테스트 발송은 실제 OTP나 reset token을 생성하지 않고 샘플 값을 사용한다.

## 8. Layered Architecture 원칙

각 bounded context 내부는 아래 레이어를 기본으로 한다.

- `domain`
  - entity
  - value object
  - domain service
  - domain event
- `application`
  - use case
  - command / query handler
  - transaction boundary
  - policy orchestration
- `infrastructure`
  - ORM / repository 구현
  - mail sender
  - redis / queue adapter
  - tailscale / network adapter
  - docker runner adapter
- `interfaces`
  - FastAPI router
  - request / response schema
  - admin API / public API entrypoint

즉, `router -> application -> domain -> repository(interface)` 흐름을 기본으로 하고,
구현체는 infrastructure에서 연결한다.
