---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 202 }
---

## 한 일

- Issue #195 (cross-aggregate JPA association 해제) 의 **세 번째이자 마지막 sub-PR** `payment-jpa-association-decouple`. PR #202 머지 → Issue #195 close → series (Stock #199 / Order #200 / Payment #202) 완료.
- `Payment.order (@OneToOne Order)` → `Long orderId` 전환. `Payment.createCompleted` 시그니처를 `(Long orderId, int amount, ...)` Long ID 패턴으로 정리.
- Gemini Code Assist review 1건 (createCompleted 의 orderId/amount 방어 검증 권장) reject 처리.

## 결정한 것

정본: `commerce-backend/docs/tasks/payment-jpa-association-decouple/adr.md`

본 세션의 *내 이해*:

- **ADR 결정 6개 중 절반이 인용으로 채워졌다**. 새 결정 3개 (association 해제 / Long ID 시그니처 / PaymentAttempt 범위 외) + 선행 PR 원칙 인용 3개 (fetch join 대체 정책 / schema 무변경 / 응답 echo 별도 트랙). 탐색 단계에서 `JpaPaymentRepository` 에 fetch join 0건, `PaymentAttempt` 이미 식별자 기반, `payment.getOrder()` main code traversal 0건임을 grep 으로 확인하니 결정해야 할 항목이 `createCompleted` 시그니처 1건으로 수렴.
- **`amount` 를 호출자가 명시적으로 전달하도록 한 부가 효과**: 이전에는 `Payment.createCompleted(order, ...)` 안에서 `order.getTotalPrice()` 가 호출돼서 "Order 의 totalPrice 를 결제 시점 amount 로 쓴다" 는 정책이 도메인 내부에 숨어 있었다. 시그니처 전환으로 이 정책이 `PaymentApprovalService.completeApprovedPayment` 의 호출 라인에 드러난다. 본래 의도는 Long ID 일관성이었지만, 이게 덤으로 따라온 효과가 더 의미 있었다.
- **`OrderRepository.existsById` 신설 안 함**: Order PR #200 에서 신설 시도 후 회수했던 학습을 그대로 따랐다. 호출처가 같은 트랜잭션에서 Order 의 다른 필드 (`completePayment()`, `getTotalPrice()`) 를 함께 쓰므로 신설 메서드의 사용처가 0건이 된다. "방어 검증 위해 메서드 신설" 의 함정 패턴이 두 번 연속 같은 모양으로 등장. memory 에 저장된 컨벤션 (sealed interface 같은 과한 추상화 금지) 과 같은 결의 판단.
- **Gemini review reject 근거**: `Order` / `Stock` / `StockHistory` 모두 정적 팩토리에서 `IllegalArgumentException` 류 null/range guard 두지 않는 게 일관된 컨벤션이고, 호출처 1곳에서 영속 entity 의 필드를 전달하므로 발생 불가능한 시나리오. CLAUDE.md "불필요한 추상화 피한다" 원칙 + 시스템 원칙 "Don't add validation for scenarios that can't happen — only validate at system boundaries".

다시 본다면:

- 탐색 단계에서 변경 면적을 grep 수치로 정량화한 게 Discuss 단계의 결정 비용을 크게 줄였다. 사용자가 처음 명시한 7개 정밀 조사 영역 (cross-aggregate 면적 / 보상 흐름 / concurrency / PG / 응답 DTO / aggregate 경계 / fixture 면적) 을 그대로 확인해 6개에서 "변경 없음" 결론 → 결정 1건으로 수렴. **다음 sub-PR series 의 마지막 PR 탐색은 "선행 PR 에서 정립된 원칙으로 커버되는 영역" 을 먼저 grep 으로 빼는 식으로 더 빨라질 수 있다.** 마지막 PR 은 새 결정 항목보다 "선행 원칙이 어디까지 커버하는가" 의 검증 단계에 가까움.

## 배운 것

- **도메인 시그니처 변환의 부가 효과는 의도 외에서 온다**: Long ID 일관성을 위해 시그니처를 바꿨더니 "정책이 코드 표면에 드러나는" 부가 효과가 따라왔다. `amount` 명시 인자가 결제 시점 정책 ("totalPrice = amount") 을 응용 계층으로 노출시켰다. 이런 효과는 ADR 단계에서 예측해서 의도로 적기보다, 회고에서 사후 정리하는 게 더 정직하다. 도메인 무관 일반 원칙.
- **LLM review 답변 패턴**: Gemini 같은 자동 review 가 일반론적으로 옳아 보이는 제안을 던지면 (1) 다른 도메인 entity 의 일관된 컨벤션 확인, (2) 호출처 신뢰 가능 여부 (system boundary 인가 internal 인가) 판단, (3) CLAUDE.md / 프로젝트 원칙 인용 — 이 세 가지를 reject 근거로 묶어 답변하면 일관된다. memory 에 저장된 "sealed interface 같은 과한 추상화 도입 금지" feedback 과 같은 결.

## 다음 단계 (지식 가치 있는 미해결)

- **PaymentAttempt 의 aggregate 경계 명시**: `Payment` 와 `PaymentAttempt` 가 `merchantPayKey` 를 공유 키로 결합되어 있는데, 이 결합이 ADR 어디에도 명시적으로 표현돼 있지 않다. 둘이 같은 aggregate 인지 별 aggregate 인지, `merchantPayKey` 가 도메인 개념인지 단순 식별자인지 — 본 series 의 정책 목적과 무관해 별도 트랙으로 미뤘다. 후속 정비 트랙이 아직 형식화되지 않음.
- **결제 시점 가격 snapshot 미해결 (Issue #201)**: Order PR #200 에서 `addOrderItem` 의 `unitPrice` 인자가 `OrderItem` 컬럼에 저장되지 않는 문제가 남았고, 본 PR 에서 `Payment.amount = order.getTotalPrice()` 를 명시 전달하게 됐지만 그 `totalPrice` 가 OrderItem 단가 누적인지 결제 시점 스냅샷인지는 여전히 모호. e-commerce 표준 (결제 시점 단가 snapshot 필수) 과 어긋남.
- **코드-schema 과도기 허용 정책 부재**: Stock/Order/Payment 의 다섯 FK (`fk_stock_product_id`, `fk_stock_history_stock_id`, `fk_order_member_id`, `fk_order_item_product_id`, `fk_payment_order_id`) 가 schema 에 남아있고 JPA 가 인식하지 않는 *과도기 상태* 다. 코드 association 해제와 schema FK 제거 사이의 lag 를 *언제까지 허용할지* 에 대한 정책이 어디에도 명시되지 않음. Flyway migration 일괄 발행은 실행 작업이지만, "과도기 lag 의 표준" 은 향후 다른 series 에서도 반복 등장할 결정 항목.
