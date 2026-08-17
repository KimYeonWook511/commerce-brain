---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, validation, stale-doc, rest, convention]
created: 2026-05-28
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# path productId 검증 — spec을 코드에 맞춤

## 컨텍스트·문제 — path id 무검증 vs spec 문구 drift

외부 AI 코드 리뷰(codex)가 PATCH/DELETE의 `@PathVariable Long productId`에 양수 검증이 없어 0/음수가 service까지 내려가 404로 응답한다고 지적했다. spec 문구("양수, 필수")와 어긋난다는 drift 지적이다.

## 검토 — @Positive 핸들러 신설 vs 코드베이스 일관 패턴 조사

처음엔 `@Validated + @Positive + ConstraintViolationException 핸들러 신설`을 제안했다. 그런데 사용자가 "다른 controller는 어떻게 하고 있어?"라고 물어 재조사했다. 결과: 코드베이스의 **모든 controller가 path id 검증을 하지 않고**, 0/음수가 내려와도 not-found로 일관 처리하고 있었다. cart만 `@Positive` 검증을 새로 도입하면 오히려 부분적 일관성이 생겨 부담이 된다.

## 결정 — 코드가 아니라 spec을 완화(정합성 양방향)

코드를 spec에 맞추는 대신, spec 문구를 "코드베이스 path id는 미존재 항목으로 4xx 응답하는 일관 정책"으로 완화했다. spec/코드 정합성 회복은 코드→spec, spec→코드 양방향으로 가능하다는 사례로 남겼다. 여기서는 코드베이스 전역 일관성이 개별 spec 문구보다 우선했다.

## 교훈 — 부분적 일관성 도입의 부담

한 곳(cart)에만 더 엄격한 검증을 도입하면, 나머지 controller와 다른 패턴이 되어 "왜 여기만 다른가"의 인지 부담과 유지보수 비용이 생긴다. "리뷰가 지적한 drift를 코드 강화로 없앤다"가 항상 정답은 아니다 — 코드베이스 전역 관행이 이미 일관되면 spec을 그 관행에 맞추는 것이 전체 정합성을 높인다. 이 판단은 리뷰가 짚은 stale/drift를 어느 방향으로 해소할지의 문제로, [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]의 리뷰 활용 주제와 이어진다. add 시점 Product 검증([[cart-add-product-존재-상태-검증]])과 함께 cart 입력 검증 체계를 이룬다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
