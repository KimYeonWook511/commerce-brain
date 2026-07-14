---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, concurrency, optimistic-lock, retry, transaction, processor-pattern, race-condition]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# 장바구니 동시성 — 낙관락 + Processor 분리 + retry

## 컨텍스트·문제 — update race / insert race 구분

cart 항목 추가·수정의 경쟁 조건을 리뷰에서 지적받았다. 핵심은 경쟁을 두 종류로 나눠 본 것이다.

- **update race**: 같은 `(memberId, productId)` row에 동시 add 요청 둘 → 두 트랜잭션이 같은 quantity를 fetch한 뒤 각자 `increaseQuantity` → 마지막 commit만 반영. 합산 손실 + `98+1+1` 같은 상황에서 99 상한이 silent로 우회될 수 있다.
- **insert race**: cart에 없는 productId에 동시 첫 add 요청 둘 → 둘 다 `Optional.empty()` 분기에서 `save` 시도 → `uk_cart_item_member_product` UNIQUE 위반으로 한쪽 500.

같은 양상이 `UpdateCartItemQuantityService`(`changeQuantity`)에도 있다.

## 검토한 대안 — (A)비관락 (B)낙관락+retry [채택] (C)atomic UPDATE (D)insert-first

- **(A) 비관락 `@Lock(PESSIMISTIC_WRITE)`** — 재고 차감이 쓰는 패턴([[재고차감-동시성-비관락과-productid-정렬]])과 형식은 같지만, cart는 `(memberId, productId)` 단위라 contention이 사실상 0이다. 매 요청 row lock overhead가 이득 없이 비용만 남고, update race만 풀고 insert race는 여전히 안전망에 별도 위임해야 한다. 기각.
- **(B) 낙관락 `@Version` + retry (채택)** — entity에 `@Version` 추가, UPDATE 충돌이 `ObjectOptimisticLockingFailureException`으로 전파, 충돌 시에만 retry. contention이 0에 가까워 평균 attempt가 1로 수렴하고 평상시 lock overhead가 없다. 도메인 메서드(`increaseQuantity`, `changeQuantity`, `validateQuantity`)를 그대로 써 invariant가 도메인에 단일 표현된다.
- **(C) atomic UPDATE 쿼리** (`SET quantity = quantity + :delta WHERE ... AND quantity + :delta <= 99`) — 단일 SQL로 가장 가볍지만, 99 상한 invariant가 SQL과 도메인에 이중 표현된다. `@Modifying`이 dirty checking을 우회해 `updated_at` 자동 갱신이 깨지고, affected rows=0 분기 후속 SELECT + `clearAutomatically=true` stale 방지가 필요하다. 기각.
- **(D) insert-first → UNIQUE catch → addQuantity fallback** — 새 사용자의 find 호출을 아끼지만, UNIQUE 식별이 Hibernate `ConstraintViolationException.getConstraintName()` 등 구현체-specific에 의존해 fragile하다. `@Transactional` 안에서 unchecked 예외가 던져지면 트랜잭션이 rollback-only로 마킹돼 같은 트랜잭션에서 두 번째 작업이 불가하다. `REQUIRES_NEW`나 빈 분리가 필요해 결국 (B)와 비슷한 구조가 되고 함정만 추가. 기각.

## 결정 이유 — contention≈0, 도메인 invariant 단일 표현, 코드베이스 일관성

낙관락을 택한 세 근거: (1) cart는 (memberId, productId) 단위라 contention이 0에 가까워 낙관락의 "충돌 시에만 비용" 특성이 최적이다. (2) 도메인 메서드를 그대로 써서 99 상한 같은 invariant가 도메인 한 곳에만 표현된다. (3) Order·Stock이 이미 같은 `@Version` + retry 패턴(`StockConcurrencyService.decreaseWithOptimisticLock`)을 써 코드베이스가 일관된다. 이 일관성은 [[order-version-낙관락-비관락-공존]]·[[재고차감-동시성-비관락과-productid-정렬]]과 같은 어휘를 공유한다.

## Processor 분리 — self-invocation 회피 + method-level 트랜잭션 컨벤션

retry loop와 트랜잭션 경계를 별도 빈으로 갈랐다.

- **outer Service**(`AddCartItemService`, `UpdateCartItemQuantityService`)는 어노테이션 없이 retry loop만 담당(`MAX_RETRY = 3`).
- **트랜잭션 경계**는 별도 빈(`AddCartItemProcessor`, `UpdateCartItemQuantityProcessor`)의 method-level `@Transactional`이 책임진다.

같은 빈 안에서 `@Transactional` 메서드를 self-invocation하면 프록시를 안 타 트랜잭션이 안 걸리는데, 빈을 분리하면 retry attempt마다 빈 경계를 넘어 새 트랜잭션·새 persistence context로 진입한다. 이 세션에서 응용 Service의 `@Transactional`을 class-level 기본값 대신 method-level에만 붙이는 컨벤션을 새로 명문화했고, Processor 패턴이 그 컨벤션의 실물이다.

## retry 대상 한정 — OOLFE만, DataIntegrityViolation은 안전망 위임

retry catch 대상은 `ObjectOptimisticLockingFailureException`(update race)만이다. insert race의 `DataIntegrityViolationException`은 retry에 넣지 않고 안전망 500에 위임한다 — DB unique 위반은 안전망에 맡기고 정상 흐름은 사전 find로 거른다는 기존 find-first 정책([[find-first-write-not-check-db-unique-멱등]])을 그대로 유지한 것이다. UNIQUE 외 다른 무결성 위반(향후 제약 추가 시)이 retry로 silent하게 묻히는 것도 막는다. cart insert race는 같은 사용자가 같은 productId를 처음 담는 순간 ms 단위로 두 번 요청해야 나는 극히 드문 경우이고, 한 번 row가 생기면 이후는 update race 경로(B로 흡수)로 분기하므로 안전망 위임이 실용적으로 충분하다.

## save 명시 호출 컨벤션

Processor 안에서는 dirty checking 묵시 의존을 끊고 `repository.save(entity)`를 명시 호출해 영속화 의도를 코드 표면에 남긴다. method-level 트랜잭션 컨벤션과 함께 이 세션에서 자리잡은 응용 계층 컨벤션이다.

## 트레이드오프·미해결 — insert race 멱등 흡수 재방문 여지

insert race를 안전망 500에 위임하는 것이 지금 결정이지만, cart UX상 "두 번째 클릭도 조용히 흡수"가 더 자연스럽다는 의견이 향후 강화되면 (C) atomic UPDATE 또는 `getConstraintName()` 기반 정확한 UNIQUE 식별 + retry로 전환할 여지가 있다. delete race도 같은 `@Version` 낙관락을 재사용하는데, cart delete는 retry loop 없이 409를 클라이언트에 그대로 노출한다 — 상세는 [[cart-delete-미존재-4xx-entity-경유-삭제]]. `IN` 절 조회 부담이 커지는 abuse 상한 논의는 [[cart-회원당-row-상한-미도입]]과 함께 열려 있다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
