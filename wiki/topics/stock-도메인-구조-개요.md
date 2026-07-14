---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [stock, domain-model, inventory, pessimistic-lock, outbox, stock-history]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-stock-domain-overview]]"
---

# 재고(stock) 도메인 구조 개요 — 차감·복구·이력·동시성 격리의 세 축

## 한 줄 정의

상품별 현재 재고를 `Product`와 1:1로 대응시키되 상품 테이블에 얹지 않고 별도 엔티티/테이블(`tbl_stock`)로 분리한 도메인. 재고를 관통하는 세 결정(주문 차감의 비관적 락, 관리자 조작·이력의 주문 흐름 분리, 재고 복구의 Outbox 비동기화)의 진입점이다.

## Product와의 1:1·의존 방향

재고는 `Product`와 논리적으로 1:1이지만 상품 엔티티에 재고 컬럼을 얹지 않고 별도 엔티티/테이블(`tbl_stock`)로 뗐다. 참조 방향은 **재고 → 상품**으로, 재고 쪽이 `product_id`로 상품을 가리킨다. 상품 조회 흐름에서 재고 row가 아직 없을 수 있는 비대칭은 [[product-상세조회-stock-의존-재고누락-0-정규화]]에서 상품 상세 응답이 재고 누락을 0으로 정규화하는 방식으로 흡수한다. 상품 도메인 전체 구조는 [[product-도메인-구조-개요]] 참고.

## 네 책임과 세 application 서비스

도메인이 지는 책임은 넷이다 — (1) 상품별 현재 재고 보관, (2) 주문 경로의 재고 차감·복구, (3) 관리자의 초기 재고 생성·수동 조정, (4) 재고 변경 이력. 이 책임이 application 계층에서 세 서비스로 갈린다.

| 서비스 | 책임 |
|---|---|
| `StockInventoryService` | 주문 흐름의 차감(`decrease`)·복구(`increase`), 단건/배치 |
| `AdminStockService` | 관리자 초기 생성·수동 증감(`increaseByAdmin`)·이력 조회 |
| `StockConcurrencyService` | 동시성 전략을 격리하는 계층 |

서비스 분리는 DDD 이관 회고에서 굳힌 원칙을 따른다. 이름은 "주문에서 호출된다"(`OrderStockService`)가 아니라 실제 책임(재고)을 앞세운 `StockInventoryService`로 짓고, 비관적 락 같은 **구현 전략은 public 메서드 이름에 노출하지 않는다**(`decreaseWithPessimisticLock` ✗ → `decrease` ○). 관리자 경로(`increaseByAdmin`, 이력 기록)와 주문 취소 경로(`increase`, 이력 미기록)를 **애초에 다른 이름의 별도 메서드**로 둔 것도, 호출 시점에 책임이 자연스럽게 강제되게 하기 위한 설계다 — 자세한 근거는 [[관리자-재고조작-별도api-이력-감사-분리]]와 [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]].

## 재고 복구 outbox 서브모듈

재고 복구만 `outbox/stock/` 서브모듈에서 Outbox 패턴 + Kafka로 비동기 처리한다(주문 취소/만료 → 복구 Outbox 발행 → 스케줄러 relay → Kafka → consumer → 단건 재고 증가). 이 구조는 한 번에 깐 over-engineering이 아니라 각 단계가 앞 단계가 만든 새 문제를 받아 푼 누적의 결과다: 외부 PG API 호출로 주문↔결제 트랜잭션을 분리 → 고아 주문 발생 → 만료 배치 정리 → 배치 chunk가 재고 락을 chunk 단위로 확대 → 비동기 분리로 락 보유 시간을 단건으로 축소 → 비동기 유실 방지 위해 Kafka → "취소 commit + Kafka 발행"의 원자성 위해 Outbox.

이 복구 경로가 **사용자 취소(동기)와 배치 만료(Outbox 비동기)로 의도적 비대칭**을 이루는 판단은 order 측 결정 [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]에 정본이 있다. 배치 자체의 chunk/retry/skip 구성은 [[주문만료-spring-batch-chunk-retry-skip]] 참고. HTTP 요청에서 발급한 traceId가 Kafka 인터셉터와 Outbox 컬럼(relay 시 MDC 복원)을 지나 주문→재고 복구 전체를 단일 traceId로 잇는데, 이 traceId 전파 결의 뿌리는 [[주문-멱등성-캐싱-after-commit-이벤트-분리]]의 이벤트 동봉과 [[mdc-정리-스코프-오너십-2규칙]]의 MDC 오너십 규칙과 같은 결이다.

## 관련 결정 링크

- [[재고차감-동시성-비관락과-productid-정렬]] — 주문 경로 차감의 기본 전략을 비관적 락으로 두고(낙관락·Redis락·atomic UPDATE 기각), 여러 상품 동시 주문 시 productId 정렬로 데드락을 회피. order 측이 이 전략에 의존만 한다.
- [[관리자-재고조작-별도api-이력-감사-분리]] — 관리자 조작을 별도 API로 빼고 그 변경만 `tbl_stock_history`에 감사 데이터로 남긴다.
- [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]] — 재고 복구 Outbox 서브모듈의 존재 이유(order 측 정본).
- [[order-도메인-구조-개요]] — 재고를 차감·복구하는 주문 흐름 전체 지도.
- [[product-도메인-구조-개요]] · [[product-상세조회-stock-의존-재고누락-0-정규화]] — 재고가 가리키는 상품 도메인과 재고 누락 정규화.

## 열린 질문

- **재고 이력 pagination 부재** — 이력 조회 첫 버전은 pagination 없이 상품별 전체 목록을 반환한다. 데이터가 늘면 후속으로 붙여야 한다([[관리자-재고조작-별도api-이력-감사-분리]]의 트레이드오프로 명시).
- **락 경쟁 모니터링** — 비관적 락을 "수용 가능"으로 판단한 근거(단일 DB·현재 경쟁 수준)가 실제 부하에서 언제 깨지는지 측정이 필요하다. 임계에 닿으면 Redis 분산 락으로 전환.

## 근거

- [[raw/sessions/backend/2026-05-29-stock-domain-overview]]
