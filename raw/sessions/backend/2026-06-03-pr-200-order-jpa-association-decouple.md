---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 200 }
---

# Order·OrderItem cross-aggregate 연관 해제 — Long ID 전환과 fetch join 대체 패턴 첫 정립

기존 도메인의 JPA cross-aggregate 객체 참조를 aggregate 경계 단위 sub-PR 로 하나씩 Long ID 로 바꿔가는 트랙(Issue #195)의 두 번째 조각이다. 선행 Stock sub-PR(#199)에 이어, 이번엔 Order/OrderItem 이 다른 aggregate(Member·Product)를 `@ManyToOne` 객체로 붙들던 걸 끊는다. Stock 에는 fetch join 이 없어 선행 PR 이 의도적으로 미뤄둔 결정 — **cross-aggregate fetch join 을 무엇으로 대체할지** — 을 Order 에서 처음으로 정립하는 게 이 세션의 핵심 무게중심이다. PR 리뷰 두 라운드에서 불필요하게 신설한 조회 메서드를 회수하고, 새로 생긴 단가 비대칭을 후속 이슈로 떼어냈다.

## 결정한 것

### 1. cross-aggregate 만 Long ID 로, same-aggregate 는 객체 참조 유지

`Order.member`(`@ManyToOne Member`) → `Long memberId`, `OrderItem.product`(`@ManyToOne Product`) → `Long productId` 로 전환했다. 반면 같은 aggregate 안의 `Order.orderItems`(`@OneToMany`)와 `OrderItem.order`(부모 `@ManyToOne`)는 그대로 뒀다.

- **가르는 기준은 lifecycle 결합.** Order↔OrderItem 은 `cascade ALL` + `orphanRemoval` 로 생명주기가 한 몸이다 — Order 가 지워지면 OrderItem 도 함께 지워지고, OrderItem 은 Order 없이 독립 존재하지 않는다. 이건 신규 도메인에도 통일해 쓰는 "같은 aggregate 내 root-child 는 객체 참조 허용" 원칙에 정확히 해당해서 유지했다. 여기까지 ID 로 끊으면 오히려 cart 같은 신규 도메인의 일관성 모델과 어긋난다.
- **Member·Product 는 Order 의 생명주기와 독립.** 회원이 탈퇴해도 주문 이력은 남아야 하고, 상품이 판매 중단돼도 기존 OrderItem 이력은 유지된다. 객체 그래프로 traverse 할 이유가 없고 ID 참조로 충분하다.
- **DB 스키마·FK 는 손대지 않는다.** `member_id`·`product_id` 컬럼은 이미 있어서 JPA 매핑만 끊으면 스키마 변경 0건으로 완결된다. `fk_order_member_id`·`fk_order_item_product_id` FK 제약은 스키마에 그대로 남고 JPA 가 더 이상 인식하지 않을 뿐. Hibernate `validate` 는 컬럼 단위 검증이라 `@Column(name="member_id", nullable=false)` 유지만으로 통과한다(선행 Stock sub-PR 에서 동일 패턴이 Testcontainers MySQL `integrationTest` 로 검증됨, 이번에도 반복 확인). FK 일괄 제거는 series 의 모든 sub-PR 완료 후 별도 트랙으로 미룬다는 게 이 series 의 "스키마 변경 0건" 메타 원칙.

### 2. fetch join 대체 — 단일 원칙 대신 사용처별 분석

`JpaOrderRepository` 의 cross-aggregate fetch join 을 어떤 패턴으로 대체할지가 이번 sub-PR 이 새로 정한 결정이다. 결론은 **일반 원칙 두 줄만 공통으로 두고 그 아래는 호출처에 맡긴다**: same-aggregate fetch(`join fetch o.orderItems`)는 유지, cross-aggregate fetch(`join fetch o.member`·`join fetch oi.product`)는 전부 제거.

- **사용처가 네 경로였고 필요한 데이터 양상이 제각각이었다.** `PaymentReadyService` 는 결제창에 상품명(productName) 노출이 필요하고, `OrderCancelService`·`OrderExpirationService` 는 stock 복원에 productId 만 있으면 되며, `OrderQueryService` 가 쓰는 `findByMerchantPayKeyAndMemberId` 는 반환 데이터에 cross-aggregate 가 아예 없었다.
- **검토한 단일 원칙 세 가지를 다 기각했다.** (A) 모든 조회를 JPQL DTO projection 으로 통일 — cancel/expiration 경로에까지 productName 을 끌어와 응답 매핑에 불필요한 부담을 준다. (B) 모든 조회를 batch composition 으로 통일 — 마찬가지로 productId 만으로 충분한 경로에 추가 쿼리를 강제한다. (C) read 전용 QueryService 분리로 통일 — 단순한 cancel/expiration 경로까지 read 모델을 새로 만들어야 해 과한 추상화. 어느 쪽이든 **단일 원칙은 일부 호출처에 비효율을 강제**한다는 게 기각 이유다.
- **택한 것: 사용처별 후처리 분리.** cancel/expiration 은 OrderItem 컬럼에 이미 productId 가 있어 cross-aggregate 추가 조회 0회로 끝난다. PaymentReady 에만 OrderItem 의 productId 목록을 모아 `productRepository.findAllById(productIds)` 1회로 productName Map 을 만들고 응답 DTO 에 외부 주입한다(`from(order, productNameByProductId)`). 메서드를 둘로 쪼개는 것보다 호출처별 후처리를 나누는 게 깔끔하다는 판단.
- **`findByMerchantPayKeyAndMemberId` 는 join 자체를 없앴다.** cross-aggregate 데이터가 필요 없는 이 경로는 where 보조용 `join o.member` 를 제거하고 `where o.memberId = :memberId` 컬럼 비교로 단순화했다.
- **트레이드오프(성능):** 기존엔 `join fetch oi.product` 로 한 쿼리에 OrderItem+Product 를 함께 로드했지만, 이제 PaymentReady 는 Order+OrderItems 조회 1회 + Product batch 조회 1회로 round-trip 이 1회 늘어난다. 다만 단일 주문의 OrderItem 개수가 보통 한 자릿수라 `IN` 절 1회는 hot path 영향이 미미하고, 오히려 예전 fetch join 이 만들던 OrderItem×Product cartesian product 를 피하는 이득도 있다. 트랜잭션 범위는 그대로.

이로써 fetch join 대체의 일반 원칙이 처음 명문화됐고, 후속 Payment sub-PR 은 이 두 줄을 그대로 따르되 Payment 도메인의 사용처별 양상만 다시 분석하면 된다.

### 3. 도메인 팩토리 시그니처를 Long ID 로 전환

`Order.create(Member)` → `Order.create(Long memberId, …)`, `order.addOrderItem(Product, int)` → `addOrderItem(Long productId, int quantity, int unitPrice)` 로 바꿨다.

- **왜 지금:** `@ManyToOne` 을 끊으면 도메인 팩토리에 Member/Product 객체를 넘길 명목적 이유가 사라진다. 도메인이 받아봐야 할 수 있는 일이 없어 의미 없는 로드가 된다. 시그니처를 재정비할 자연스러운 시점.
- **검토한 대안:** (A) 시그니처 유지 — application 이 계속 Product 를 `findById` 로 로드해 도메인에 넘긴다. 해제 후엔 도메인이 그 객체로 할 일이 없고 단위 테스트 fixture 도 Member/Product entity 를 계속 만들어야 해 기각. (B) Long ID 시그니처 채택.
- **핵심 판단 — 검증 조회 흐름은 그대로 둔다:** ID 로 바꿨어도 application 의 product 존재 검증은 기존 `findById`/`findAllById` 흐름을 유지했다. 어차피 호출처가 같은 트랜잭션에서 `product.getPrice()` 같은 객체 필드를 함께 쓰므로 객체 로드가 필요하다. 그래서 검증 전용 조회 API 를 신설하지 않고 `ProductRepository` 인터페이스는 그대로 뒀다(이 판단을 처음엔 어겼다가 회수했다 — "막힌 점" 참조).
- **부수 이득:** unit test fixture 부담이 준다. Member/Product entity 없이 ID 만으로 Order 를 조립할 수 있다.
- **트레이드오프(변경 면적):** `Order.create(member)`·`addOrderItem(product, qty)` 호출부가 order 도메인 밖 payment·cart 테스트까지 퍼져 있어 fixture 변경 면적이 컸다. cross-aggregate 객체 참조가 테스트 fixture 에서도 도메인 경계를 넘어 침투해 있던 결과다. 선행 Stock sub-PR 과 같은 현상이고 후속 Payment sub-PR 에서도 같은 확산이 예상된다.

### 4. PaymentReady 상품명 조립에 null 가드 추가 (리뷰 반영)

상품명을 만드는 `buildProductName` 이 예전엔 `items.get(0).getProduct().getName()` 이었다 — Product 가 LAZY 연관이라 Hibernate 가 자동 로드했고, 행이 사라졌으면 프록시 해석 자체가 예외를 던지는 자동 안전망이 있었다. 이번에 이걸 명시적 `productsById.get(productId)` Map 조회로 바꾸면서 없는 상품이면 `null` 이 나올 수 있게 됐고, AI 코드 리뷰(gemini)가 여기에 가드가 필요하다고 지적했다. 결과적으로 `firstProduct == null` 이면 `ProductException(PRODUCT_NOT_FOUND)` 을 던지도록 넣었다(reject).

- **정상 흐름에선 이 null 이 날 수 없다.** soft delete 된 상품은 Map 에 여전히 포함되고, hard delete 는 코드 경로와 FK 제약이 막는다. 즉 방어 코드 없이도 정상 흐름은 안전하다는 게 원저자 판단.
- **그럼에도 reject 를 넣은 이유 — 정책 일관성:** unique 위반 등 예외 상황을 개별 방어 코드로 처리하지 않고 공통 안전망(500)에 위임하되 정상 흐름만 사전 조회로 다룬다는 find-first 안전망 정책 위에서, 예기치 못한 상태를 조용히 통과시키기보다 명시적 예외로 끊는 게 일관된다.
- **결정적 근거 — 코드베이스의 실제 패턴:** `OrderCreateProcessor` 가 이미 똑같은 `findAllById` + `Map.get` 패턴에 똑같은 형태의 가드(`product == null` → `ProductException(PRODUCT_NOT_FOUND)`)를 쓰고 있었다. 이번 PR 이 LAZY 자동 안전망을 명시 Map 조회로 바꾼 변경 의도와, 같은 패턴엔 같은 가드를 둔다는 코드베이스 일관성이 맞물려 가드를 채택했다.
- **내 이해:** 단일 정책 문서(ADR)만으로 모든 사례를 정당화하지 말 것. 방어 가드를 둘지 말지 같은 판단에선, "코드베이스의 다른 곳이 같은 패턴을 어떻게 처리했나" 가 단일 ADR 보다 더 큰 정보를 준다. 정책과 실제 패턴이 충돌하면 후자를 먼저 본다.

### 5. OrderItem unitPrice 비대칭 — 후속 이슈(#201)로 분리

이번 PR 이 `addOrderItem(productId, qty, unitPrice)` 시그니처를 들였는데, 정작 `OrderItem` 에는 가격 컬럼이 없다. 그래서 넘어온 unitPrice 는 `Order.totalPrice` 누적에만 쓰이고 그대로 휘발한다 — 결제 시점 가격 snapshot 이 없다.

- **왜 지금 안 고치나:** 가격 컬럼 신설은 Flyway migration 이 필요해 이 series 의 "스키마 변경 0건" 메타 원칙과 정면 충돌한다. 그래서 별도 이슈 #201 로 떼어냈다(추후 PR #204 로 구현·머지됨). `OrderItem.java` 의 "가격도 넣어야 하나?" 미해결 주석도 그 트랙에서 함께 결정한다.
- **내 이해:** 시그니처 전환 sub-PR 이 새로운 비대칭을 낳을 수 있다. 도메인 메서드 인자가 entity 컬럼과 1:1 로 안 맞으면 그건 빚이고, fixture 가 `addOrderItem(1L, 1, 999)` 처럼 productId 와 unitPrice 를 임의로 분리할 수 있어 코드 오용 표면이 넓어진다. ADR 의 "후속 정비 항목" 문장만으로는 추적이 안 된다 — 이슈 등록이 더 강한 tracking 장치다.

## 막힌 점·해결

### 쓰지도 않을 조회 메서드를 신설했다가 사후 회수

초기 사용자 요청("`findById` 말고 `exists` 같은 메서드를 만들어서 효율을 조금 높여보")을 **사용처 분석 없이** 도메인 시그니처 결정(위 3)에 그대로 박아 `ProductRepository.existsById` 를 신설했다. 구현 → 루트 문서 동기화 → 회고까지 다 끝낸 뒤 리뷰 단계에서야 호출처가 0건임이 드러나 회수했다.

- **회수 이유:** 모든 Order 생성 경로가 `product.getPrice()` 를 `addOrderItem` 의 unitPrice 인자로 넘긴다. 즉 어차피 객체를 로드해야 하므로 `existsById` 로 대체 가능한 사용처가 0건이다. 코드베이스 컨벤션("사용하지 않는 코드 추가 금지")과 정면 충돌.
- **정리 면적:** 코드 파일뿐 아니라 이 태스크 문서 5개 + 루트 ADR 색인까지 되돌려야 했다. 성격이 다른 변경이라 코드 되돌림(refactor) 커밋과 문서 정리(docs) 커밋으로 분리했다.
- **다시 본다면:** 설계 단계에서 `grep "productRepository\." src/main` 한 줄이면 끝났을 일이다. "`findById` 는 모든 컬럼 SELECT 라 비효율" 이라는 정적 분석만으로 결정하면, 실제 동적 호출 흐름에선 객체 필드가 어차피 필요하다는 사실을 놓친다. 사용자 결정이라도 사용처 확인 뒤에 적용해야 한다.

## 배운 것

- **사용처별 분석을 단일 원칙보다 우선.** fetch join 대체에서 정립했고, 뒤이은 Payment sub-PR 에도 같은 정신을 적용한다.
- **신설 API 의 정당성은 사용처가 입증한다.** 사용처 0건 = 신설 보류. 정적 분석만으로 결정 금지. `grep` 한 줄이 ADR·구현·회고를 사후에 되돌리는 것보다 압도적으로 싸다.
- **리뷰는 self-review 의 보완.** 일반 패턴을 짚는 AI 리뷰(gemini)와 코드베이스 일관성을 파는 subagent 독립 리뷰가 서로 다른 발견(null 가드 / 미사용 메서드·단가 비대칭)을 줬다. 두 라운드 리뷰가 단일 PR 의 빚을 명시적으로 끌어올렸다.
- **방어 가드 vs 정책의 균형엔 단일 답이 없다.** 코드베이스 다른 곳의 실제 패턴이 단일 ADR 보다 의사결정에 더 큰 정보를 줄 수 있다.

## 미해결·열린 질문

- **Payment sub-PR 로 이어진다:** 여기서 정립한 일반 원칙(same-aggregate fetch 유지 / cross-aggregate fetch 제거)을 Payment 도메인에 적용한다. Payment 의 fetch join 사용처별 양상 분석이 필요하다.
- **OrderItem 결제 시점 가격 snapshot(#201):** 컬럼 신설 vs Money VO 도입 결정, backfill 정책, 응답 DTO 영향 면적을 후속 트랙에서 정한다.
- **FK 일괄 제거 트랙:** series 의 "스키마 변경 0건" 원칙의 마무리 — 모든 sub-PR 머지 후 Flyway 한 개로 FK 제약을 일괄 제거한다.
- **Order 생성 시 Product soft delete 체크 부재:** 이게 의도된 결정인지 그냥 빠진 건지는 별개 검토가 필요하다. 사용자가 "그런 제약은 따로 정해야지만 존재" 라고 언급해 후자(누락) 가능성을 시사했다.
