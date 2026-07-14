---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [ddd, aggregate, cross-aggregate-reference, foreign-key, convention, migration]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# 다른 aggregate는 Long ID로만 참조 — 신설 도메인 컨벤션

신설 도메인이 다른 aggregate를 참조할 때 객체 참조(`@ManyToOne`/`@JoinColumn`) 대신 원시 `Long` ID만 보유한다는 **컨벤션의 canonical 노트**다. cart([[cart-도메인-골격-cartitem-단일-aggregate]])에서 명문화됐고, payment/order 재설계에서 재확인됐다.

## 컨텍스트·문제 — 기존 객체 참조의 이중 표현·N+1·fetch join 부담

기존 도메인(`Order.member`, `OrderItem.product`, `Stock.product` 등)은 객체 참조를 광범위하게 쓴다. 그런데 application 계층은 실제로는 대부분 ID로 흐름을 다룬다. 그 결과 도메인과 인터페이스 사이에 같은 관계가 객체(entity)와 ID(Long)로 이중 표현되고, 조회 시 N+1·fetch join 튜닝 부담이 누적됐다. DDD가 원래 권하는 "다른 aggregate는 identity(ID)로만 참조한다" 원칙이 코드베이스에서 지켜지지 않고 있었다.

## 결정 — 신설 도메인은 다른 aggregate를 Long ID로만 참조

- **CartItem**은 `memberId`·`productId`를 원시 `Long`으로 저장하고 FK 객체 참조를 쓰지 않는다.
- **payment/order 재설계**에서도 같은 원칙을 확인했다: Payment는 소속을 `order_id`(다른 도메인의 PK 값)로만 표현하고 `FOREIGN KEY` 물리 제약을 걸지 않는다. type별로 참조 대상을 달리하지 않고 모든 Payment 행을 `order_id`로 통일했다.
- 이 컨벤션은 특정 phase에 국한하지 않고 이후 신설 도메인 전체에 적용한다.

## 트레이드오프 — FK 무결성 상실, application 검증이 유일 게이트

DB 참조 무결성을 FK가 보장하지 않는다. 대신 정합성은 (1) application 레벨 검증(참조 대상 조회 후 진행), (2) UNIQUE 제약, (3) 삭제 순서 정책이 나눠 책임진다. cart에서 이 트레이드오프가 곧바로 구체화된 것이 [[cart-add-product-존재-상태-검증]]이다 — FK가 없으니 "존재하지 않는 productId로 orphan row가 생기는" 결함을 막는 유일한 게이트가 add 시점 application 검증이 됐다.

FK 무결성 상실은 무제한 방임이 아니다. FK 제약은 도메인 간 결합을 강하게 만들어 마이그레이션·삭제 순서·테스트 픽스처에서 불편이 크다는 것이 재설계에서 재확인한 판단이고, 그 불편을 피하는 대가로 존재 정합성을 application으로 옮긴 것이다.

## 조회 비용 — findAllById PK 인덱스

객체 참조 대신 ID를 들면 연관 데이터를 별도 조회해야 한다(cart는 `productRepository.findAllById(productIds)`). 그러나 이는 PK 인덱스 조회라 비용이 무시 가능하다. `IN` 절 binding이 회원당 row 수만큼 커질 수 있다는 확장성 우려는 [[cart-회원당-row-상한-미도입]]에서 별도 tradeoff로 다뤘다.

## 적용 범위 — 신설 도메인 컨벤션 명문화, 기존 도메인 마이그레이션은 별도 트랙

- **신설 도메인**: 이 컨벤션을 기본값으로 강제한다.
- **기존 도메인(`Order.member` 등)의 `@ManyToOne`→ID 전환**: 별도 트랙으로 분리했다. 실제 마이그레이션 실행 전략은 [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]]로 위임한다. 이 컨벤션과 JPA 결합 끊기 series([[schema-무변경-decouple-series-메타원칙과-scope-규율]])는 같은 방향을 공유한다 — 이 컨벤션이 "왜 ID로 참조하나"의 원칙 근거라면, 결합 끊기 series는 그 원칙을 기존 코드에 소급 적용하는 실행이다.

FK 제거의 부산물도 주의할 지점이다: schema가 제공하던 참조 무결성이 사라지면 그 FK에 기대던 backfill·migration·데이터 분포 가정도 다시 봐야 한다 — [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]에서 FK 제거 뒤 INNER JOIN backfill이 리뷰에서 뒤집힌 것이 그 실례다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]
- [[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]]
- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]
