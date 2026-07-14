---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, compensation, saga, pg-gateway, dependency-inversion, exception-handling, naverpay]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# 보상 흐름 설계 — 정책은 payment.application, PG는 cancel 콜백, 의미별 dispatcher 분리

> [!note] 05-29 스냅샷 결정
> [[payment-도메인-구조-개요]] 스냅샷 시점. 이후 exception/reconciliation 재설계의 보상 노트들([[결제승인완료-보상-완료우선-이중결제-adapter매핑]] 등)과 연결·보정된다.

## 컨텍스트·문제(보상 정책이 PG 서비스에 붙음)

`NaverPayApprovalService`가 보상 dispatcher 4개 + 공통 골격을 갖고 있었다. 보상 정책(어떤 실패 → cancel 필요/불필요, cancel reason, cancel amount)은 **PG-agnostic한 결제 도메인 책임**이고, PG-specific한 건 cancel API 호출·응답 해석뿐이다. 정책이 PG 서비스에 붙어 있으면 레이어 의존이 역전되고 PG 변경 시 정책 코드가 함께 영향받는다.

## 후보 비교(PgCanceller / PaymentGateway / Strategy)

보상 dispatcher 4개 + 공통 골격을 `PaymentApprovalCompensationService`(payment.application)로 옮기고, PG cancel은 콜백으로 위임하기로 하되 그 콜백 형태를 3택했다:

1. **`PgCanceller` 좁은 콜백 ✓** — `@FunctionalInterface cancel(PaymentAttempt, String) → CancelOutcome`. NaverPay가 `this::pgCancel` 메서드 참조로 구현. **인터페이스 추가 없이 의존 역전**하는 최소 구조. payment.application이 `NaverPayCancelResult`를 직접 import하지 않게 된다.
2. **`PaymentGateway` port 완전 inversion ✗** — PG-agnostic approve/cancel 통합 port. PG가 둘 이상 추가되면 자연스러우나 현 시점 over-engineering.
3. **Strategy 패턴 ✗** — PG별 보상 전략 객체. PG가 하나뿐인데 premature.

## 솔직한 도입 배경(AI 추천 PgCanceller 시도)

**본인의 선호는 `PaymentGateway` 완전 inversion**이다(Gateway가 도메인 의미를 더 잘 표현). 다만 이번엔 *AI가 추천한 `PgCanceller` 좁은 콜백*을 한 번 시도해본 학습 흔적이라, 지금 코드를 봐도 의도가 명확히 와닿지 않는다. 추측되는 의도는 NaverPay에 application이 직접 의존하지 않게 하는 의존성 역전. PG를 추가하기 전까지 역할이 모호하고, PG가 둘 이상 늘어나는 시점에 `PaymentGateway` port로 자연 승격하거나 다시 설계할 여지가 있다. (학습 단계라 새 추상화 도입을 최소화하는 판단은 [[paymentattempt-호출정책-문서-javadoc-archunit-보류]]와 같은 결.)

## CancelOutcome 변환·ALREADY_CANCELED→SUCCESS 매핑

PG-specific `NaverPayCancelResult.Status` → 도메인 `CancelOutcome.Status` 매핑은 `pgCancel` 콜백 안에서 이뤄진다. `ALREADY_CANCELED`는 `SUCCESS`로 매핑한다(PG 측이 이미 취소됨 = 우리 입장에선 cancel 목적 달성). `case SUCCESS, ALREADY_CANCELED -> CancelOutcome.success()`.

## 의미별 dispatcher 분리(시그니처로 도메인 규칙 표현)

기존엔 `catch(PaymentException)`·`catch(CustomException)`·`catch(Exception)` 세 블록이 모두 하나의 보상 메서드를 호출해 같은 메서드가 다른 의미로 쓰였다. 이를 의미별 4개 dispatcher로 분리했다:

```
compensateMerchantKeyMismatch(attempt)                    ← PgCanceller 없음
compensateAmountMismatch(attempt, amount, pgCanceller)    ← PgCanceller 있음
compensateDuplicatePayment(attempt, ex, pgCanceller)      ← PgCanceller 있음
compensateUnexpected(attempt, ex, code, pgCanceller)      ← PgCanceller 있음
```

분리의 두 동기(둘 다 의식적): (1) **변경 영향 범위 축소** — 정책 변경 시 해당 메서드 하나만 수정. (2) **코드 구조로 도메인 의미 노출** — `compensateMerchantKeyMismatch`만 *PgCanceller를 안 받는다*. PG가 우리가 발급한 `merchantPayKey`를 모름 = PG 측에 결제가 성립하지 않음 = cancel 대상 없음. 그 도메인 규칙이 함수 시그니처 한 줄로 즉시 드러난다. **도메인 규칙이 주석/문서가 아니라 메서드 시그니처에 박히는 가치.**

## 다시 본다면·미해결

- **PgCanceller → PaymentGateway 재검토** — 본인 선호는 Gateway 쪽(도메인 의미 명확). PG 둘 이상 추가 시점이 아니어도 가치가 있어 payment 미결 tradeoff로 남는다. Issue #174([[payment-order-도메인분리와-pg격리]])와 함께 검토하면 자연스럽다.
- **`ALREADY_CANCELED` 모니터링 부재** — PG가 우리보다 먼저 취소한 상태가 SUCCESS로 매핑되며 흔적 없이 정상 처리된다. 그러나 통신 이상/타이밍/외부 비정상 신호일 수 있어 운영 인지용 log/메트릭 필요.
- **인과 사슬** — 도메인 가드([[paymentattempt-상태전이-도메인-검증-defensive]]) → catch 정책 → Payment 존재로 race 축소([[보상판단-payment-존재-lock-대신-db-unique]]) → 보상 정책 application 이동. 각 단계가 그 단계의 문제만 해결하는 단계적 진화다. 보상 catch의 2차 예외 처리 원칙은 [[보상-catch-2차예외-평탄화-원칙]], DDD 이관 네이밍은 [[ddd-이관-컨벤션-adapter-command-query-네이밍]]과 연결.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
