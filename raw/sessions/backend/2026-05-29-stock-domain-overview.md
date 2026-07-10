---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

## 도메인 개요

- `Product : Stock = 1:1` 관계. stock은 별도 엔티티/테이블로 분리.
- 책임: 상품별 현재 재고 / 주문 경로 재고 차감·복구 / 관리자 초기 재고 생성·수동 조정 / 재고 변경 이력.
- application 서비스 3개:
  - `StockInventoryService` — 주문 흐름 차감 (단건/배치)
  - `AdminStockService` — 관리자 초기 생성·증가·감소·이력 조회
  - `StockConcurrencyService` — 동시성 전략 격리
- 재고 복구는 `outbox/stock/` 서브 모듈에서 Outbox 패턴 + Kafka로 비동기 처리.
  - 주문 취소/만료 → `StockRestoreOutboxCreateService` → Scheduler → Kafka → Consumer → `StockRestoreOutboxConsumeService`

## 핵심 결정

### 1. 비관적 락 기본 전략 (ADR-003)

주문 경로 재고 차감은 비관적 락(`SELECT ... FOR UPDATE`)을 기본으로 둔다.

**선택 근거** — 후보 비교:

- **낙관적 락 ✗**: 재고는 동시성 충돌이 잦은 특성이라 잦은 retry로 서버 부담만 커진다.
- **Redis 분산 락 ✗**: 단일 DB 환경이라 외부 동기화 지점이 불필요. DB 부하가 심해지는 시점에 도입 검토.
- **atomic UPDATE (`WHERE quantity >= N`) ✗**: affected rows = 0 일 때 "상품 없음"과 "재고 부족"을 구분할 수 없어 도메인 예외를 분리해 던질 수 없다. 사전 SELECT를 추가하면 결국 같은 row lock이라 atomic 이점이 사라진다.
- **비관적 락 ✓**: `SELECT FOR UPDATE`로 잡은 상태에서 "상품 존재 여부 → 재고 충분 여부"를 도메인 객체 안에서 명시적으로 검증하고 각각 다른 예외로 던질 수 있다. 같은 트랜잭션이라 자동 롤백.

**트레이드오프**: 경쟁이 심해지면 락 대기·DB 부담이 커진다. 임계에 닿으면 Redis 락으로 전환 예정.

**사실관계 정정 메모**: 처음에는 "atomic UPDATE는 보상 트랜잭션이 필요하다"라고 생각했지만, RDB 트랜잭션 안의 UPDATE는 다른 SQL과 똑같이 자동 롤백된다. 보상 트랜잭션이 진짜 필요한 경우는 "트랜잭션 경계 밖 외부 시스템 호출" 또는 "트랜잭션 없이 즉시 반영되는 작업(Redis INCR 등)". 비관적 락이 atomic UPDATE보다 우위인 진짜 이유는 *원자성/롤백*이 아니라 *실패 원인을 도메인 예외로 정밀하게 구분할 수 있다*는 점.

### 2. 관리자 재고 관리 분리 + 변경 이력 (ADR-004)

관리자 초기 재고 생성·수동 증가/감소는 상품 API와 별도의 재고 API로 분리하고, 변경은 `tbl_stock_history`에 감사 데이터로 남긴다.

**왜 주문 차감/복구는 history에 안 남기는가** — "테이블 폭증" 우려보다 *테이블 목적 자체* 가 이유:

- **`tbl_stock_history`는 관리자 감사용으로 설계됨**. 핵심 컬럼이 `관리자 member_id + 변경 사유`. 주문 차감은 관리자가 없고 사유도 단조로워서 같은 테이블에 끼우면 NULL/sentinel이 늘어나고 "감사" 의미가 흐려진다.
- **주문 흐름은 다른 도메인이 이미 기록**. 주문 생성/취소/만료는 order 도메인이, 재고 복구는 `tbl_outbox_event` (Kafka payload 포함)가 기록. traceId(ADR-017, 019)로 주문→재고 흐름이 단일 traceId로 연결됨. 같은 사실을 history에도 적으면 SoR이 둘이 되어 일관성 부담만 늘어남.
- **변경의 성격이 다름**. 관리자 변경은 *수동·의도적·외부 입력*, 주문 차감은 *자동·시스템 결과*. 둘을 섞으면 history 조회 시 매번 case-split 필요.

→ history 테이블의 *목적*과 *조회 동기*가 "관리자 책임 추적" 한 가지로 좁아서, 다른 변경원은 의도적으로 배제.

### 3. 재고 복구는 Outbox + Kafka 비동기 흐름 (`outbox/stock/`)

이 구조는 한 번에 깐 over-engineering이 아니라 *각 단계가 그 단계의 문제를 풀려고* 누적된 결과:

1. **주문↔결제 트랜잭션 분리** — 결제는 외부 PG API라 같은 트랜잭션에 묶으면 트랜잭션 길이가 비현실적. 분리하면 "주문은 생겼는데 결제 미완료" *고아 주문*이 발생 가능.

2. **고아 주문 정리는 Spring Batch (만료 처리)** — polling 대신 batch를 택한 이유:
   - 시점 일관성 (cut-off 기준 명확)
   - 운영 가시성 (Spring Batch metadata로 진행/실패/재시작 추적)
   - peak 부하 회피 (오프피크에 몰아서)
   - 장애 복구 spike가 정상 트래픽에 섞이지 않음

3. **그런데 batch chunk = 하나의 트랜잭션** — chunk size만큼 read-process-write가 한 트랜잭션 안에서 commit. 재고 row X lock을 chunk가 잡으면 chunk 완료까지 유지 → 정상 주문 흐름의 `SELECT FOR UPDATE`가 chunk 처리 시간 내내 대기. **락 보유 시간이 단건이 아니라 chunk 단위로 확대됨.**

4. **그래서 비동기 분리** — batch는 *복구 명령만 발행*하고, 실제 재고 차감 복구는 consumer가 *단건 처리*. 락 보유 시간 = 단건 처리 시간으로 축소.

5. **비동기 = 유실 가능 → 메시지 큐 (Kafka)** — 발행이 확실히 소비되도록.

6. **Kafka 발행 = 외부 시스템 호출 → "주문 취소 commit + Kafka 발행"의 원자성 필요 → Outbox** — 주문 취소 DB commit과 Kafka publish가 분리되면 "DB는 commit인데 Kafka 발행은 실패" → 복구 메시지 영구 유실. Outbox row를 같은 DB 트랜잭션에 박아놓고 scheduler가 안정적으로 publish + 재시도.

**핵심 사슬**: PG(외부 API) → 고아 주문 → batch 정리 → chunk 락 → 비동기 분리 → 유실 방지 → 메시지 큐 → 발행 보장 → Outbox. 단계마다 도입된 *그 단계 특유의 문제*가 있다.

## 도메인 경계에서 배운 것

### DDD 이관 회고에서 (`docs/ddd/stock-ddd-migration-retrospective.md`)

- DDD 이관 커밋과 legacy 삭제 커밋을 섞지 않는다 — 리뷰 부담·변경량 분리.
- 책임 중심 네이밍: `StockInventoryService` ≪ `OrderStockService` — "주문에서 호출되는" 보다 "재고 책임" 우선.
- 구현 전략(비관적 락)은 public method 이름에 노출하지 않는다 (`decreaseWithPessimisticLock` ✗ → `decrease` ○). 호출부가 구현 세부에 묶이는 걸 막음.
- 테스트는 application service 책임 단위로 분리 (Admin / Inventory / Concurrency).

### 초기 재고 0 vs 이력 quantityChange 0 충돌 (`docs/tasks/stock-management/retrospective.md`)

- 요구사항 충돌: 초기 재고 0 허용 / `StockHistory.quantityChange` 0 불허.
- 해결: "초기 재고 0이면 stock만 생성하고 history는 저장 안 함" 예외 정책.
- 교훈: 요구사항끼리 충돌 가능한 정책은 *구현 전 계획 단계*에 잠가야 한다.

## 다시 본다면

- (사용자 작성 영역 — 현재 시점에서 stock 도메인을 처음부터 짠다면 바꿀 것 / 그대로 갈 것)

## 다음 단계 / 미해결

- 재고 history pagination 부재 — ADR-004 트레이드오프로 명시. 데이터 늘어나면 후속 작업.
- 락 경쟁 모니터링 — ADR-003의 "수용 가능 판단"이 실제 부하에서 깨지는 시점 측정 필요. 임계에 닿으면 Redis 락 전환.

## 인용

- `[[commerce-backend/docs/ADR.md#ADR-003]]` — 비관적 락
- `[[commerce-backend/docs/ADR.md#ADR-004]]` — 관리자 재고 분리·이력
- `[[commerce-backend/docs/ADR.md#ADR-019]]` — Outbox traceId 전파 (재고 복구 흐름 포함)
- `[[commerce-backend/docs/ddd/stock-ddd-migration-retrospective.md]]` — DDD 이관 회고
- `[[commerce-backend/docs/tasks/stock-management/retrospective.md]]` — 관리자 재고 API 회고
- `[[commerce-backend/docs/architecture.md]]` — outbox/stock 흐름
