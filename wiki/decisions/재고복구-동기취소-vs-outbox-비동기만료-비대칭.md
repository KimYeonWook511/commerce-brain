---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, stock, outbox, kafka, transaction-boundary, eventual-consistency, spring-batch]
created: 2026-05-29
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-stock-domain-overview]]"
---

# 재고 복구 비대칭 — 사용자취소는 동기, 만료는 Outbox+Kafka 비동기

## 컨텍스트·문제(복구 경로 두 종류)

주문이 취소되면 차감했던 재고를 다시 복구해야 한다. 그런데 취소는 두 경로로 온다: **사용자 주도 취소**(단건, 즉시)와 **배치 주도 만료**(chunk 단위, 고아 주문 정리, [[주문만료-spring-batch-chunk-retry-skip]]). 두 경로에 같은 복구 방식을 강제할지, 다르게 갈지가 문제다.

## 왜 비대칭인가(단건 vs chunk 락 보유시간)

- **사용자 취소 = 동기**: `OrderCancelService`가 같은 트랜잭션에서 `stockInventoryService.increase(...)`를 동기 호출한다. 단건 트랜잭션이라 동기 복구가 잡는 락 시간 = 단건 차감 시간. 운영 부담이 없다.
- **만료 = Outbox 비동기**: `OrderExpirationService`는 `outboxService.createStockRestoreOutboxEvent(...)`로 outbox 이벤트만 발행하고, Kafka consumer가 건당 별 트랜잭션으로 단건 복구한다. 이유는 락 보유 시간이다 — 만료는 chunk 트랜잭션(size 100)이라 chunk 안에서 100건 stock row를 직접 `SELECT FOR UPDATE`로 잡으면 chunk가 끝날 때까지 락이 유지되고, 그동안 정상 주문 흐름의 차감이 chunk 처리 시간 내내 대기한다. 명령만 발행하고 consumer가 단건으로 풀면 락 보유 시간이 다시 단건 처리 시간으로 줄어든다.

`expireOrder` 안에서 직접 stock 복구를 호출하지 않는 것은 order 도메인의 책임 경계(취소 사실만 알림)와 stock 복구의 실제 수행을 분리해 order 트랜잭션을 짧게 끝내려는 것이다.

## Outbox+Kafka 사슬(각 단계의 도입 이유)

이 구조는 한 번에 깐 over-engineering이 아니라 각 단계가 앞 단계가 만든 새 문제를 받아 푼 누적 결과다([[stock-도메인-구조-개요]]에서 상세). 사슬:

1. **주문↔결제 트랜잭션 분리** — 결제는 외부 PG API 호출이라 같은 트랜잭션에 묶으면 트랜잭션이 비현실적으로 길어진다 → 분리하면 "주문은 생겼는데 결제 미완료"인 **고아 주문** 발생.
2. **고아 주문 정리는 만료 배치** — polling 대신 batch(시점 일관성·운영 가시성·peak 부하 회피).
3. **배치 chunk = 한 트랜잭션 → 락 확대** — chunk가 stock 락을 잡으면 chunk 처리 내내 정상 차감이 대기.
4. **그래서 비동기 분리** — 배치는 복구 명령만 발행, consumer가 단건 복구 → 락 보유 시간이 단건으로 축소.
5. **비동기 = 유실 가능 → 메시지 큐(Kafka)** — 발행한 복구 명령이 확실히 소비되도록.
6. **Kafka 발행 = 외부 시스템 호출 → 원자성 필요 → Outbox** — "주문 취소 DB commit + Kafka publish"가 따로 놀면 "DB는 commit됐는데 Kafka 발행 실패"로 복구 메시지가 영구 유실될 수 있다. Outbox row를 같은 DB 트랜잭션에 박고 relay(스케줄러)가 안정적으로 publish + 재시도한다.

핵심 사슬: 외부 PG API → 고아 주문 → 만료 배치 → chunk 락 확대 → 비동기 분리 → 유실 방지 → 메시지 큐 → 발행 보장 → Outbox. HTTP 요청의 traceId가 Kafka(producer/consumer 인터셉터)와 Outbox(traceId 컬럼 저장 → relay 시 MDC 복원) 경계를 지나 주문→재고 복구 전체를 단일 traceId로 잇는다 — 이벤트 경계의 traceId 명시 동봉은 [[주문-멱등성-캐싱-after-commit-이벤트-분리]]와 같은 결이다.

## 사용자취소를 outbox로 통일하지 않은 이유(단순성 우선)

대칭성(모든 취소를 outbox로 통일)보다 단순성을 우선했다. 사용자 취소는 단건이라 락 부담이 없으므로 직관적인 동기 호출이 낫다. 만료는 chunk 락 부담 때문에 outbox가 필수지만, 부담 없는 흐름까지 outbox로 감싸면 불필요한 비동기 복잡도가 늘 뿐이다.

## 관리자 increase vs 주문취소 increase 메서드 분리

재고 증가는 호출 경로별로 애초에 별도 메서드다([[관리자-재고조작-별도api-이력-감사-분리]]의 보강):

- `AdminStockService.increaseByAdmin(...)` — 관리자 경로(변경 이력 기록 책임).
- `StockInventoryService.increase(...)` — 주문 취소 경로(이력 미기록).

이름부터 다른 메서드라 호출 시점에 책임이 자연스럽게 강제된다. 같은 이름의 분기 메서드가 아니라 별도 메서드다.

## 트레이드오프(eventual vs 즉시 일관성)

- 만료 경로는 eventual consistency — 만료 → 복구 사이 짧은 gap이 있다.
- 사용자 취소는 즉시 일관성. 두 흐름의 의미가 달라 사용자 경험도 다르다.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
- [[raw/sessions/backend/2026-05-29-stock-domain-overview]]
