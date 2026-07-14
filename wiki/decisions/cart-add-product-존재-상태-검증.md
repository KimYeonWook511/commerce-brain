---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, validation, product, soft-delete, error-code]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# 장바구니 추가 시 Product 존재·상태 검증

## 컨텍스트·문제 — codex 지적, FK 없는 cart의 유일한 정합성 게이트

외부 AI 코드 리뷰(codex)가 `POST /cart/items`는 `productId`의 `@Positive`만 보고 실제 Product 존재를 확인하지 않는다고 지적했다 — 임의의 미존재 productId로도 201을 받고 orphan cart row가 생긴다. cart는 다른 aggregate를 FK 없이 `Long` ID로만 참조([[cross-aggregate-fk-to-id-참조-컨벤션]])하기로 한 결정 때문에, DB 참조 무결성이 없다. 따라서 데이터 정합성의 **유일한 게이트가 application 검증**이다. 이 노트는 그 컨벤션이 만든 트레이드오프(FK 무결성 상실 → application이 게이트)의 직접 귀결이다.

## 결정 — add 시점 Product 존재·상태 검증

`AddCartItemProcessor`가 row를 만들기 전에 `productRepository.findById`로 Product를 검증한다. add 시점에 걸러야 orphan row가 애초에 생기지 않는다.

## 에러 코드 설계 — CART-404-2(결함) vs CART-409/STOPPED(일시 차단) 분리

검증 실패를 성격에 따라 두 코드로 분리했다.

- 미존재 또는 soft-deleted → `CART_ITEM_PRODUCT_NOT_FOUND`(`CART-404-2`, 404). Product가 존재하지 않는 **결함** 상황.
- `STOPPED` → `CART_ITEM_PRODUCT_UNAVAILABLE`(`CART-409`, 409). 상품이 잠시 판매 중단된 **일시 차단** 상황.

결함과 일시 차단을 코드로 갈라야 운영에서 따로 추적할 수 있다. soft-delete를 미존재와 같은 결함으로 묶는 것은 [[product-soft-delete-deletedat-주문이력-보존]]의 "삭제된 상품은 신규 구매 불가" 성격과 일관된다.

## '처음부터 못 사는 상품' vs '담은 뒤 상태 변화'(unavailable 마킹) 구분

두 문제를 다르게 다뤘다.

- **처음부터 못 사는 상품을 새로 담는 동작**: add 시점에 차단(위 에러 코드). 애초에 담기지 않게 막는 게 자연스럽다.
- **이미 담아둔 항목이 STOPPED/soft-deleted가 됨**: 담긴 뒤의 상태 변화는 삭제하지 않고, 응답에 `unavailable=true`로 마킹해 보존·표시하는 별도 정책. 사용자가 담아둔 사실은 유지하되 구매 불가만 알린다.

## PATCH 미검증 이유

`PATCH`(수량 변경)에는 Product 검증을 두지 않는다. 기존 cart row의 수량 변경일 뿐이라, 상태 변화는 이미 `unavailable` 응답 마킹으로 운영 가시성이 확보돼 있다. 검증을 중복으로 걸 이유가 없다. 이는 add(신규 진입 게이트)와 patch(기존 row 수정)의 책임을 다르게 본 것으로, path id 검증을 코드베이스 일관 정책으로 완화한 [[cart-path-id-검증-spec을-코드에-맞춤]]과 함께 cart 입력 검증 체계를 이룬다. 에러 코드 체계(CART-404-1/404-2/409)는 [[cart-delete-미존재-4xx-entity-경유-삭제]]와 공유하며, 안정화되면 api-contract 노트로 승격할 후보다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
