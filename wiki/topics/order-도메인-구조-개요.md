---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, order-item, aggregate, domain-model, idempotency, concurrency, expiration]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# 주문(order) 도메인 구조 개요 — 엔티티·상태·서비스 경계 지도

## 한 줄 정의

주문 도메인은 `Order` aggregate를 중심으로 멱등성(생성)·동시성(재고 차감)·비대칭 복구(취소/만료)·낙관락/비관락 공존이라는 네 개의 설계 축을 품는다. 이 노트는 그 구조와 각 결정으로 들어가는 진입점 지도다.

## 엔티티·테이블·unique 제약

- `Order : OrderItem = 1:N` (Aggregate). `OrderItem`은 독립 유스케이스가 없어 별도 최상위 도메인 패키지를 만들지 않고 `order.domain.OrderItem`으로 aggregate 내부에 묶었다.
- `Order : Payment = 1:1` — 결제가 주문을 참조하는 방향. 주문은 결제 흐름에서 채워지는 `merchantPayKey`를 unique로 보유한다(결제 식별자가 주문에 박히는 이 구조의 도메인 책임 누수는 [[payment-도메인-구조-개요]]에서 재설계 후보로 다뤄진다).
- `Order : Member = N:1`.
- `tbl_order` 주요 컬럼: `id`, `version`(낙관락 `@Version`), `member_id`, `total_price`, `status`, `merchant_pay_key`(unique), `idempotency_key`(nullable).
- unique 제약 2개: `merchant_pay_key` 단일 unique, `uk_order_member_idempotency (member_id, idempotency_key)` 복합 unique. `idempotency_key`는 NULL 허용 — 멱등성 없는 학습 경로(`OrderConcurrencyService`)와 호환 + MySQL이 NULL을 unique 비교에서 제외하는 동작을 활용한 것. 상세는 [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]].

## OrderStatus 전이와 도메인 캡슐화

- enum 값은 `INIT`, `CANCELED`, `RECEIVED`, `PAID`, `COMPLETED` 5개지만, 실제 전이는 `INIT → CANCELED`(`order.cancel()`)와 `INIT → PAID`(`order.completePayment()`)만 코드에 존재한다. `RECEIVED`, `COMPLETED`는 enum 정의만 있고 전이 코드가 없는 향후 단계용 placeholder다.
- 상태 전이·검증은 도메인 메서드에 캡슐화했다: `order.cancel()`, `order.completePayment()`, `order.checkPayable()`. 모두 `status != INIT`이면 도메인 예외를 던진다(`checkPayable()`은 전이 없이 선조건만 검증). 결제 도메인이 `succeed`/`fail` 전이를 도메인 메서드에서 명시 검증하는 것과 같은 결의 명시적 선조건 검증 패턴이다.
- `Order.assignMerchantPayKey(...)`는 `null`일 때만 set하는 멱등 setter — 한 Order = 한 merchantPayKey 영구.

## application 서비스 유스케이스 분리

한 덩어리 `OrderCommandService`를 두지 않고 유스케이스 책임 단위로 쪼갰다(공통 컨벤션은 [[ddd-이관-컨벤션-adapter-command-query-네이밍]]).

| Service | 책임 | 트랜잭션 |
|---|---|---|
| `OrderCreateService` | 멱등성 분기 only | `NOT_SUPPORTED`(트랜잭션 없음) |
| `OrderCreateProcessor` | 재고 차감 + 주문 저장 + 이벤트 발행 | `@Transactional` |
| `OrderCancelService` | 사용자 주도 취소(동기 재고 복구) | `@Transactional` |
| `OrderExpirationService` | 만료 처리(취소 + outbox 이벤트) | `@Transactional` |
| `OrderQueryService` | `merchantPayKey` 단건 조회(결제 흐름용) | `readOnly` |
| `OrderConcurrencyService` | 동시성 전략 비교 학습용, production 아님 | 전략별 |

한 서비스에 생성·취소·만료·조회·동시성 실험을 다 두면 public API가 과하게 넓어진다. `OrderConcurrencyService`의 production 격리는 [[order-concurrency-service-학습코드-격리]] 참조.

## 외부 경계(stock/redis/payment/outbox)

- Stock: `StockInventoryService.decrease`/`increase` 동기 호출(생성/사용자 취소 경로). 락 전략은 [[재고차감-동시성-비관락과-productid-정렬]] · [[stock-도메인-구조-개요]]에 의존.
- Outbox → Stock: `OutboxService.createStockRestoreOutboxEvent(...)` 비동기 발행(만료 경로만). 동기/비동기 비대칭은 [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]].
- Redis: `OrderIdempotencyStore` port → `RedisOrderIdempotencyStore` adapter(멱등 캐싱은 [[주문-멱등성-캐싱-after-commit-이벤트-분리]]).
- Payment: `OrderQueryService`를 통해 결제가 `merchantPayKey`로 Order를 가져간다.

## 코드의 작은 신호(가격 스냅샷 부재 등)

- `OrderItem`에 가격 컬럼이 없다 — 주문 시점 가격 스냅샷을 남기지 않고 `product.getPrice() * quantity`를 `totalPrice`에 누적한다. 주문 후 상품 가격이 바뀌면 `totalPrice`와 현재 product price가 불일치한다. 코드 주석에 "가격도 넣어야 하나… 추후 고려하기" TODO가 남아 있다. 쿠폰/정산/리뷰 도입 전 반드시 정리해야 하는 부채로, [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]에서 다룬다.
- `OrderStatus.RECEIVED`, `COMPLETED`는 enum에만 정의되고 전이 코드가 없는 placeholder.

## 관련 결정 링크

- 생성 멱등성: [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]] / [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- 재고 차감 동시성: [[재고차감-동시성-비관락과-productid-정렬]]
- 취소/만료 비대칭·배치: [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]] / [[주문만료-spring-batch-chunk-retry-skip]]
- 락 공존·학습 코드: [[order-version-낙관락-비관락-공존]] / [[order-concurrency-service-학습코드-격리]]
- 도메인 공통 컨벤션: [[ddd-이관-컨벤션-adapter-command-query-네이밍]]

## 열린 질문

- `OrderItem` 가격 스냅샷 부채 정리(쿠폰/정산 도입 전).
- 만료 batch 숫자 튜닝(chunk 100 / retry 3 / skip 10 / cron 5분 / TTL 10분 / 만료 cutoff 60분 — 모두 기본값).
- `Order.@Version`의 실제 OLE 발생 데이터 수집.
- `OrderConcurrencyService` 운영 진입 시 제거 처리.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
