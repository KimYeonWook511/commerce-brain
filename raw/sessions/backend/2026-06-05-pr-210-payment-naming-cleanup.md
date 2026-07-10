---
platform: backend
author: KimYeonWook511
created: 2026-06-05
origin:
  - { type: pr, repo: commerce-backend, ref: 210 }
---

# 결제 도메인 네이밍 잔재 정리 (PR #210, 이슈 #209)

(같은 PR 의 AI-메타 메모: 2026-06-05-pr-210-harness-executor-hang-lessons-ai)

## 한 일
- payment 도메인 재설계(PR #205)에서 엔티티 `PaymentAttempt` → `Payment` 통합 후 남은 옛 `attempt` 네이밍 잔재 제거. 이슈 #209 → PR #210.
- 정본 결정 문서: `docs/tasks/payment-naming-cleanup/adr.md`.

## 결정한 것

### mark 계열 메서드 → 동사화 (멀쩡한 동사는 안 건드림)
- `PaymentReservation.markUsed`→`use`, `markExpired`→`expire`. `succeed`/`fail`은 이미 동사라 유지. `markUnknown`은 "결과 불명"에 마땅한 동사가 없어 mark 유지.
- 후보 중 "succeed/fail까지 markSucceeded/markFailed로 통일"도 있었지만, 멀쩡한 동사를 mark로 끌어내리는 대신 어색한 쪽(markUsed/markExpired)만 올리는 게 churn도 적고 자연스러움. markUnknown 1개 예외는 수용.

### attempt 식별자 전면 제거 + 서비스 rename
- 변수/repo메서드/서비스/에러코드 식별자에서 attempt 제거. 서비스: `PaymentApprovalAttemptService`→`PaymentApprovalRecordService`(기존 `PaymentApprovalService`=승인 성공 오케스트레이션과 충돌해서 Record로 구분), `PaymentCancellationAttemptService`→`PaymentCancellationService`(충돌 없음).
- 에러코드는 enum *식별자만* rename(`PAYMENT_ATTEMPT_*`→`PAYMENT_RECORD_*` 등). code 문자열(`PAYMENT-404-2`)·한국어 메시지는 외부 계약이라 보존.

### 정리 경계 — "옛 엔티티 잔재만, 진짜 시도(try)는 보존"
- 식별자가 `Payment` 엔티티(=옛 PaymentAttempt)를 가리키면 정리, 행위의 "시도(try)"를 뜻하면 보존.
- 보존: `attackerAttempt`(보안테스트의 공격 시도), concurrent/retry attempt, 한국어 "시도"(에러 메시지·테스트 @DisplayName), test-only `postprocess` 패키지(test에만 있어 잔재여도 무해; 미래 배치 도입 때 일괄 정비 예정), 역사문서(`docs/ADR.md` 과거 ADR 서술·머지된 task 폴더·flyway migration 파일·outbox 테이블의 `attempt_count` 컬럼).
- 사용자가 스코프를 점점 좁힘: "내가 말한 잔재는 paymentAttempt로 쓰이던 식별자, 진짜 '시도' 성격은 손대지 마라."

### 클래스명 verb/noun 컨벤션은 이번 PR에서 손 안 댐 → 별도 이슈
- 코드베이스가 단일 유스케이스=동사형(`ReservePaymentService`, `AddCartItem`, `OrderCreate`)과 개념묶음=명사형(`PaymentApproval`, `ProductQuery`)이 혼재. 전체 강제 컨벤션 없음.
- 이번엔 Attempt 제거만(최소 변경), 전면 컨벤션 정리는 후속 이슈로 분리.

### enum 값 네이밍은 제외 (논의했다 빠짐)
- `PaymentStatus`(REQUESTED/SUCCEEDED/FAILED/UNKNOWN)를 SUCCESS/REQUEST 등으로 바꿀지 논의. 결론: 안 바꿈.
  - 이유1: `@Enumerated(EnumType.STRING)`이라 enum 상수 이름이 곧 DB 저장 문자열 → rename = 데이터 마이그레이션 = 순수 refactor 아님(이 PR의 "DB 변화 없음" 제약 위반).
  - 이유2: status enum은 "지금 어떤 상태인가"라 과거분사/형용사형이 정석. REQUESTED/SUCCEEDED가 이미 균일하고 옳음. 동사형(REQUEST)은 부적절, 명사형(SUCCESS/FAILURE)은 UNKNOWN에서 깨짐.

### dead code 제거: succeed()의 failCode/failDetail null 리셋
- `Payment.succeed()`는 가드로 REQUESTED 상태에서만 호출되고, REQUESTED 결제는 failCode가 설정될 경로가 없음 → null 리셋은 증명 가능한 no-op dead code. 제거.
- 테스트 `PaymentTest`의 "succeed 후 failCode null" 단언은 "리셋 검증"이 아니라 "성공 결제엔 failCode 없다"는 불변식이라 제거 후에도 통과 → 유지(이 판단을 한 번 더 확인하고 단언 보존).

### save() / saveAndFlush 보존 — load-bearing
- `PaymentRepositoryAdapter.save()`는 `saveAndFlush`(즉시 flush). `succeed`(결제 행 자체의 성공 전이)와 `succeedApproval`(승인 성공 + 주문 완료를 한 트랜잭션으로 묶는 오케스트레이션 메서드)의 명시 save 호출은 `uk_payment_approved_order_key` 위반의 `DataIntegrityViolationException`을 승인 흐름 try-catch 안에서 잡아 이중결제 보상을 트리거하는 데 의존.
- `uk_payment_approved_order_key` = NULL 트릭 unique: 컬럼이 APPROVE+SUCCEEDED일 때만 orderId로 채워지고 그 외엔 NULL인데, unique 제약은 NULL을 중복 허용하므로 "성공 승인 1건만" 강제되고 미승인/실패 결제 행은 제약에 안 걸림. flush를 commit 시점으로 미루면 위반 예외가 try-catch 밖에서 터져 보상 catch를 빠져나감 → 명시 save 제거 금지.

### 리뷰 후속: 명시 save() 4곳 추가 (PR #210 Gemini 리뷰)
- Gemini가 `PaymentApprovalRecordService.fail/failIfRequested`, `PaymentCancellationService.succeed/fail`에 상태 변경 후 `paymentRepository.save(payment)` 명시 호출 제안. 근거는 "응용 계층은 JPA dirty checking 묵시 의존 대신 repository.save(entity)를 명시 호출한다"는 프로젝트 컨벤션(docs/ADR.md).
- 미묘점: 그 컨벤션 문서(docs/ADR.md)가 적용 범위를 "신설 서비스에만 적용, 기존 서비스의 dirty-checking 마이그레이션은 별도 트랙"으로 명시 → 이 두 서비스는 PR #205의 기존 서비스(이번엔 rename만)라 엄밀히는 범위 밖이었음. 그래도 같은 서비스 안 불일치(create/markUnknown은 이미 save, fail/succeed는 안 함)가 실재해서 사용자가 accept. 4곳 각각 별도 커밋.
- 이 4곳은 flush 타이밍이 load-bearing 아님(approvedOrderKey NULL트릭 미관여)이라 save 추가해도 안전.

## 배운 것
- `@Enumerated(EnumType.STRING)` 매핑 enum의 상수 이름은 DB에 저장되는 값이다 → 상수 rename은 순수 refactor가 아니라 데이터 마이그레이션 동반.
- status enum 네이밍은 "상태 서술"이라 과거분사/형용사형(REQUESTED/SUCCEEDED)이 정석. 동사형은 행위라 부적절.
- `saveAndFlush`의 즉시 flush 타이밍이 제약위반 catch의 동작을 좌우할 수 있다 — "save 명시 호출 통일" 같은 스타일 정리가 실은 동작-인접 변경일 수 있으니 flush 타이밍 의존부터 확인.

## 다음 단계
- payment(및 코드베이스) 서비스 클래스명 verb/noun 컨벤션 전면 정리 — 별도 이슈로 분리 예정(아직 미생성).
