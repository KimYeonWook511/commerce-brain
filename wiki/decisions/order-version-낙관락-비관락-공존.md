---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, optimistic-lock, pessimistic-lock, version, race-condition, concurrency]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# Order @Version 낙관락 + 비관락 공존 — 만료/결제 race를 @Version이 잡는다

## 컨텍스트·문제(만료와 결제의 동시 접근)

같은 Order를 두 흐름이 동시에 건드릴 수 있다: 만료 batch가 고아 주문을 취소하려는 순간, 결제 콜백이 그 주문을 완료(PAID)로 전이시킬 수 있다. 어느 한쪽만의 락 전략으로는 이 race를 안전하게 다루기 어렵다.

## 현재 코드 동작(만료 낙관락 / 결제 비관락)

- `Order`에 `@Version private Long version`.
- **만료 흐름**(`OrderExpirationService.expireOrder`): `findByIdWithItems()` **일반 SELECT**(FOR UPDATE 아님) → `order.cancel()` → outbox 이벤트 발행 → commit 시 Order update + `@Version` 검사.
- **결제 흐름**(결제 승인 서비스): `findByMerchantPayKeyForUpdate` **비관락** → `order.completePayment()` → save.
- **batch 정책**(`OrderExpirationBatchConfig`): `retry(OLE, 3)` + `skip(OLE, 10)` — [[주문만료-spring-batch-chunk-retry-skip]].

## @Version이 빛나는 race window(실제 시나리오)

실제 코드상 성립하는 race:

- t1: 만료 batch가 Order load(version=1, status=INIT).
- t2: 결제 콜백이 같은 Order에 FOR UPDATE 획득 → `completePayment()` → save(version=2) → commit.
- t3: 만료 batch가 `cancel()` → save 시도 → `WHERE version=1` update 0 rows → **OLE**.
- batch retry → load(version=2, status=PAID) → `cancel()` 호출 → 도메인 예외(`status != INIT`) → skip 정책으로 단건 보류.

즉 `@Version`이 만료/결제 race를 안전하게 catch하는 핵심 메커니즘이고, 그 뒤처리는 batch의 retry(OLE)/skip 정책이 맡는다.

## 비관락만으로는 안 되는 이유(chunk 락 보유)

만료 batch도 FOR UPDATE로 잡으면 결제 흐름과 직렬화되어 race가 사라지긴 한다. 하지만 chunk 단위 FOR UPDATE는 chunk 처리 시간 내내 락을 보유해 정상 주문/결제 흐름을 차단한다(재고 쪽에서 같은 chunk 락 확대 문제를 outbox로 푼 [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]와 같은 압력). 그래서 만료는 FOR UPDATE를 안 잡고 낙관락에 의존, 결제는 비관락으로 두고, 둘이 만나는 race window를 `@Version`이 잡는다.

재고 차감 경로가 비관락 + productId 정렬을 쓰는 것([[재고차감-동시성-비관락과-productid-정렬]])과 합쳐, 주문 도메인은 흐름별로 락 전략을 나눠 쓴다. 각 락 전략의 동작·비교는 [[order-concurrency-service-학습코드-격리]]에 실험용으로 보존돼 있다(`createOrderWithOptimisticLock` 등).

## 미해결(실제 OLE 발생 데이터 수집)

이 race가 실제 어떤 흐름에서 얼마나 자주 OLE로 터지는지 데이터가 없다. 운영에서 OLE 발생 사례를 수집해 retry(3)/skip(10) 숫자가 적절한지 검증할 예정.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
