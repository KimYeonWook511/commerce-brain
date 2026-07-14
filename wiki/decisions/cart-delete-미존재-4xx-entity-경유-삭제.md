---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, delete-semantics, rest, optimistic-lock, error-code, idempotency]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# DELETE 미존재 4xx + entity 경유 삭제

## 컨텍스트·문제 — 200+null 멱등 정책의 silent log 회귀 + PATCH와 비대칭

초기 구현은 `DELETE /cart/items/{productId}`가 미존재 시 `200 + null` 반환(멱등 정책)이었다. 두 문제가 있었다.

- **silent log 회귀**: `RemoveCartItemService`가 0 row여도 `log.info("장바구니 항목 삭제 ...")`를 찍어 운영 로그가 부정확해졌다.
- **PATCH와 비대칭**: PATCH는 미존재 시 4xx throw인데 DELETE만 200이라 정책이 어긋났다.

## 결정 — 미존재 시 CART-404-1 4xx로 통일, 멱등성보다 가시성 우선

DELETE도 미존재 시 `CART_ITEM_NOT_FOUND`(`CART-404-1`) 4xx로 통일했다. cart DELETE는 사용자 명시 액션이라 동일 DELETE 재요청 같은 멱등 시나리오가 거의 없다. REST DELETE의 멱등 성질을 약하게 깨더라도 명확한 피드백·운영 가시성을 우선했다. 이 에러 코드(CART-404-1)는 [[cart-add-product-존재-상태-검증]]의 CART-404-2/CART-409와 함께 cart 에러코드 체계를 이룬다.

## bulk @Modifying DELETE → entity 경유 delete 리팩터

구현은 처음 bulk `@Modifying DELETE`로 짰다. find→delete 사이 race window를 결정 문서의 "범위 제한" 단락으로 명문화하는 방식이었다. 그런데 사용자가 도메인 질문을 던졌다:

> "find해왔으면 해당 entity를 delete에 넘기면 되잖아 왜 벌크쿼리로 하는거야?"

이 한 마디로 `findByMemberIdAndProductId`로 managed entity를 로드한 뒤 `cartItemRepository.delete(cartItem)`로 리팩터했다. LLM은 옵션 비교(`clearAutomatically`를 붙일까)로 접근했는데, 사용자는 "왜 굳이 그렇게 짰지?"라는 도메인 질문으로 접근했다 — 이 일화는 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]에서 사용자 도메인 인사이트가 추상적 리뷰보다 강할 수 있다는 대표 예시로 인용된다.

## 한 번의 리팩터로 동시 해결된 세 가지(version race·clearAutomatically·문서)

entity 경유 delete로 바꾸자 세 가지가 한꺼번에 풀렸다.

1. **@Version race 처리**: entity를 통한 delete라 `@Version` 체크가 적용돼, 동시 DELETE race(다른 트랜잭션이 먼저 삭제했거나 add로 version을 올린 경우)에서 두 번째 트랜잭션이 `ObjectOptimisticLockingFailureException` → 409(`COMMON-409-1`)로 surface된다. race 시점의 silent log 회귀도 자연 차단.
2. **clearAutomatically 불필요**: bulk delete가 사라져, 리뷰에서 나온 `@Modifying(clearAutomatically=true)` 부착 권고(persistence context stale 회피)가 불필요해졌다.
3. **문서 단순화**: 결정 문서의 관련 단락이 "범위 제한"에서 "race 처리"로 단순해졌다.

## delete race 시 409(COMMON-409-1) 노출 — retry는 클라이언트 위임

`RemoveCartItemService`는 add/update 경로([[cart-동시성-낙관락-processor-분리-retry]])와 달리 retry loop가 없어 409가 클라이언트에 그대로 노출된다. delete의 `@Version` 낙관락은 add/update와 같은 메커니즘을 재사용하되, retry 책임은 클라이언트에 위임한다(재요청 시 보통 404 또는 정상 성공으로 정리됨). delete는 사용자 단발 액션이라 서버가 자동 재시도할 이득이 낮다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
