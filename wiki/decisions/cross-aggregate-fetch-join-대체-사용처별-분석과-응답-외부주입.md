---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [fetch-join, jpa, query, batch-composition, dto-projection, response-dto, cross-aggregate, order]
created: 2026-06-03
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]"
---

# cross-aggregate fetch join 대체 — 단일 원칙 대신 사용처별 분석 + 응답 DTO 외부 주입

객체 참조를 끊으면 깨지는 cross-aggregate fetch join을 무엇으로 대체할지, Order sub-PR(#200)에서 처음 정립한 결정. [[schema-무변경-decouple-series-메타원칙과-scope-규율]]이 Stock에서 이 결정을 의도적으로 미뤄뒀고(Stock에는 fetch join 사용처가 없어서), Order에서 실제 사용처를 보고 정했다.

## 컨텍스트 — 연관 해제로 깨지는 fetch join

`JpaOrderRepository`에는 cross-aggregate fetch join(`join fetch o.member`·`join fetch oi.product`)이 있었다. Order/OrderItem이 Member·Product를 `@ManyToOne` 객체로 붙들던 걸 `Long memberId`/`Long productId`로 끊으면 이 fetch join들이 깨진다. 무엇으로 대체할지가 이 sub-PR의 무게중심이었다.

## 검토한 단일 원칙 3안 — 전부 기각

일관성을 위해 하나의 원칙으로 모든 조회를 통일하는 안들을 먼저 놓았다.

- **(A) 모든 조회를 JPQL DTO projection으로 통일:** cancel/expiration 경로에까지 productName을 끌어와 응답 매핑에 불필요한 부담을 준다.
- **(B) 모든 조회를 batch composition으로 통일:** productId만으로 충분한 경로에도 추가 쿼리를 강제한다.
- **(C) read 전용 QueryService 분리로 통일:** 단순한 cancel/expiration 경로까지 read 모델을 새로 만들어야 해 과한 추상화.

셋 다 **일부 호출처에 비효율을 강제**한다는 게 공통 기각 이유다. 사용처가 네 경로였고 필요한 데이터 양상이 제각각이었다 — `PaymentReadyService`는 결제창에 상품명이 필요하고, `OrderCancelService`·`OrderExpirationService`는 stock 복원에 productId만 있으면 되며, `findByMerchantPayKeyAndMemberId`는 반환 데이터에 cross-aggregate가 아예 없었다.

## 결정 — 공통 두 줄 + 사용처별 후처리 분리

공통 원칙은 딱 두 줄만 둔다: **same-aggregate fetch(`join fetch o.orderItems`)는 유지, cross-aggregate fetch는 전부 제거.** 그 아래는 호출처에 맡긴다.

- **cancel/expiration:** OrderItem 컬럼에 이미 productId가 있어 cross-aggregate 추가 조회 0회로 끝난다.
- **PaymentReady만:** OrderItem의 productId 목록을 모아 `productRepository.findAllById(productIds)` 1회로 productName Map을 만들어 응답 DTO에 외부 주입한다(`from(order, productNameByProductId)`). 메서드를 둘로 쪼개는 것보다 호출처별 후처리를 나누는 게 깔끔하다는 판단.
- **`findByMerchantPayKeyAndMemberId`는 join 자체 제거:** cross-aggregate 데이터가 필요 없으므로 where 보조용 `join o.member`를 없애고 `where o.memberId = :memberId` 컬럼 비교로 단순화했다.

이로써 fetch join 대체의 일반 원칙이 처음 명문화됐고, 이 원칙은 Order 태스크 문서에서 단일 관리한다. "사용처별 분석을 단일 원칙보다 우선"하는 이 정신은 [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]]와 결이 같다.

## 응답 DTO 외부 컨텍스트 명시 주입 패턴

application 계층이 엔티티 + 외부 컨텍스트를 명시적으로 조립해 응답 DTO를 만드는 패턴이 series를 관통했다.

- **Stock에서 시작 — `from(history, productId)`:** 연관 해제로 `history.getStock().getProduct().getId()` traversal이 불가능해지자, 응답의 productId를 세 안(필드 제거 / 컬럼 신설 / 외부 주입) 중 외부 주입으로 살렸다. StockHistory aggregate의 invariant는 `stockId`이고 productId는 audit row의 본질이 아니라 현재 조회 endpoint의 path 컨텍스트일 뿐이라, application이 `from(history, productId)`로 명시 조립하는 의도가 시그니처에 그대로 드러난다.
- **Order에서 확장 — `from(order, productNameByProductId)`:** 같은 패턴이 Map 주입으로 확장됐다.

이 패턴은 [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]]의 "정책·컨텍스트를 코드 표면으로 드러낸다"는 흐름과 짝을 이룬다.

## null 가드 — LAZY 자동 안전망을 명시 조회로 바꾸며 생김

`buildProductName`이 예전엔 `items.get(0).getProduct().getName()`이었다 — Product가 LAZY 연관이라 행이 사라졌으면 프록시 해석이 예외를 던지는 자동 안전망이 있었다. 이걸 명시적 `productsById.get(productId)` Map 조회로 바꾸면서 없는 상품이면 `null`이 나올 수 있게 됐고, AI 리뷰(gemini)가 가드를 지적했다. `firstProduct == null`이면 `ProductException(PRODUCT_NOT_FOUND)`를 던지도록 넣었다.

- **정상 흐름에선 이 null이 날 수 없다:** soft delete된 상품은 Map에 여전히 포함되고, hard delete는 코드 경로와 FK 제약이 막는다.
- **그래도 넣은 이유 — 코드베이스 실제 패턴:** `OrderCreateProcessor`가 이미 똑같은 `findAllById`+`Map.get` 패턴에 똑같은 가드를 쓰고 있었다. 단일 ADR보다 코드베이스의 실제 패턴이 방어 가드 판단에 더 큰 정보를 준다는 판단([[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]]).

## 트레이드오프 — round-trip 1회 증가

기존엔 `join fetch oi.product`로 한 쿼리에 OrderItem+Product를 함께 로드했지만, 이제 PaymentReady는 Order+OrderItems 조회 1회 + Product batch 조회 1회로 round-trip이 1회 늘어난다. 다만 단일 주문의 OrderItem 개수가 보통 한 자릿수라 `IN` 절 1회는 hot path 영향이 미미하고, 오히려 예전 fetch join이 만들던 OrderItem×Product cartesian product를 피하는 이득이 있다. 트랜잭션 범위는 그대로.

## Payment sub-PR — fetch join 0건, 원칙 인용만

`JpaPaymentRepository`에는 fetch join JPQL이 0건이고 derived query(`findByMerchantPayKey`, `existsByMerchantPayKeyAndStatus`)만 있어, Payment는 이 원칙을 **새로 정하지 않고 인용만** 했다. 응답 echo 정리도 Payment 응답(`NaverPayApproveResponse`)이 자기 필드만으로 조립돼 외부 주입 패턴을 적용할 사용처가 없었다. series의 마지막 PR이 "선행 원칙이 어디까지 커버하나"를 검증하는 자리였다는 사례.

## 근거

- [[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]
