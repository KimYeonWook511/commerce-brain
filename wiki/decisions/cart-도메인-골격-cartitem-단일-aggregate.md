---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, aggregate, ddd, domain-model, order]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# 장바구니 도메인 골격 — CartItem 단일 entity aggregate

## 컨텍스트·문제 — 회원당 cart 1개, cart 메타데이터 부재

장바구니를 신규 도메인으로 도입할 때 aggregate 형태를 어떻게 잡을지가 첫 결정이었다. 회원당 cart는 1개이고, cart 자체에 붙는 메타데이터(쿠폰 슬롯·메모·상태·만료)가 현재도 예정에도 없다. "사용자의 cart"는 결국 `memberId`로 필터한 항목 리스트일 뿐이다.

## 검토한 대안 — Cart(root)-CartItem(N) 2계층 vs CartItem 단일 aggregate

- **Cart(root)-CartItem(N) 2계층**: DDD 교과서적인 형태지만, Cart entity를 만들어도 `(id, memberId, createdAt, updatedAt)`뿐인 빈 컨테이너가 된다. cart 레벨에 담을 상태가 없는데도 root 계층과 조인·수명주기 관리 비용만 얹힌다.
- **CartItem 단일 entity aggregate (채택)**: `CartItem(memberId, productId, quantity)` 하나만 두고 root를 CartItem 자신으로 한다. 기존 코드베이스의 `StockHistory`·`RefreshToken`이 이미 쓰는 단일 entity aggregate 패턴과 일관된다.

## 결정 — CartItem-only 단일 entity aggregate, root=CartItem 자신

빈 Cart 컨테이너를 만들지 않고 `CartItem` 단일 entity를 aggregate root로 삼았다. cart 레벨 상태가 없다는 현재 사실을 모델에 정직하게 반영한 것이다. cross-aggregate 참조는 객체 참조가 아니라 `memberId`·`productId`를 원시 `Long`으로 들고, 이는 [[cross-aggregate-fk-to-id-참조-컨벤션]]의 대표 적용 사례가 됐다.

## 가격 미저장 — 조회 시점 Product 재조립

`CartItem`은 `productId`·`quantity`만 갖고 가격을 저장하지 않는다. cart 조회 시점에 `productRepository.findAllById(productIds)`로 Product를 다시 읽어 최신 `price`·`name`·`imageUrl`·`status`를 응답에 조립한다. cart에 가격을 박아두면 stale해지고, 어차피 주문 흐름이 주문 시점에 Product 가격을 재검증하므로 cart의 가격은 신뢰할 수 없는 값이 된다. 조회는 PK 인덱스라 비용이 무시 가능하다.

## 주문 성공 시 cart 제거를 주문 트랜잭션 안에서 처리

주문이 성공하면 `OrderCreateProcessor`의 주문 저장 트랜잭션 **안에서** `CartItemRemover.removeByMemberAndProducts(memberId, productIds)`(내부적으로 `deleteByMemberIdAndProductIdIn`)를 호출한다. cart와 order가 같은 RDB라 커밋-후 이벤트로 분리할 이유가 없고, "주문은 성공했는데 cart는 그대로"인 일시 불일치를 애초에 만들지 않는다. 주문 요청 productId가 cart에 없어도 0 row 삭제로 자연 처리되므로 주문에 cart 존재 검증을 걸지 않는다 — "지금 구매"·재시도 같은 정상 흐름을 막지 않기 위해서다. 멱등 응답 경로는 `OrderCreateProcessor.execute`를 타지 않아 두 번 제거되지 않아 별도 가드가 불필요하다. 이 결합은 주문 멱등 처리([[주문-멱등성-캐싱-after-commit-이벤트-분리]])와 대비되는 지점이다 — cart 제거는 커밋-후 이벤트가 아니라 트랜잭션 내 처리로 일부러 갈랐다.

## 트레이드오프·확장 경로 — cartId 마이그레이션

단일 aggregate라 cart 레벨 메타데이터를 담을 자리가 지금은 없다. 향후 쿠폰·메모·만료 같은 cart 단위 상태가 필요해지면 Cart aggregate를 추가하고 `cartId`를 붙이는 마이그레이션으로 확장한다. 지금 없는 계층을 미리 만들지 않는 YAGNI 판단이고, 확장이 필요할 때의 경로가 명확해 부담이 낮다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
