---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

## 도메인 개요

payment 도메인은 **두 Aggregate**(`Payment`, `PaymentAttempt`)와 **PG provider 서브패키지**(`payment/naverpay/`)로 구성된다.

- **`Payment`** — 결제 최종 상태. `order_id`, `merchantPayKey`, `pgPaymentId` 모두 unique. 한 주문당 최대 1건. 존재 = "결제 완료".
- **`PaymentAttempt`** — 승인/취소 시도 이력. unique key `(merchant_pay_key, provider, payment_id, type)`. 같은 결제 건에 대해 `type=APPROVE` row와 `type=CANCEL` row가 별도로 존재. status: `REQUESTED → SUCCEEDED | FAILED` 단방향.

application 서비스 5개 + naverpay 1개:

- `PaymentReadyService` — 결제 준비 (PG 호출 전 사전 데이터 생성)
- `PaymentApprovalService` — 결제 완료 반영 (`completeApprovedPayment`) + 보상 필요 여부 판단 (`isCompensationRequired`)
- `PaymentApprovalAttemptService` — APPROVE attempt 생성·전이 (`getOrCreate`, `succeed`, `fail`, `failIfRequested`)
- `PaymentCancellationAttemptService` — CANCEL attempt 생성·전이 (`getOrCreate`, `succeed`, `fail`)
- `PaymentApprovalCompensationService` — 보상 dispatcher 4개 (`compensateMerchantKeyMismatch` / `AmountMismatch` / `DuplicatePayment` / `Unexpected`) + private `runPgCancel`
- `NaverPayApprovalService` (`naverpay.application`) — PG 응답 switch로 main flow 조율. PG cancel은 `this::pgCancel` 메서드 참조로 `PgCanceller` 콜백 제공.

PG 연동:

- `NaverPayGateway` (`naverpay.infrastructure`) — `NaverPayClient` 호출 + 응답 코드(`NaverPayApproveCode` 등) → 도메인 코드(`PaymentAttemptFailCode`, `PaymentErrorCode`) 매핑. PG-specific. `NaverPayApproveResult` / `NaverPayCancelResult` 반환.
- `payment.application.port.PgCanceller` — `@FunctionalInterface cancel(PaymentAttempt, String) → CancelOutcome`. payment.application이 PG-specific 타입을 import하지 않게 하는 좁은 콜백 경계.

핵심 흐름:

```
[승인]
NaverPayController
  → NaverPayApprovalService.processApproveAttempt
       (멱등 switch: REQUESTED → PG 호출 / SUCCEEDED → no-op / FAILED → 재시도)
  → NaverPayGateway.approve  (PG 호출 + 응답 코드 매핑)
  → completeVerifiedApproval
       (attempt.verifyApprovedResponse → succeed → PaymentApprovalService.completeApprovedPayment)
  → catch 분기 → PaymentApprovalCompensationService.compensate{...}(this::pgCancel)
       → runPgCancel: failIfRequested → isCompensationRequired → getOrCreate(CANCEL) → pgCancel → succeed/fail
```

## 핵심 결정

### 1. PaymentAttempt 멱등 재요청 amount mismatch는 명시적 거부 (ADR-010)

**결정 요약**: `(merchantPayKey, provider, paymentId, type)` 멱등 키 재요청이 기존 attempt amount와 다르면 `PAYMENT_ATTEMPT_AMOUNT_MISMATCH` (409 Conflict). 기존 attempt 상태(REQUESTED/FAILED/SUCCEEDED)와 무관.

**두 ErrorCode의 의미 차이가 결정적** — 비교 대상·시점·운영 대응이 완전히 다른 두 검증:

| ErrorCode | HTTP | 비교 대상 | 의미 | 운영 대응 |
|---|---|---|---|---|
| `PAYMENT_ATTEMPT_AMOUNT_MISMATCH` | 409 | *DB에 기록된 amount* ↔ *사용자가 같은 멱등 키로 보낸 amount* | 호출자 측 *재요청 mismatch* — 같은 키로 다른 값 보냄 | 호출자(클라이언트) 산정 오류 디버깅 |
| `PAYMENT_AMOUNT_MISMATCH` | 400 | *PG에서 결제 완료된 amount* ↔ *사용자 요청 amount* | 외부 응답 검증 — 세 가지 원인 가능: PG 측 문제 / 악의적 사용자 / 우리 서버 산정 오류 | PG 측 알람 + 보안 모니터링 + 내부 디버깅 |

→ 둘은 *비교 대상과 비교 시점이 완전히 다른 검증*. 운영상 원인 추적 경로도 다름 → ErrorCode + HTTP status 분리가 자연스러움.

**검증 위치도 catch 한 곳에 한정** (ADR-B): `save()` 전 pre-check면 정상 경로마다 SELECT 1회 추가. `NOT_SUPPORTED`에서 save commit 직후 unique 위반이 잡히므로 catch에서 amount 비교 + 재조회로 충분.

**상태 무관 일관 거부** (ADR-C): FAILED에만 허용하면 "FAILED면 amount 수정 가능"이라는 *암묵적 규칙*이 생겨 멱등성 계약 흐려짐.

**merchantPayKey 생명주기 — 코드 직접 확인 결과**:

| 단계 | 동작 | 위치 |
|---|---|---|
| Order 생성 | merchantPayKey 없이 Order 만듦 | `OrderCreateProcessor.execute()` |
| 결제 준비 | `null`이면 `PAY-{ULID}` 발급 후 `order.assignMerchantPayKey()` 호출 | `PaymentReadyService.readyPayment()` |
| 결제 승인 | merchantPayKey로 Order 조회 → Payment·PaymentAttempt에 *복사* 저장 | `NaverPayApprovalService.approve()` |

**`assignMerchantPayKey()` 핵심**: null일 때만 set (멱등 setter) — **한 Order = 한 merchantPayKey 영구**.

**ADR-010 "새 merchantPayKey 발급" 표현의 실제 의미**:
- ADR-010 본문: "*amount 변경이 필요하면 새 `merchantPayKey`로 새 요청 발급*"
- 실제 코드: 같은 Order에서는 *새 merchantPayKey 발급 불가* (멱등 setter)
- 따라서 *amount 변경 = 새 Order 생성 = 새 merchantPayKey*
- ADR-010 표현이 오해의 소지 있음 → "새 Order로 새 요청 발급"이 정확

**미해결 — 도메인 경계 재설계 검토 (Issue #174)**:
- 현재 Order에 merchantPayKey(unique)가 저장된 구조 = *결제 도메인 식별자가 주문 도메인에 박힘* → 도메인 책임 누수.
- 한 Order = 한 PG 제약 (사용자가 다른 PG 시도 시 새 Order 필요).
- 검토 옵션: (A) 현재 유지 / (B) 별도 `payment_reference` 테이블 권장 / (C) Payment 의미 변경 / (D) PaymentAttempt 재사용 권장 안 함.
- **Issue #174** (refactor: payment 도메인 Order↔결제 경계 재설계) 로 등록.

**미해결 — `PAYMENT_AMOUNT_MISMATCH`(400)의 세 원인 구분 메트릭 부재**:
- 현재 한 ErrorCode로 PG 문제 / 악의적 사용자 / 우리 서버 산정 오류 세 가지가 합쳐져 있음.
- 운영 시점에 어느 게 원인인지 *추가 컨텍스트(로그·메트릭)*로 구분해야 함.
- 향후 별도 ErrorCode 분리나 reason 메타데이터 추가 검토.

**트레이드오프**: 호출자가 잘못된 amount로 재시도하면 즉시 4xx 실패 (이전에는 침묵 후 뒤늦게 발견).

### 2. PaymentAttempt 도메인 라이프사이클 보호 — succeed/fail 선조건 검증 (ADR-012)

**결정 요약**: `PaymentAttempt.succeed(respondedAt)` / `fail(failCode, detail, respondedAt)`은 호출 시점에 `status == REQUESTED`를 검증. 위반 시 `PAYMENT_ATTEMPT_STATUS_TRANSITION_NOT_ALLOWED` (500). **멱등 자기 전이도 거부**.

**왜 멱등 자기 전이(SUCCEEDED → SUCCEEDED)도 거부하나** — 도메인 모델의 최후 방어선:

상위 레이어 멱등 처리(`getOrCreate` + `processApproveAttempt` switch)가 *완벽하면* 같은 attempt에 succeed가 두 번 호출될 일이 없음. 하지만 *상위가 깨질 수도 있는 경우들*이 있음:

- PG 콜백 중복 발송 (네이버페이 retry, 사용자가 redirect URL 두 번 클릭 등)
- 상위 switch case 누락 (SUCCEEDED 케이스를 빠뜨리면 default 동작)
- 향후 새 호출 경로 추가 (admin manual reconcile 등)
- 테스트 오류

**거부 안 했을 때의 위험**:
- 두 번째 `succeed()` 호출 = `failCode = null` 초기화 + `respondedAt` 덮어쓰기
- → *흔적 없이 실패 사유가 사라짐*. 운영 시점 추적 불가.

**거부했을 때의 가치**:
- 즉시 500 → 운영 대시보드 알람 → 상위 멱등 처리 버그를 *즉시 감지*
- 도메인 모델이 *상위의 완벽함을 가정하지 않고 자기 무결성을 지킴* (Defensive programming)

→ 도메인이 *최후의 방어선*. 상위에 한 번 더 의존하지 않음.

**여러 결정이 누적된 구조**:

1. **상태 전이는 엄격 검증** — 위 설명 그대로.
2. **type 가드는 처음에는 도입 → 나중에 제거** — *도메인 책임 경계의 자연스러운 이동*:
   - `PaymentAttempt`에는 `type` 컬럼 (`APPROVE` / `CANCEL`)이 있음.
   - 초기 ADR-012의 mark 메서드 4개(`markApproveSucceeded`, `markApproveFailed`, `markCancelSucceeded`, `markCancelFailed`)는 *상태 가드 + type 가드*를 함께 검증.
   - **type 가드 도입 의도**: 메서드 이름에 *처리할 type이 명시*되어 있음 (예: `markApproveSucceeded`). 의도와 다른 type이 들어오면 *호출자 실수*. 도메인이 그것을 즉시 거부해서 *메서드 이름에 표현된 의도가 침범당하지 않도록 방어*.
   - 이후 ADR-2(payment-attempt-service-split)에서 service가 `PaymentApprovalAttemptService`(APPROVE 전용) vs `PaymentCancellationAttemptService`(CANCEL 전용)로 분리됨.
   - service 분리 결과: *호출자가 type을 잘못 보낼 경로 자체가 사라짐*. APPROVE 흐름은 APPROVE attempt만, CANCEL 흐름은 CANCEL attempt만 다룸 → type 가드의 *방어 가치 상실*.
   - mark 4개 → `succeed` / `fail` 2개로 통합 (type 가드 제거, 상태 가드는 유지).
   - **진화의 의미**: 처음엔 *도메인 메서드가 type 무결성도 책임*. 이후 service 분리로 *흐름이 type을 자연 강제* → 도메인의 type 책임이 중복이 됨 → 중복 제거. *상위 구조 변화가 도메인 모델의 책임 경계를 다시 그리는* 자연스러운 진화. 처음부터 완성된 설계가 아니라 *코드 진화의 결과*.
3. **에러 코드는 500**: amount mismatch(외부 원인 4xx)와 *도메인 무결성 위반(내부 결함)*은 모니터링 카테고리가 다름. 운영 대시보드에서 "호출자 4xx" vs "내부 5xx" 분리.

**선택하지 않은 대안**:
- *멱등 자기 전이 허용 (SUCCEEDED → SUCCEEDED)* — 위 "거부 안 했을 때의 위험" 때문에 기각.
- *enum-level 가드만, 도메인 메서드는 무방어* — Order 도메인(`Order.cancel`: `status != INIT` → throw) 패턴과 일관성을 위해 도메인 보호 선택.

**트레이드오프 + 그게 다른 ADR을 유발**: 새 검증이 도입되면서 *보상 흐름의 catch 블록에서 fail() 호출 시 race window throw* 가능성 발생 → ADR-013 (보상 catch 2차 예외 처리) + ADR-D (try-catch 임시 처방). 이후 ADR-014 (Payment 존재 체크)로 race window 자체가 축소되어 임시 처방이 보조 방어선으로 격하.

### 3. ArchUnit 대신 ADR + JavaDoc으로 호출 정책을 명문화 (ADR-014 PaymentAttempt mark 캡슐화)

**결정 요약**: `PaymentAttempt.succeed`/`fail`은 `PaymentApprovalAttemptService` / `PaymentCancellationAttemptService` 외부에서 직접 호출하지 않는다. Java `public` 접근자라 컴파일러 강제 불가 → ADR + JavaDoc + 단일 호출처 유지로 정책 강제.

**후보 비교**:
- **ArchUnit CI 차단** — 가장 강한 보장. 도입 비용 발생.
- **패키지 구조 변경 (package-private)** — `PaymentAttempt`와 `*AttemptService`를 같은 패키지로 이동. 대규모 변경 + 기존 패키지 분리 의도 훼손.
- **ADR + JavaDoc ✓** — 현재 위반 경로 없음. 새 기여자 실수만 방지하면 충분.

**ArchUnit 미도입의 결정적 이유 — 학습 단계의 우선순위**:
- 현 시점은 *개발 과정 자체가 주*, 그 과정의 고민을 *문서로 녹이는 것이 부가적 가치*인 단계.
- ArchUnit 도입은 *별도 학습 + 룰 설계 + CI 통합* 시간이 필요한데, 그 비용이 얼마나 될지 가늠 안 됨.
- 반면 ADR + JavaDoc은 *바로 개발하면서 기록 가능*한 방식 — 채택 비용 0, 즉시 적용.
- 즉 "학습 단계라 도구를 더 늘리고 싶지 않음" + "개발 흐름을 끊지 않고 기록 가능한 방법 우선" 두 가지가 결합된 의도적 선택.

**향후 ArchUnit 도입 시점**:
- 다른 도메인 아키텍처 테스트와 함께 일관되게 도입 권장. 단일 도메인 단독 도입은 룰 운영 부담 대비 가치 낮음.
- 운영 진입 + 안정화 단계에 검토.

### 4. 보상 진행 여부는 PaymentAttempt 상태가 아니라 Payment 엔티티 존재로 판단 (ADR-014)

**결정 요약**: `NaverPayApprovalService.failApproveAndCancelApprovedPayment`가 PG cancel 전 `PaymentApprovalService.isCompensationRequired(merchantPayKey)` 호출 → Payment 존재 시 cancel skip + log.warn.

**문제 시나리오 (#114)**:

```
Thread A: completeApprovedPayment
  → succeedApproveAttempt (메모리상 SUCCEEDED)
  → order.completePayment() race throw
  → catch → failApproveAndCancelApprovedPayment
       → failApprove → 새 status 가드 throw (REQUESTED 아님)
       → PG cancel 흐름 중단
       → PG는 결제 승인됨 + 우리 시스템 미반영 (외부 정합성 깨짐)
```

기존 구조는 *PaymentAttempt.status로 cancel 진행 여부 판단* → attempt에 row lock이 없어 race-unsafe.

**Payment 도입의 사고 흐름 — lock 회피 + DB unique 활용 패턴의 일관된 적용**:

원래는 *Payment 테이블 없이 PaymentAttempt만으로* 최종 결제 상태까지 표현하려 했음. 결제 시도 row를 update해서 최종 상태로 전이.

문제 인식:
- 코드 분기 너무 많아짐 (시도/완료/취소가 같은 row에 섞임)
- 동시성/멱등 처리 위해 *lock 필요*
- lock 채택 시 보안↑ but **서비스 안정성↓**. 결제는 빈도가 높아 *lock 경쟁이 클*것으로 판단.
- 보안 vs 안정성 트레이드오프 인지.

해결 방향: **"lock 안 쓰는 방법 필요"**
- unique 전략 + 최종 상태 별도 테이블 (Payment) 도입
- DB unique 제약이 *lock 역할을 자연스럽게* 함
- 멱등성/동시성을 lock이 아니라 *데이터 모델로 보장*

→ 결정 4번의 "Payment 존재 = race-safe"는 *우연이 아니라 lock 회피 패턴의 일관된 적용*. 결정 1번 (PaymentAttempt 멱등 키 unique)도 같은 패턴 — 이 도메인의 *통일된 설계 원칙*.

**후보 비교**:
- **`PaymentAttempt @Version` (낙관적 락)** ✗ — DB 스키마 변경 (운영 마이그레이션) + attempt 수준 락 범위가 Order lock과 중첩.
- **`PaymentAttempt FOR UPDATE` (비관적 락)** ✗ — Order FOR UPDATE와의 *락 획득 순서 조율* 필요. 데드락 가능성.
- **Payment 존재 체크 ✓** — Payment의 `order_id`/`merchantPayKey`/`pgPaymentId` 모두 unique + `completeApprovedPayment`가 Order FOR UPDATE 안에서 Payment 저장 → Payment 존재는 DB 레벨에서 *race-safe*. **스키마 변경 없음**.

**DDD 관점**: 두 별도 Aggregate(`Payment`, `PaymentAttempt`)의 *독립 불변식*을 cross-Aggregate 협력으로 활용. `PaymentApprovalService`가 Payment Aggregate owner이므로 `isCompensationRequired`를 노출 — NaverPay adapter가 `paymentRepository`에 직접 접근하지 않음. 미래 Payment 도메인 분리 시 외부 API로 자연 승격 가능.

**트랜잭션 격리 (ADR-T2)** + **현재 인식의 한계 — Issue #160으로 이미 등록됨**:

- 결정 시점 의도: *명시적 트랜잭션 격리 표현* — 메서드가 어떤 상태로 쓰이는지 코드로 드러냄.
- 현재 구현: `PaymentApprovalCompensationService`에 클래스 레벨 `@Transactional` 금지 + `isCompensationRequired`의 `REQUIRES_NEW`로 격리.
- **DDD 학습 후 자기 비판**:
  - *Application 레이어가 JPA(1차 캐시 동작)에 의존* — Application 코드가 인프라 디테일(JPA의 1차 캐시 우회 필요)을 알게 되는 구조. DDD 원칙 위반.
  - `REQUIRES_NEW`는 *새 DB 커넥션 요구* — 외부 API 호출처럼 *경계가 확실히 분리된* 경우만 적합. 단순 1차 캐시 우회용으로 쓰면 *리소스 낭비 + 위험* (커넥션 풀 고갈, 데드락 등).
- **더 깊은 분석 (Issue #160)**:
  - 호출 경로를 추적해보면 이미 "트랜잭션 구조 분리" 패턴이 적용되어 있어 *격리할 외부 1차 캐시가 실제로 존재하지 않음*.
  - 즉 `REQUIRES_NEW`가 **실효조차 없는 상태**.
  - 호출 흐름:
    ```
    NaverPayApprovalService.completeVerifiedApproval()   ← @Transactional 없음
      try { paymentApprovalService.completeApprovedPayment(...) ← TX1 }
      catch {
        // TX1 PaymentException으로 롤백됨 → 1차 캐시 소멸
        paymentApprovalCompensationService.runPgCancel()  ← @Transactional 없음
          └─ paymentApprovalService.isCompensationRequired(...)  ← 외부 TX 없음
      }
    ```
- 작업 범위: `REQUIRES_NEW` 제거 + 주석을 비즈니스 의도(보상 트랜잭션 시작 전 커밋 완료된 DB 상태 기준 판단)로 교체.
- **Issue #160** (refactor: PaymentApprovalService.isCompensationRequired REQUIRES_NEW 제거) 로 이미 등록됨.

### 5. 보상 정책은 payment.application 책임, PG 어댑터는 cancel 콜백만 제공 (ADR-015)

**결정 요약**: `NaverPayApprovalService`의 보상 dispatcher 4개 + 공통 골격을 `PaymentApprovalCompensationService`(payment.application)로 이동. PG cancel은 `PgCanceller` @FunctionalInterface 콜백으로 위임. PG 응답은 `CancelOutcome` record로 변환해 payment.application이 `NaverPayCancelResult`를 직접 import하지 않음.

**후보 비교 (3택)**:
1. **`PgCanceller` 좁은 콜백 ✓** — `@FunctionalInterface cancel(PaymentAttempt, String) → CancelOutcome`. NaverPay가 `this::pgCancel` 메서드 참조로 구현. 인터페이스 추가 *없이* 의존 역전. *지금 필요한 최소 구조*.
2. **`PaymentGateway` port 완전 inversion ✗** — PG-agnostic approve/cancel 통합 port. PG 둘 이상 추가 시 자연스러우나 *현 시점 over-engineering*.
3. **Strategy 패턴 ✗** — PG별 보상 전략 객체. PG 하나뿐인데 premature.

**솔직한 도입 배경 — AI 추천을 시도해본 학습 흔적**:
- 본인의 *선호는 `PaymentGateway` 완전 inversion* — Gateway가 도메인 의미를 더 잘 표현.
- 다만 *AI가 추천한 `PgCanceller` 좁은 콜백 방식*을 한 번 시도해본 결과물.
- AI 의견을 *완전히 동화시키지 못한 상태* — 지금 코드를 봐도 "왜 이렇게 했을까?" 의도가 명확히 안 와닿음.
- 추측되는 의도: NaverPay에 *application이 직접 의존하지 않게* 하기 위한 의존성 역전.
- → *PG 추가 시점 전까지는 역할이 모호*. PG 둘 이상 추가되는 시점에 `PaymentGateway` port로 자연 승격 또는 처음부터 다시 설계.

**`CancelOutcome` 변환 책임**: PG-specific `NaverPayCancelResult.Status` → 도메인 `CancelOutcome.Status` 매핑. `ALREADY_CANCELED`는 `SUCCESS`로 매핑 (PG 측 이미 취소된 상태 = 우리 입장에서 cancel 목적 달성).

**`compensateMerchantKeyMismatch` 특이성**: 4개 dispatcher 중 *유일하게 `PgCanceller`를 파라미터로 받지 않음*. PG가 우리가 발급한 `merchantPayKey`를 모르는 상황 = PG 측에 결제 자체가 성립하지 않음 = cancel 호출 대상이 없음. 메서드 시그니처로 의도 표현.

**`PgCanceller` 예외 swallow 정책**: `runPgCancel` 내에서 `pgCanceller.cancel` 중 `PaymentException` 발생 시 log.warn 후 swallow. *원래 승인 실패 예외(1차)가 보상 실패 예외(2차)에 가려지지 않도록* (ADR-013 보상 catch 2차 예외 정책의 적용).

**미해결 — `ALREADY_CANCELED` 케이스 모니터링 부재**:
- 코드 확인 결과 `NaverPayGatewayImpl.java:138`에서 `NaverPayCancelResult.alreadyCanceled()` 반환 후 *SUCCESS로 매핑되며 흔적 없이 정상 처리*.
- 그러나 *PG가 우리보다 먼저 취소한 상태* = 통신 이상/타이밍 이슈/외부 시스템 비정상의 신호일 수 있음.
- 운영팀이 이 케이스를 *별도로 인지*할 수 있도록 log/메트릭 추가 필요.
- 향후 보강 대상.

**트레이드오프**: PG가 둘 이상 추가될 때 `PgCanceller` 주입 위치 재설계 필요. 그 시점에 `PaymentGateway` port로 자연 승격.

### 6. 보상 catch 2차 예외는 의도 캡슐화 메서드로 평탄화 (ADR-013, exception-strategy.md)

**결정 요약**: catch 블록에서 *보상 트랜잭션 / 알림 발송* 같은 2차 작업이 또 예외를 던지면 1차 예외가 가려지거나 보상 흐름이 중단됨. 정책:
1. catch 진입 즉시 1차 예외 `log.error` (ERROR)
2. 2차 작업은 *예외 안 던지는 의도 캡슐화 메서드*로 호출처가 try-catch 없이 평탄하게
3. 그래도 던지는 경우: 덜 중요 → `log.warn` + 1차 전파, 치명적 → `addSuppressed` Composite

**적용 예**:
- `PaymentApprovalAttemptService.failIfRequested` — "REQUESTED면 fail 처리, 아니면 skip" 의도 캡슐화. 호출처(`runPgCancel`)는 try-catch 없이 호출. race window에서 attempt가 이미 SUCCEEDED여도 PG cancel은 멈추지 않고 mark만 skip.
- `PaymentApprovalService.isCompensationRequired` — 보상 진행 여부 판단을 Payment Aggregate owner에 캡슐화. NaverPay adapter가 `paymentRepository`에 직접 접근하지 않음.

**왜 일반 원칙으로 명문화했나** — 세 가지 동기:

1. **다른 도메인에서도 같은 패턴이 나타날 것**:
   - 보상 catch 2차 예외 처리는 *payment 특화* 문제가 아니라 *외부 시스템 호출 + 보상 흐름*이 있는 모든 도메인에 적용 가능.
   - payment에서 처음 발견했지만 *향후 다른 외부 연동(이메일, SMS, 외부 API 등)에서도 같은 패턴* 예상.
   - 한 곳 임시 처방으로 두면 비슷한 상황마다 매번 새로 발견하고 다시 해결해야 함.

2. **MD 파일에 의도를 남기는 것 자체가 가치**:
   - 나중에 *해당 부분에 문제가 생겼을 때* "왜 과거에 이런 식으로 코드를 짰지?" 에 대한 답이 ADR에 남아있음.
   - 코드만 보면 의도가 안 보이고, 주석은 부분적. 정책 문서가 *결정의 컨텍스트*까지 보존.

3. **payment 도메인의 복잡성이 기록의 필요성을 강제했음**:
   - payment는 *너무 많은 상황을 고려*해야 함 — race window, 보상 흐름, 외부 PG 응답 다양성, 멱등성, 도메인 무결성 검증 등.
   - *단순 머리로만 기억하기엔 한계*가 있다는 인식.
   - → 기록이 *선택이 아니라 필수*. ADR-013은 그 인식의 산물.

**진화 경로**:
- ADR-012 직후: `failApproveAndCancelApprovedPayment` 내부 `failApprove`를 try-catch로 감싸는 **임시 처방** (ADR-D). catch 범위 너무 넓어 `PAYMENT_ATTEMPT_NOT_FOUND` 같은 의도치 않은 예외까지 삼키는 문제 식별.
- ADR-013: 일반 원칙 (1차 ERROR, 2차 WARN, 의도 캡슐화 메서드) 명문화.
- ADR-014: Payment 존재 체크로 race window 자체 축소 → ADR-D try-catch는 *보조 방어선*으로 격하.

### 7. `completeVerifiedApproval` catch 분기를 의미별 보상 메서드로 분리 (ADR-3 of payment-compensation-policy)

**기존 구조**:
- `catch(PaymentException)` + `catch(CustomException)` + `catch(Exception)` 3개 블록이 모두 `failApproveAndCancelApprovedPayment` 호출.
- *같은 보상 메서드가 다른 의미*로 쓰임 (금액 불일치 취소 / 중복 결제 취소 / 키 불일치 실패 처리 / 예상치 못한 예외 취소).

**현재 — 의미별 4개 dispatcher**:

```
compensateMerchantKeyMismatch(attempt)                              ← PgCanceller 없음
compensateAmountMismatch(attempt, amount, pgCanceller)              ← PgCanceller 있음
compensateDuplicatePayment(attempt, ex, pgCanceller)                ← PgCanceller 있음
compensateUnexpected(attempt, ex, code, msg, pgCanceller)           ← PgCanceller 있음
```

**분리의 두 가지 동기 — 둘 다 의식적**:

1. **변경 영향 범위 축소**:
   - 한 메서드 + 파라미터 분기 구조면 정책 변경 시 메서드 내부 if-else를 수정해야 함
   - 분리되어 있으면 *해당 메서드 하나만 수정*
   - 예: "duplicate payment일 때만 cancel reason을 다르게 보내자"가 생기면 `compensateDuplicatePayment`만 만지면 됨

2. **코드 구조로 도메인 의미가 드러남**:
   - `compensateMerchantKeyMismatch`만 *PgCanceller 파라미터를 받지 않음*. 시그니처를 읽는 순간 "이 케이스에서는 PG cancel을 안 한다"는 도메인 규칙이 *코드 자체로* 드러남.
   - 왜 그런가: merchantPayKey mismatch = PG가 우리가 발급한 키를 모름 = *PG 측에 결제 자체가 성립하지 않음* → cancel 호출 대상이 없음.
   - 기존 일률 호출 구조에서는 *메서드 본문 깊이 들어가서 if-else 추적*해야 이 규칙을 알 수 있었음. 분리 후에는 *함수 시그니처 한 줄*로 즉시 보임.
   - **도메인 규칙이 주석/문서가 아니라 메서드 시그니처에 박히는 가치**.

**Strategy 패턴 미채택**: PG 하나 + 보상 시나리오 4개 → Strategy 객체화는 over-design. 의미별 메서드 이름 + 시그니처로 시나리오 표현이 더 명확.

**부수 발견**: `compensateMerchantKeyMismatch`에서 PG cancel 부재가 *코드 구조로 드러남*. 일률 호출 구조에서는 숨겨졌던 의미 차이.

## 도메인 경계에서 배운 것

### DDD 이관 (payment-ddd-migration + naverpay-ddd-migration)

**payment core 먼저, naverpay 다음으로 분리한 이유**: 결제는 외부 PG 연동과 내부 결제 상태 반영이 함께 섞이기 쉬워 *core/provider 전환을 분리*해야 변경량과 리스크가 관리 가능. core 정리 후 provider를 같은 DDD 레이어로 맞추는 2-step.

**naverpay는 자체 도메인 엔티티가 없음** → `payment.domain` 사용. *독립 패키지로 유지하되 내부 구조만 4-layer로 정리*. PG가 둘 이상 늘어나기 전까지 인터페이스 도입 보류.

**Gateway는 PG 호출 + 응답 코드 매핑까지** — `NaverPayGateway`가 `NaverPayClient` 호출 + `NaverPayException` 처리 + 응답 코드(`NaverPayApproveCode`) → 도메인 코드(`PaymentAttemptFailCode`) 매핑까지 담당. *Gateway가 application layer에 절대 의존하지 않음* (역방향 의존 금지) → `failApprove`, `succeedCancel` 같은 attempt 상태 반영은 `NaverPayApprovalService`(application)에 유지.

**Result 타입 이중화**: Gateway → Service 전달용 `NaverPayApproveResult` (`naverpay.infrastructure.result`) vs Service → Controller 반환용 `NaverPayApproveResponse` (`naverpay.application.result`). 같은 정보의 표현이 *경계마다 다른 책임*을 가짐.

### find-first 패턴 통일 (ADR-011)

payment는 *find-first 6곳 중 3곳*을 차지 — `PaymentApprovalService`, `PaymentApprovalAttemptService`, `PaymentCancellationAttemptService`. Application/Adapter 어디서도 `DuplicateKeyException` catch 안 함. unique 위반 race는 `GlobalExceptionHandler` `DataAccessException` 부모 핸들러가 500 (`COMMON-500-2`)으로 처리.

**적용 조건** (정책 명문화):

1. 트랜잭션이 짧다 (race window 좁음)
2. 정상 흐름 동시 충돌 확률이 낮다 (사용자 입력 식별자 / idempotency key 기반 unique)

이 조건이 깨지면 try-save-catch 패턴이 더 적합. ADR-002의 `(member_id, idempotency_key)` unique 위반 fallback 재조회 로직도 본 정책으로 대체됨.

### 작업 분리 원칙

- **DDD 이관 ≠ legacy 삭제**: 같은 PR에 묶지 않음. 변경량 분리 + 리뷰 부담 분리.
- **흐름별 service 분리 후 도메인 메서드 통합**: APPROVE/CANCEL service를 먼저 나누면 *도메인 type 가드가 자연 강제*되어 도메인 메서드 통합이 따라옴. 순서가 거꾸로면 도메인 통합이 안 됨.
- **보상 정책 이동도 단계적**: ADR-012 (도메인 가드) → ADR-013 (catch 정책) → ADR-014 (Payment 존재 체크로 race 축소) → ADR-015 (보상 정책 application 이동). *각 단계가 그 단계의 문제만 해결*. 한 번에 깐 over-engineering이 아님.

### 라인 수 예측의 한계

`payment-compensation-to-domain` PRD에서 "330줄 → 150줄 이하" 예상했으나 실제 236줄. *삭제 라인만 집계하고 잔여 유틸/변환 코드 규모 누락*. → 향후 라인 수 예측은 "삭제/잔여/신설"로 세분화하거나 예측 자체를 생략하고 책임 이동을 검증 기준으로.

## 다시 본다면

- (사용자 작성 영역 — payment 도메인을 처음부터 짠다면 바꿀 것 / 그대로 갈 것. ADR-012의 type 가드를 처음부터 안 넣을 것인지, `PgCanceller` 콜백 대신 처음부터 `PaymentGateway` port로 갈 것인지, 보상 정책을 `PaymentApprovalService` 안에 그냥 두고 별도 `CompensationService`를 안 만들 것인지 등)

## 다음 단계 / 미해결

- **Issue #160** (refactor: PaymentApprovalService.isCompensationRequired REQUIRES_NEW 제거) — 결정 4번에서 본 자기 비판의 정식 등록.
- **Issue #174** (refactor: payment 도메인 Order↔결제 경계 재설계) — 결정 1번에서 발견. Order에서 merchantPayKey 분리, payment_reference 별도 테이블 검토 등.
- **PgCanceller → PaymentGateway 재검토** — 결정 5번에서 사용자가 "사실 Gateway가 좋아 보임"이라고 명시. PG 둘 이상 추가 시점이 아니어도 *도메인 의미 명확화* 측면에서 가치. Issue #174와 함께 검토하면 자연스러움.
- **ALREADY_CANCELED 모니터링 보강** — 결정 5번에서 발견. PG가 우리보다 먼저 취소한 비정상 상황이 *흔적 없이* 정상 처리됨. 운영 인지를 위한 log/메트릭 추가 필요.
- **`PAYMENT_AMOUNT_MISMATCH`(400) 세 원인 구분 메트릭** — 결정 1번에서 발견. PG 문제 / 악의적 사용자 / 우리 서버 산정 오류 세 가지가 합쳐져 있음.
- **ArchUnit 도입** — `PaymentAttempt.succeed`/`fail` 직접 호출 차단을 CI에서 강제. payment-attempt-service-split / payment-compensation-policy / payment-compensation-to-domain 회고 모두 동일하게 제안. 다른 도메인 아키텍처 테스트와 일관되게 도입 권장.
- **PaymentReference Value Object** — `merchantPayKey`가 `Payment`/`PaymentAttempt` 두 Aggregate 간 협력 키로 String 원시 타입으로 흐름. VO화하면 협력 경계가 타입으로 드러남. Payment 도메인 분리 시 함께 검토.
- **PaymentAttempt 상태 전이 표 문서화** — 현재 mark 메서드 내부 코드로만 표현. enum/별도 문서에 허용/거부 표 명시. Order 도메인 상태 전이 규칙과 함께 정리.
- **`isCompensationRequired` cancel skip 빈도 모니터링** — 정상 race 결과이지만 운영 빈도가 높으면 결제 흐름 이상 신호. 별도 메트릭 수집 권장.
- **`runPgCancel` 단계별 로그 수준 정비** — `isCompensationRequired == false` (정상 race)와 `cancelAttempt.status != REQUESTED` (이미 처리된 cancel)가 모두 log.warn. 의미가 다르므로 분리.
- **Payment 도메인 분리 시점** — 현 시점은 동일 backend 안. 분리 시 `isCompensationRequired`가 외부 API로 승격되고 `PaymentApprovalService`가 anti-corruption layer 역할 자연 수행.

## 인용

- `[[commerce-backend/docs/ADR.md#ADR-010]]` — PaymentAttempt amount mismatch 409
- `[[commerce-backend/docs/ADR.md#ADR-011]]` — find-first 패턴 통일 (DAO 예외 안전망)
- `[[commerce-backend/docs/ADR.md#ADR-012]]` — PaymentAttempt 상태 전이 도메인 검증
- `[[commerce-backend/docs/ADR.md#ADR-013]]` — 보상 catch 2차 예외 처리 일반 원칙
- `[[commerce-backend/docs/ADR.md#ADR-014]]` — Payment 존재 체크로 보상 판단
- `[[commerce-backend/docs/ADR.md#ADR-015]]` — 보상 정책 payment.application 이동, PgCanceller 콜백
- `[[commerce-backend/docs/architecture.md]]` — 결제 승인 데이터 흐름, payment 도메인 서비스 테이블
- `[[commerce-backend/docs/exception-strategy.md]]` — find-first / 보상 catch 2차 예외 / PgCanceller swallow
- `[[commerce-backend/docs/ddd/payment-ddd-migration-retrospective.md]]` — payment core DDD 이관
- `[[commerce-backend/docs/ddd/naverpay-ddd-migration-retrospective.md]]` — naverpay DDD 이관, Gateway 분리
- `[[commerce-backend/docs/tasks/payment-attempt-idempotency]]` — ADR-010 도입 task
- `[[commerce-backend/docs/tasks/payment-attempt-state-transition-policy]]` — ADR-012 도입 task
- `[[commerce-backend/docs/tasks/payment-attempt-service-split]]` — APPROVE/CANCEL service 분리 + 도메인 메서드 통합
- `[[commerce-backend/docs/tasks/payment-compensation-policy]]` — ADR-014 (Payment 존재 체크) task
- `[[commerce-backend/docs/tasks/payment-compensation-to-domain]]` — ADR-015 (보상 정책 이동, PgCanceller) task
- `[[raw/sessions/backend/2026-05-29-order-domain-overview]]` — Order 측 merchantPayKey 생명주기 + Issue #174 (도메인 경계 재설계)
