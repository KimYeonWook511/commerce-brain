---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [domain-factory, id-reference, jpa, test-fixture, order, payment, price-snapshot]
created: 2026-06-03
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]"
---

# 도메인 팩토리 시그니처를 Long ID로 전환 — 부가효과로 드러난 정책과 fixture 침투

cross-aggregate 연관을 끊은 뒤([[schema-무변경-decouple-series-메타원칙과-scope-규율]]) 자연히 따라온 도메인 팩토리 시그니처 재정비 결정. Order(#200)에서 시작해 Payment(#202)에서 반복됐고, 의도했던 것보다 부가효과가 더 값졌다.

## 컨텍스트 — 팩토리가 객체를 받을 이유 소멸

`@ManyToOne`/`@OneToOne`을 끊으면 도메인 팩토리에 Member/Product/Order 객체를 넘길 명목적 이유가 사라진다. 도메인이 받아봐야 할 수 있는 일이 없어 의미 없는 로드가 된다. 시그니처를 재정비할 자연스러운 시점이었다.

## 결정 — Long ID + 값으로 전환

- **Order:** `Order.create(Member)` → `Order.create(Long memberId, …)`, `order.addOrderItem(Product, int)` → `addOrderItem(Long productId, int quantity, int unitPrice)`.
- **Payment:** `Payment.createCompleted(order, …)` → `createCompleted(Long orderId, int amount, PaymentProvider provider, String merchantPayKey, String pgPaymentId, LocalDateTime approvedAt)`.

**검토한 대안 — 시그니처 유지(A):** application이 계속 Product/Order를 `findById`로 로드해 객체째 도메인에 넘긴다. 하지만 해제 후엔 도메인이 그 객체로 할 일이 (Order의 경우 `getTotalPrice()` 호출 하나뿐) 거의 없어 "외부 객체 의존 제거"라는 Long ID 전환의 취지를 절반만 달성하고, 단위 테스트 fixture도 Member/Product/Order entity를 계속 조립해야 한다. 기각. Long ID 시그니처(B)를 택해 도메인의 외부 객체 의존을 0으로 만들었다.

## 검증 조회 흐름은 유지 — 존재검증 전용 API 신설 안 함

ID로 바꿨어도 application의 존재 검증은 기존 `findById`/`findAllById` 흐름을 그대로 뒀다. 어차피 호출처가 같은 트랜잭션에서 `product.getPrice()`·`order.getTotalPrice()`·`order.completePayment()` 같은 객체 필드를 함께 쓰므로 객체 로드가 필요하기 때문이다.

- **Order에서 한 번 어겼다가 회수:** 초기에 "`findById` 대신 `exists`로 효율을 높이자"는 요청을 사용처 분석 없이 박아 `ProductRepository.existsById`를 신설했다. 구현·문서 동기·회고까지 끝낸 뒤 리뷰에서야 호출처가 0건임이 드러나 회수했다 — 모든 Order 생성 경로가 `product.getPrice()`를 `unitPrice` 인자로 넘겨 어차피 객체를 로드하므로 대체 사용처가 0. `grep "productRepository\." src/main` 한 줄이면 끝났을 일이었다.
- **Payment에서 같은 함정을 미리 차단:** `OrderRepository`에 존재검증 메서드를 신설하지 않았다(변경 0건). `findByMerchantPayKeyForUpdate`가 없으면 `ORDER_NOT_FOUND`를 던져 존재 검증이 이미 포함된다. "방어 검증을 위해 메서드를 신설"하는 함정이 두 번 연속 같은 모양으로 나왔고, 두 번째엔 선행 학습으로 바로 막았다. 미사용 API 회수 판단은 [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]]에서 일반화한다.

## 부가효과 — 정책의 표면화

의도치 않은, 그러나 더 의미 있던 소득. 이전엔 `Payment.createCompleted(order, …)` 내부에서 `amount = order.getTotalPrice()`가 호출됐다 — "Order의 totalPrice를 결제 시점 amount로 쓴다"는 **정책이 도메인 안에 숨어** 있었다. `amount`를 호출자가 명시적으로 넘기게 바꾸니(실제 `Payment.createCompleted(order.getId(), order.getTotalPrice(), …)`로 호출) 이 정책이 application 호출 라인에 드러난다. 본래 의도는 Long ID 일관성이었는데 이 표면화가 덤으로 값졌다. 이런 효과는 설계 단계에서 의도로 미리 적기보다 회고에서 사후 정리하는 게 정직하다. 정책·오염을 도메인 밖으로 밀어내는 이 결은 [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]]과도 닿는다.

관련해 Payment에서 Gemini가 `orderId`/`amount`에 null·범위 방어 검증을 권했으나 기각했다 — 이 코드베이스의 정적 팩토리(`Order`, `Stock`, `StockHistory`)는 range guard로 예외를 던지지 않는 게 일관 컨벤션이고, 호출처가 영속된 엔티티 필드를 넘기는 내부 호출(시스템 경계 아님)이라 발생 불가 시나리오다.

## 트레이드오프 — fixture 침투 면적

`Order.create(member)`·`addOrderItem(product, qty)`·`createCompleted(order, …)` 호출부가 order 도메인 밖 payment·cart 테스트까지 퍼져 있어 fixture 변경 면적이 컸다. cross-aggregate 객체 참조가 test fixture에서도 도메인 경계를 넘어 침투해 있던 결과다. Stock sub-PR의 builder 시그니처 침투와 같은 현상이고, 영향 면적을 잡을 땐 의존 도메인 test fixture까지 미리 포함해야 한다는 교훈. **부수 이득**으로는 Member/Product/Order entity 없이 ID만으로 도메인을 조립할 수 있어 이후 fixture 부담이 준다.

## 미해결 — OrderItem unitPrice 비대칭 (#201)

`addOrderItem(productId, qty, unitPrice)` 시그니처를 들였는데 정작 `OrderItem`에는 가격 컬럼이 없다. 넘어온 unitPrice는 `Order.totalPrice` 누적에만 쓰이고 그대로 휘발한다 — **결제 시점 가격 snapshot이 없다.**

- **왜 지금 안 고치나:** 가격 컬럼 신설은 Flyway migration이 필요해 series의 "schema 변경 0건" 메타원칙과 정면 충돌한다. Issue #201로 분리했고(추후 구현 트랙으로 이어짐, [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]] 참조).
- **왜 이슈로 떼나:** 도메인 메서드 인자가 entity 컬럼과 1:1로 안 맞으면 그건 빚이고, fixture가 `addOrderItem(1L, 1, 999)`처럼 productId와 unitPrice를 임의로 분리할 수 있어 코드 오용 표면이 넓어진다. ADR의 "후속 정비 항목" 문장만으로는 추적이 안 돼 이슈 등록이 더 강한 tracking 장치다.
- 관련해 Order 생성 시 Product soft delete 체크 부재가 의도인지 누락인지도 별개 검토 대상으로 남았다.

## 근거

- [[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]
