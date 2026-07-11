---
platform: backend
author: KimYeonWook511
created: 2026-06-05
origin:
  - { type: pr, repo: commerce-backend, ref: 210 }
---

# 결제 도메인 attempt 네이밍 잔재 정리 — 옛 PaymentAttempt 식별자만 걷어내고 진짜 '시도'는 보존

결제 도메인 재설계(PR #205)에서 엔티티 `PaymentAttempt`를 `Payment`로 통합했는데, 변수·repo 메서드·서비스 클래스·에러코드 식별자에는 옛 `attempt` 네이밍이 그대로 남아 있었다. 그 잔재를 걷어내고 도메인 상태 반영 메서드의 이름 패턴을 정리한 세션(이슈 #209 → PR #210)이다. 핵심 제약은 "순수 refactor — 외부 동작·API·DB 스키마는 일절 안 건드린다"였고, 그래서 어디까지가 정리 대상이고 어디부터가 보존 대상인지 경계를 긋는 게 실제 작업의 대부분이었다.

## 결정한 것

### 1. mark 계열 메서드 → 동사화 (멀쩡한 동사는 안 건드림)
- **핵심:** 예약(reservation) 엔티티 `PaymentReservation`의 `markUsed`→`use`, `markExpired`→`expire`로 축약했다. `succeed`/`fail`은 이미 동사라 그대로 두고, `markUnknown`은 "결과 불명"에 마땅한 동사가 없어 `mark` 접두사를 정직한 표현으로 유지했다.
- **검토한 대안:** 반대로 `succeed`/`fail`까지 `markSucceeded`/`markFailed`로 `mark*`에 통일하는 안도 있었다. 기각. 멀쩡한 동사를 `mark`로 끌어내리면 오히려 어색하고 변경 범위(churn)만 커진다. 어색한 쪽(`markUsed`/`markExpired`)만 동사로 끌어올리는 게 churn도 적고 도메인 행위 중심 네이밍 원칙과도 일관된다. `markUnknown` 하나가 예외로 남지만 동사 부재를 이유로 수용했다.

### 2. attempt 식별자 전면 제거 + 서비스 rename
- **핵심:** `Payment` 타입(=옛 `PaymentAttempt`)을 가리키는 변수·파라미터·필드·repo 메서드·처리 메서드에서 `attempt`를 걷어냈다. 예: 변수 `approveAttempt`→`approvePayment`, repo `findApproveAttempt`→`findApprovePayment`, 처리 `processApproveAttempt`→`processApprovePayment`.
- **서비스 rename:** 승인 실패·기록을 다루는 `PaymentApprovalAttemptService`→`PaymentApprovalRecordService`, 취소를 다루는 `PaymentCancellationAttemptService`→`PaymentCancellationService`.
  - 취소 쪽은 `Payment`+`Cancellation`으로 이름 충돌이 없어 깔끔하게 `attempt`만 뗐다.
  - 승인 쪽은 그냥 `PaymentApprovalService`로 줄이면 **이미 존재하는** `PaymentApprovalService`(승인 성공 + 주문 완료를 한 트랜잭션으로 묶는 오케스트레이션 서비스)와 충돌한다. 그래서 이 서비스가 "승인 시도 기록"을 담당한다는 뜻으로 `Record`를 넣어 `PaymentApprovalRecordService`로 구분했다.
- **에러코드는 식별자만 rename:** enum 상수 이름(`PAYMENT_ATTEMPT_NOT_FOUND`→`PAYMENT_RECORD_NOT_FOUND`, `PAYMENT_ATTEMPT_AMOUNT_MISMATCH`→`PAYMENT_RECORD_AMOUNT_MISMATCH`, `PAYMENT_ATTEMPT_STATUS_TRANSITION_NOT_ALLOWED`→`PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`)만 바꾸고, code 문자열(`PAYMENT-404-2` 등)과 한국어 메시지("결제 시도 이력…")는 **외부 계약**이라 손대지 않았다.

### 3. 정리 경계 — "옛 엔티티 잔재만, 진짜 '시도(try)'는 보존"
- **판단 기준:** 식별자가 `Payment` 엔티티(=옛 `PaymentAttempt`)를 가리키면 정리 대상, 행위로서의 "시도(try)"를 뜻하면 보존 대상.
- **보존한 것:** 보안 테스트의 공격 시도 `attackerAttempt`, 동시성/재시도 맥락의 concurrent·retry attempt, 한국어 "시도"(에러 메시지·테스트 `@DisplayName`), test 전용 `postprocess` 패키지(테스트에만 있어 잔재여도 무해 — 미래에 배치 도입할 때 일괄 정비 예정), 그리고 역사 기록(과거 결정을 누적하던 레거시 단일 ADR 문서의 서술, 이미 머지된 task 폴더, flyway migration 파일, outbox 테이블의 `attempt_count` 컬럼). 역사 기록은 결정 당시 상태(`PaymentAttempt`)를 기록한 것이라 소급 수정하지 않는다.
- 작업 중 스코프를 점점 좁혀 명시했다: "내가 말한 잔재는 `paymentAttempt`로 쓰이던 식별자다. 진짜 '시도' 성격은 손대지 마라." `s/attempt/payment/g` 식 전역 치환이었다면 의미를 훼손했을 것이라, 각 식별자를 case-by-case로 판단했다.

### 4. 클래스명 verb/noun 컨벤션은 이번 PR에서 손 안 댐 → 별도 이슈
- 코드베이스는 단일 유스케이스=동사형(`ReservePaymentService`, `AddCartItem`, `OrderCreate`)과 개념 묶음=명사형(`PaymentApproval`, `ProductQuery`)이 혼재하고, 이를 강제하는 전면 컨벤션은 없다.
- 이번엔 `attempt` 제거라는 최소 변경만 하고, verb/noun 전면 컨벤션 정리는 후속 이슈로 분리했다. 방금 만든 `PaymentApprovalRecordService` 같은 이름도 그 컨벤션 재검토 대상이다.

### 5. status enum 값 네이밍은 제외 (논의했다 뺌)
- `PaymentStatus`(`REQUESTED`/`SUCCEEDED`/`FAILED`/`UNKNOWN`)를 `SUCCESS`/`REQUEST` 같은 형태로 바꿀지 논의했으나 **안 바꾸기로** 했다.
  - **이유 1 (제약 위반):** 이 enum은 `@Enumerated(EnumType.STRING)`이라 상수 이름이 곧 DB에 저장되는 문자열이다. rename하면 DB에 이미 저장된 값과 불일치가 생겨 데이터 마이그레이션이 반드시 따라붙는다 — 이 PR의 "DB 변화 없음" 제약을 깬다.
  - **이유 2 (의미):** status enum은 "지금 어떤 상태인가"를 나타내므로 과거분사/형용사형이 정석이다. `REQUESTED`/`SUCCEEDED`는 이미 균일하고 옳다. 동사형(`REQUEST`)은 행위라 부적절하고, 명사형(`SUCCESS`/`FAILURE`)은 `UNKNOWN`에서 균질성이 깨진다.

### 6. dead code 제거 — succeed()의 failCode/failDetail null 리셋
- **핵심:** `Payment.succeed()` 안의 `failCode = null; failDetail = null;` 두 줄을 제거했다.
- **증명:** `succeed()`는 진입 가드로 `REQUESTED` 상태에서만 호출되고, `REQUESTED` 상태의 결제는 `failCode`가 설정될 경로가 애초에 없다. 따라서 그 null 리셋은 증명 가능한 no-op dead code다.
- **테스트 단언은 유지:** `PaymentTest`의 "succeed 후 failCode가 null" 단언은 "리셋 동작을 검증"하는 게 아니라 "성공한 결제엔 failCode가 없다"는 불변식을 검증하는 것이라, 리셋 코드를 지워도 그대로 통과한다 → 이 판단을 한 번 더 확인하고 단언은 보존했다.

### 7. save() / saveAndFlush 보존 — load-bearing
- **핵심:** 결제 저장을 담당하는 JPA infra adapter `PaymentRepositoryAdapter.save()`는 내부적으로 `saveAndFlush`(즉시 flush)를 쓴다. 승인 오케스트레이션에서 결제 행 성공 전이 후에 이뤄지는 명시적 `save()` 호출은 단순 관성 코드가 아니라 동작에 필수적이다 — 지우면 안 된다.
- **왜 load-bearing인가 — NULL 트릭 unique 제약:** "한 주문에 성공한 승인 결제는 1건"을 강제하는 unique 제약 `uk_payment_approved_order_key`가 있다. 대상 컬럼(`approvedOrderKey`)은 타입이 APPROVE이고 상태가 SUCCEEDED로 전이될 때만 `orderId`로 채워지고 그 외엔 NULL로 남는다. unique 제약은 NULL 중복을 허용하므로, 미승인·실패 결제 행은 제약에 걸리지 않고 "성공 승인 1건만" 강제된다.
- **flush 타이밍이 catch를 좌우한다:** 승인 성공 + 주문 완료를 한 트랜잭션으로 묶는 오케스트레이션 메서드(`succeedApproval`)는 성공 전이 뒤 명시적으로 저장하는데, 이 `saveAndFlush`의 즉시 flush 덕에 제약 위반이 **트랜잭션 커밋 전, 승인 흐름의 try-catch 안에서** 확정된다. 그 `DataIntegrityViolationException`을 같은 트랜잭션 catch에서 잡아 이중결제 보상을 트리거한다. flush를 커밋 시점으로 미루면 위반 예외가 try-catch 밖에서 터져 보상 catch를 빠져나가므로, 이 명시적 save는 제거 금지다.

### 8. 리뷰 후속 — 명시 save() 4곳 추가
- **핵심:** AI 코드 리뷰(Gemini)가 상태 변경 후 `paymentRepository.save(payment)`를 명시 호출하지 않던 4곳에 추가를 제안했고 수용했다: 승인 기록 서비스의 `fail`·`failIfRequested`, 취소 서비스의 `succeed`·`fail`. (`failIfRequested`는 보상 흐름에서 호출되는 경로다.) 4곳 각각 별도 커밋으로 반영했다.
- **근거가 된 컨벤션:** "응용 계층은 JPA dirty checking에 묵시적으로 기대지 말고 `repository.save(entity)`를 명시 호출한다"는 프로젝트 영속화 컨벤션. managed entity의 `save()`는 JPA 내부적으로 no-op이지만, 코드 표면에 "이 시점에 저장 의도"를 드러내 응용 계층의 영속화 책임을 명시하는 게 목적이다.
- **미묘한 점 — 엄밀히는 적용 범위 밖이었다:** 그 컨벤션은 적용 범위를 "이 결정 이후 **신설되는** 응용 Service에만 적용하고, 기존 도메인의 dirty-checking 의존 코드 마이그레이션은 별도 트랙"으로 못박아 뒀다. 그런데 이 두 서비스는 PR #205부터 있던 기존 서비스라(이번 PR은 rename만) 엄밀히는 범위 밖이었다. 그래도 **같은 서비스 안의 불일치**(예: `create`·`markUnknownIfRequested`는 이미 save를 명시 호출하는데 `fail`류는 안 함)가 실재해서, 일관성을 위해 수용했다.
- **안전성:** 이 4곳은 flush 타이밍이 load-bearing이 아니다(NULL 트릭 unique 제약에 관여하지 않음). 그래서 명시 save를 더해도 위 승인 흐름의 이중결제 보상 동작에는 영향이 없어 안전하다.

## 배운 것

- **`@Enumerated(EnumType.STRING)` 매핑 enum의 상수 이름은 DB에 저장되는 값 그 자체다.** 상수 rename은 순수 refactor가 아니라 데이터 마이그레이션을 동반한다. 이번에 정리한 `PaymentErrorCode` enum 식별자들은 영속 대상이 아니라 자유롭게 바꿀 수 있었지만, `PaymentStatus` 같은 상태 enum 값은 그렇지 않다.
- **status enum 네이밍은 "상태 서술"이라 과거분사/형용사형(`REQUESTED`/`SUCCEEDED`)이 정석이다.** 동사형은 행위라 부적절하고, 명사형은 `UNKNOWN` 같은 값에서 균질성이 깨진다.
- **`saveAndFlush`의 즉시 flush 타이밍이 제약 위반 catch의 동작을 좌우할 수 있다.** "명시 save 호출을 통일하자" 같은 스타일 정리가 실은 동작-인접 변경일 수 있으니, 손대기 전에 flush 타이밍에 의존하는 부분(여기선 NULL 트릭 unique 제약 + 보상 catch)이 있는지부터 확인해야 한다.
- **의미가 두 갈래인 단어(`attempt` = 옛 엔티티 잔재 vs 진짜 '시도')는 전역 치환하지 말고 어원 인접 식별자를 먼저 목록화한 뒤 하나씩 판단한다.** 도메인 결정 문서에 "이 식별자가 정리 대상 엔티티를 가리키는가, 진짜 의미의 단어인가"라는 판단 기준을 명문화해두면 판단 비용이 준다.
- **엔티티 rename은 연관 식별자(변수·파라미터·repo 메서드·서비스·에러코드·테스트)까지 같은 PR에서 함께 정리하는 게 낫다.** 이번 잔재 정리 PR은 PR #205에서 `PaymentAttempt → Payment` rename이 끝났음에도 식별자 정리가 후속으로 남겨져 별도 cleanup PR이 필요해진 결과다.

## 미해결·열린 질문

- **서비스 클래스명 verb/noun 컨벤션 전면 정리** — 별도 후속 이슈로 분리 예정(아직 미생성). 이번에 만든 `PaymentApprovalRecordService` 등의 이름도 컨벤션 확정 후 재검토 대상이다.
- **`PaymentStatus`·`PaymentReservationStatus` enum 값 rename** — DB 마이그레이션 작업과 짝을 이뤄 별도 task로만 처리 가능. 이번엔 의도적으로 제외.
- **test 전용 `postprocess` 패키지의 attempt 잔재 정비** — 배치 도입 시 일괄 정비로 미룸.
