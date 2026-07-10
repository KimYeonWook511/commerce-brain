---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

## 도메인 개요

### 엔티티 / 테이블

- `Order : OrderItem = 1:N` (Aggregate). `OrderItem`은 독립 유스케이스가 없어 **별도 최상위 패키지를 만들지 않고** `order.domain.OrderItem`으로 묶음 (DDD 회고에서 확정).
- `Order : Payment = 1:1` — Payment 가 `order_id` unique 로 보유.
- `Order : Member = N:1`.
- `tbl_order` 주요 컬럼: `id`, `version`(낙관락), `member_id`, `total_price`, `status`, `merchant_pay_key`(unique), `idempotency_key`(nullable).
- 유일 제약 2개: `merchant_pay_key UNIQUE`, `uk_order_member_idempotency (member_id, idempotency_key) UNIQUE`. `idempotency_key` 는 NULL 허용 — `OrderConcurrencyService` 같은 멱등성 없는 경로 호환 + MySQL NULL 은 unique 적용 제외라는 동작 활용.

### 상태 (`OrderStatus`)

- `INIT` → `CANCELED` / `PAID` / (`RECEIVED`, `COMPLETED` 는 enum 만 존재, 실제 전이 코드 없음).
- 상태 전이는 도메인 메서드에 캡슐화 (`order.cancel()`, `order.completePayment()`, `order.checkPayable()`) — `status != INIT` 이면 도메인 예외. payment 의 `succeed`/`fail` 처럼 *명시적 선조건 검증* 패턴 (ADR-012 와 같은 결).

### Application 서비스 (유스케이스 단위 분리)

| Service | 책임 | 트랜잭션 |
|---|---|---|
| `OrderCreateService` | 멱등성 분기 only. `@Transactional(NOT_SUPPORTED)` | 트랜잭션 없음 |
| `OrderCreateProcessor` | 실제 재고 차감 + 주문 저장 + 이벤트 발행 | `@Transactional` |
| `OrderCancelService` | 사용자 주도 취소 (동기 재고 복구) | `@Transactional` |
| `OrderExpirationService` | 만료 처리 (취소 + outbox 이벤트) | `@Transactional` |
| `OrderQueryService` | `merchantPayKey` 단건 조회 (결제 흐름용) | `readOnly` |
| `OrderConcurrencyService` | 동시성 전략 비교 *학습용*, production 흐름 아님 | `@Transactional` |

- `OrderCommandService` 한 덩어리로 두지 않고 책임 단위로 쪼갠 결정은 DDD 이관 회고에 명시.
- `OrderConcurrencyService` 는 production 의 `createOrder` 와 *섞지 않고* 분리 — 학습용 strategy 노출이 *목적*이라 strategy 네이밍이 의도적으로 유지됨.

### Batch

- 만료 처리는 `OrderExpirationBatchConfig` (Spring Batch, chunk = 100, default cron `0 */5 * * * *`).
- Reader: `id > lastId` 페이징 + `status = INIT and createdAt < cutoff` (no offset, no count — keyset).
- Writer: chunk 안에서 `OrderExpirationService.expireOrder(orderId, requestedAt)` 단건 호출 = chunk 트랜잭션 안에서 N건 처리.
- `faultTolerant + retry(OptimisticLockingFailureException, 3) + skip(OptimisticLockingFailureException | CustomException, 10)`.
- 비즈니스 유스케이스(`expireOrder`)는 batch 패키지에 두지 않고 application service 로 두어 *scheduler/admin/manual 흐름에서도 재사용 가능* 하게 설계 (DDD 회고 명시).

### 외부 경계

- Stock: `StockInventoryService.decrease/increase` 동기 호출 (create / 사용자 cancel).
- Outbox/Stock: `OutboxService.createStockRestoreOutboxEvent(...)` 비동기 발행 (만료 경로만).
- Redis: `OrderIdempotencyStore` port → `RedisOrderIdempotencyStore` adapter.
- Payment: `OrderQueryService` 를 통해 payment 가 `merchantPayKey` 로 Order 를 가져감.

---

## 핵심 결정

### 1. 멱등성: Redis 1차 + RDB unique 2차 (ADR-002 / order-idempotency task)

**결정 요약**: 주문 생성은 `Idempotency-Key` 헤더 필수. Redis `setIfAbsent` 로 1차 reserve, `(member_id, idempotency_key)` RDB unique 로 최종 보장.

**왜 이중 방어인가 — 결정적 동기**:
- **Redis 죽어도 동작 가능해야 함** — 주문은 핵심 기능. Redis 장애 시에도 주문 자체는 동작해야 함. 그러기 위해서 DB에서도 멱등성을 보장할 수단이 필요했고, RDB unique 제약이 그 역할.
- Redis 1차의 가치: 정상 흐름 99% 흡수 → DB 부하 절감, 빠른 hit 응답.
- RDB 2차의 가치: TTL 만료 / Redis 장애의 *구조적* 빈틈 + Redis 죽었을 때도 멱등성 자체는 깨지지 않음.

**왜 `(member_id, idempotency_key)` 복합 키인가**:
- 클라이언트 UUID 충돌 가능성을 조금이라도 낮추기 위함. UUID 자체로도 충돌 확률은 극히 낮지만 *멤버 격리로 한 겹 더*.

**흐름 (현재 코드 기준)**:

```
createOrder(command)
 ├─ idempotencyKey 비어있음 → INVALID_REQUEST
 ├─ reserve(memberId, key, ttl=600초)
 │   ├─ true  → attemptCreateOrder
 │   └─ false (이미 PROCESSING / COMPLETED / Redis 장애 fallback)
 │       ├─ getCompletedOrderId() hit → DB findById → 반환 (source=redis)
 │       └─ miss → attemptCreateOrder
attemptCreateOrder
 ├─ findByMemberIdAndIdempotencyKey 사전 체크 → 있으면 Redis complete 갱신 후 반환 (source=db)
 ├─ 없으면 OrderCreateProcessor.execute() (별도 @Transactional)
 │   ├─ 성공 → publishEvent(OrderIdempotencyCacheEvent) → AFTER_COMMIT 에서 Redis complete
 │   └─ RuntimeException (race 충돌 시 DataIntegrityViolationException 포함)
 │       → orderIdempotencyStore.clear() 후 rethrow (안전망 500, ADR-011)
```

**숨은 디테일**:
- `OrderCreateService`가 `@Transactional(NOT_SUPPORTED)`인 이유: `DataIntegrityViolationException` catch 후 *같은 트랜잭션에서 DB 재조회 불가* (rollback-only). Processor를 별도 Bean으로 떼고, Service는 트랜잭션 없이 분기만 → Processor 실패 시 새 트랜잭션으로 재조회 가능. ADR-008(회원가입 트랜잭션 분리)과 동일한 패턴.
- ADR-002 본문의 catch 분기는 **ADR-011 (find-first 정책)** 으로 전환됨. 사전 `find`가 정상 멱등 재요청을 흡수하고 race window 충돌만 안전망 500으로 보냄.
- `clear()` on rollback: race 충돌로 인한 RuntimeException 시 Redis의 `PROCESSING` 마커를 지워야 다음 정당한 재시도가 막히지 않음.

**현재 코드의 미해결/이상함 — Issue 등록됨, 후속 개선 예정**:

코드 직접 확인 결과 (`OrderCreateService.java`, `RedisOrderIdempotencyStore.java`, `OrderIdempotencyStatus.java`):

- **TTL = 600초 (10분)** — `@Value("${order.idempotency.ttl-seconds:600}")`.
  - *기억으로 재구성한 동기*: 네이버페이 결제 처리 시간(10분)과 맞춤. PG 측에서 결제가 10분 안에 완료 안 되면 그 주문은 PG 측 만료 대상이 되어 사용자도 새 시도로 넘어가게 되므로, 같은 멱등 키 재사용을 차단할 이유가 사라지는 시점이 10분.
  - **두 개의 다른 만료가 공존**:
    - TTL 10분 = PG 결제 처리 시간 기반 *멱등 키 활성 기간*
    - 만료 cutoff 60분 (`order.expiration.minutes:60`) = 별도 정책. 좀비 주문 cleanup 배치 주기 (PG 응답 없이 남은 주문 정리)
  - 두 숫자의 의미가 다름. 기억 기반 재구성이라 코드 개선 시점에 의도를 다시 정리하고 정확한 코멘트를 코드에 남길 예정.

- **`FAILED` enum 정의만 있고 어디서도 set 안 됨** — `OrderIdempotencyStatus.FAILED` 정의 + `parseCompletedOrderId()`도 COMPLETED 접두사만 인식. *실패 캐싱은 미구현*. **Issue #172** (feat: 주문 멱등성 실패 캐싱 정책 결정 및 구현) 로 등록됨.

- **PROCESSING 동시 요청 → 안전망 500**:
  - 시나리오: 같은 idempotencyKey로 Req A·B가 거의 동시 진입
  - Req A: reserve true → Processor.execute() 진행 중 (트랜잭션 commit 전)
  - Req B: reserve false (PROCESSING 마커) → `getCompletedOrderId()` miss (COMPLETED만 hit으로 봄) → attemptCreateOrder → 사전 find empty (Req A commit 전) → INSERT 시도 → unique 위반 → `clear()` 후 rethrow → **안전망 500**

- **추가 race window — Req B의 `clear()`가 Req A의 PROCESSING 마커도 지움**:
  - `clear()` = `redisTemplate.delete(key)` 로 키 통째 삭제
  - Req A의 AFTER_COMMIT 이벤트가 commit 후 `complete()`로 키를 다시 만들지만,
  - Req A commit 전에 Req C가 또 오면 → reserve true (PROCESSING 마커 없음) → 같은 패턴 반복 → 또 500

- **이상적 흐름과의 갭**:
  - PROCESSING 상태일 때 같은 키 요청은 *대기* 또는 *"현재 처리 중"* 응답이 자연스러움
  - 현재는 동시 요청이 모두 500을 받는 구조 — 클라이언트 입장에서 *"처리 중인지 실패인지" 구분 불가능*
  - 향후 개선 방향: `getCompletedOrderId`를 *PROCESSING도 인식*하도록 확장 + 적절한 응답 (예: 409 Conflict + "처리 중" 메시지) + retry 정책 명확화 + Req B catch의 `clear()` 동작도 재검토 (조건부 clear?)
  - **Issue #171** (fix: 주문 멱등성 PROCESSING 상태 동시 요청 안전망 500 응답 개선) 로 등록됨.

**트레이드오프**:
- TTL 만료 후 재요청 시 재고 차감 → unique 위반 → 롤백이 드물게 발생. 정확성에는 문제 없음.
- `idempotency_key` NULL 허용 → 멱등성 없는 경로(`OrderConcurrencyService` 학습용)와의 호환을 위함. MySQL NULL은 unique 비교 제외 동작에 의존.

### 2. AFTER_COMMIT 이벤트로 Redis 캐싱 분리 (ADR-005 구현)

**결정 요약**: Order 저장 트랜잭션 안에서 `applicationEventPublisher.publishEvent(OrderIdempotencyCacheEvent)` → `RedisOrderIdempotencyStore.handle()`가 `@TransactionalEventListener(AFTER_COMMIT)`으로 받아 Redis complete.

**왜 트랜잭션 안에서 Redis 호출 안 하나**:
- Redis 장애 시 RDB 롤백을 막아야 함. 멱등성 캐싱은 *정합성이 아니라 편의*. RDB가 최종 보장.
- Redis 호출이 동일 트랜잭션 안이면 Redis 장애 → 트랜잭션 전체 롤백 → 주문도 사라짐. ADR-005 기본 정책.
- `TransactionSynchronizationManager` 직접 사용보다 `@TransactionalEventListener`가 *DDD 레이어 경계 유지*에 자연스러움 — Application이 Infrastructure(Redis adapter)를 직접 알지 않아도 됨.

**traceId 전파 (ADR-019)**:
- 이벤트 객체가 publisher 시점의 `LogContext.getTraceId()`를 *동봉*. listener가 MDC에 push.
- AFTER_COMMIT은 기본 동기 실행 → 호출 스레드 MDC가 살아있으면 그대로 보존, 없을 때만 이벤트 traceId를 fallback으로 push.

**현재 상태의 미해결 — 동기/비동기 인식 실수 (솔직한 메모)**:
- `@TransactionalEventListener(AFTER_COMMIT)`가 *비동기로 동작한다고 잘못 인식*하고 도입했음. 실제는 **동기** — 호출 스레드에서 commit 후 그대로 실행.
- 즉 *클라이언트 응답 latency*에 Redis 호출 시간(`complete()` 1회)이 포함됨.
- 영향 자체는 작지만 (Redis 호출이 빠름), AFTER_COMMIT의 *원래 의도(트랜잭션과 분리한 별개 흐름)*가 *비동기까지여야 완성*된다는 점에서 미완.
- 향후 비동기로 분리 예정 (`@Async` 또는 `ApplicationEventMulticaster` TaskExecutor 주입).
- ADR-019의 *traceId 동봉 설계*가 사실 *이 비동기 전환을 대비한 것* — `pushTraceIdIfMissing` fallback 로직이 비동기 전환 후 진가를 발휘하게 됨.
- **Issue #173** (refactor: 주문 멱등성 캐싱 AFTER_COMMIT 리스너 비동기 전환) 로 등록됨.

**트레이드오프**:
- RDB 커밋 ~ Redis 캐싱 사이 짧은 gap에 동일 키 요청 오면 MISS → INSERT 시도 → unique 위반 → DB 재조회 (find-first로 흡수). 정확성은 OK.
- 이벤트 객체마다 `traceId` 필드 추가하는 반복 작업. ADR-019에서 이벤트가 5개 이상 되면 Multicaster wrapping 재검토. 현재 1개라 동봉으로 충분.

### 3. 비관적 락 + productId 정렬 (ADR-003, OrderCreateProcessor)

**결정 요약**: `OrderCreateProcessor.execute()`가 시작하자마자 items를 `productId` 오름차순으로 정렬한 뒤 stock 차감.

**왜 정렬인가**:
- 같은 두 상품을 서로 다른 순서로 차감하면 → 트랜잭션끼리 `SELECT FOR UPDATE`가 **상호 락 holding → 데드락**.
- 모든 트랜잭션이 동일 순서로 락을 잡으면 데드락 회피 (전형적인 lock ordering).

**왜 batch decrease가 아닌 단건 정렬인가** — 측정 기반 + 안정성 의사결정:

1. **InnoDB IN 절은 atomic lock acquisition이 아님**
   - 직관적으로 batch가 락 보유 시간이 더 짧을 거라고 추측했지만,
   - **테스트로 확인한 사실**: `SELECT ... FOR UPDATE` IN 절도 atomic하게 모든 락을 잡는 게 아니라, *옵티마이저가 인덱스 보고 결정한 순서로 순차적 row lock 획득*. 즉 batch도 단건 정렬과 사실상 같은 lock acquisition 패턴.
   - batch의 진짜 이점이 좁아짐 — 네트워크 round-trip 절감(N → 1) + SQL parsing 비용 절감뿐. 락 보유 시간 우위 없음.

2. **batch의 락 순서가 옵티마이저 / 인덱스 구조에 의존 → 잠재적 데드락 위험**
   - IN 절의 락 획득 순서는 *옵티마이저가 인덱스를 보고 결정*. 우리가 명시한 IN 적힌 순서 자체로 보장되는 게 아니라 *옵티마이저 결정*에 의존.
   - 인덱스 구조가 바뀌거나 옵티마이저 동작이 바뀌면 (MySQL 업그레이드 / 인덱스 추가·삭제 / 통계 변화) **의도한 락 순서가 깨져서 데드락 발생 가능**.
   - IN 절 정렬을 명시적으로 적을 수도 있지만 → *native query에 정렬 로직을 넣어야 함* → 나중에 수정 시 *문자열 텍스트 찾아서 수정*하는 불편 + 컴파일러·IDE refactoring으로 잡히지 않음.

3. **단건 정렬의 진짜 우위 — 추상화 깊이와 의존성 안정성**
   - 정렬 로직이 *Java 코드*(`List.sort()`) — 컴파일러 검증, IDE refactoring 지원.
   - DB 옵티마이저 동작과 무관 — 인덱스 변경·MySQL 업그레이드에도 영향 없음.
   - 각 차감 사이에 도메인 로직 끼울 수 있음 (`Stock.decrease(n)` 단위 검증·예외 분기).
   - 에러 메시지 정밀도 — 상품별 부족 사유 명시 가능.

→ batch는 *옵티마이저·인덱스 의존성*이라는 hidden coupling을 가지며, 그것이 *유지보수 시점에 깨질 수 있는 위험*을 안고 있음. 단건 정렬은 모든 정렬 책임이 *Java 코드 안에 명시*되어 있어 안정성 우위. `OrderConcurrencyService.createOrderWithPessimisticLockBatch`는 비교 실험으로 남겨두고 production은 단건 정렬 채택.

**stock 도메인 ADR-003(비관적 락 기본)을 order 측에서 활용** — order는 stock의 락 전략에 의존만 한다. order 자체는 락을 잡지 않음 (단, `findByMerchantPayKeyForUpdate`가 별도로 존재 — payment 흐름 사용).

### 4. 사용자 취소(동기 복구) vs 만료(Outbox 비동기 복구) — 의도적 비대칭

**결정 요약**:
- `OrderCancelService` (사용자 주도 취소): `stockInventoryService.increase(...)`를 **같은 트랜잭션에서 동기 호출**
- `OrderExpirationService` (배치 주도 만료): `outboxService.createStockRestoreOutboxEvent(...)`로 **outbox 이벤트만 발행** → Kafka → consumer가 단건 복구

**왜 비대칭인가** (stock 도메인 raw의 "락 보유 시간이 chunk 단위로 확대됨" 사슬과 정확히 동일):
- 사용자 취소는 *단건 트랜잭션*. 동기 복구가 lock을 잡는 시간 = 단건 차감 시간. 운영상 부담 없음.
- 만료는 *chunk 트랜잭션* (chunk size 100). chunk 안에서 100건의 stock row를 직접 `SELECT FOR UPDATE`로 잡으면 → chunk 완료까지 모든 stock 락이 유지됨 → 정상 주문 흐름의 차감이 chunk 처리 시간 내내 대기.
- 그래서 만료는 *명령만 발행*하고, consumer가 *건당 별 트랜잭션*으로 풀어줌. 락 보유 시간 = 단건 처리 시간.
- Outbox + Kafka인 이유: "주문 취소 commit + Kafka 발행"의 원자성 필요. 자세한 사슬은 stock 도메인 raw ADR-003 섹션 참조.

**`expireOrder` 안에서 직접 stock 복구를 호출하지 않는 이유**: `stockRestoreOutboxEvent`가 *order 도메인의 책임 경계*(취소 사실만 알림)와 *stock 복구의 실제 수행*을 분리해서, order 트랜잭션이 짧게 끝나도록.

**사용자 취소를 outbox로 통일하지 않은 이유 — 단순성 우선의 의도적 판단**:
- 사용자 취소는 *단건이라 락 부담이 없음* → 단순한 동기 호출이 더 낫다는 판단.
- *대칭성*(모든 취소를 outbox로 통일)보다 *단순성*(부담 없는 흐름은 직관적 동기 호출)을 우선.
- 만료 경로는 chunk 단위 락 부담 때문에 outbox가 필수, 사용자 취소는 그 부담이 없으므로 동기로 충분.

**관리자 증가 API와 메서드 시그니처 레벨 분리** (ADR-004 보강):
- `AdminStockService.increaseByAdmin(...)` — 관리자 경로 (history 기록 책임)
- `StockInventoryService.increase(...)` — 주문 취소 경로 (history 미기록)
- 이름부터 다른 메서드라서 *호출 시점에 책임이 자연 강제*됨. 같은 이름의 분기 메서드가 아니라 *애초에 별도 메서드*.

**트레이드오프**:
- 만료 경로는 eventual consistency. 만료 → 복구 사이 짧은 gap.
- 사용자 취소는 즉시 일관성. 두 흐름의 의미가 달라 사용자도 다른 경험을 받음.

### 5. Spring Batch + chunk = 100, retry/skip 차등

**결정 요약**: 만료는 5분 주기 cron, chunk 100, faultTolerant + retry(`OptimisticLockingFailureException`, 3회) + skip(`OptimisticLockingFailureException` | `CustomException`, 10건).

**왜 polling이 아니라 batch인가** (stock 도메인 raw의 batch 채택 이유와 동일):
- 시점 일관성 — cutoff (`now - 60분`) 기준 명확.
- 운영 가시성 — Spring Batch metadata로 진행/실패/재시작 추적.
- peak 부하 회피, 장애 복구 spike가 정상 트래픽에 안 섞임.

**retry/skip 정책**:
- `OptimisticLockingFailureException`은 retry → skip 둘 다 — 일시 충돌은 재시도, 그래도 안 되면 단건 skip하고 다음 chunk 진행.
- `CustomException`은 skip만 — 도메인 위반(주문이 이미 다른 상태로 전이 등)은 재시도해도 같은 결과라 skip.
- skip 한도 10 = "전체 실패 vs 개별 단건 보류"의 임계.

**ItemReader의 keyset 페이징** (`id > lastId` + `chunkSize`): offset 페이징의 누적 cost 회피. cutoff 시점이 reader 생성 시 1회 고정.

**숫자(chunk 100 / retry 3 / skip 10 / cron 5분)는 모두 기본값** — 부하 테스트 결과나 운영 경험 기반이 아님. 추후 부하 테스트(ADR-016 k6) 결과로 튜닝 예정.

**Writer의 lambda vs `Chunk<>` 주석 — 학습 단계 흔적**:
- 주석 처리된 `Chunk<>` 받는 ItemWriter 방식과 현재 사용 중인 lambda 방식이 함께 남아있음.
- lambda 표현이 학습이 덜 돼서 *직관적으로 와닿지 않음* → 자기 스타일로 코드를 짠 뒤 lambda 표현을 주석으로 남겨둠. 학습이 충분해지면 lambda로 정리할 예정.

**`@Profile("!test")` — 테스트 격리 목적이나 production 코드 책임 분리 측면 자기 비판 있음**:
- `OrderExpirationJobScheduler`에 `@Profile("!test")` 부착 — test profile일 때 스케줄러 자동 실행을 차단해서 테스트 격리.
- 의식적으로 적용한 패턴.
- 다만 *production 코드(스케줄러 클래스)가 테스트 격리 책임까지 떠안는 구조*가 됨. 다른 스케줄러가 추가될 때마다 같은 annotation을 반복해야 함 → 코드 스멜 인지.
- 향후 검토 대안:
  - (B) `@TestConfiguration`에서 스케줄러 Bean을 mock/no-op로 override
  - (C) 별도 `SchedulerConfig` 클래스로 등록 분리하고 그 config에만 `@Profile("!test")` 부착 → 스케줄러 자체 코드는 깨끗하게 유지
- 통합 테스트에서 만료 흐름 검증은 `JobLauncher` 수동 launch로 수행.

### 6. `@Version` 낙관락 + 비관락 공존 (Order 엔티티)

**결정 요약**: `Order`에 `@Version private Long version`. 만료 batch가 `retry(OptimisticLockingFailureException, 3)`로 OLE를 처리.

**현재 코드 동작**:
- **만료 흐름** (`OrderExpirationService.expireOrder`): `findByIdWithItems()` 일반 SELECT (FOR UPDATE 아님) → `order.cancel()` → outbox 이벤트 발행 → commit 시 Order update + `@Version` 검사
- **결제 흐름** (`PaymentApprovalService`): `findByMerchantPayKeyForUpdate` 비관락 → `order.completePayment()` → save
- **batch 정책** (`OrderExpirationBatchConfig`): `retry(OptimisticLockingFailureException, 3)` + `skip(OLE, 10)`

**`@Version`이 빛나는 race window — 실제 코드상 성립**:
- 시나리오: 만료 batch와 결제 콜백이 같은 Order를 동시에 다룸
  - t1: 만료 batch가 Order load (version=1, status=INIT)
  - t2: 결제 콜백이 같은 Order에 FOR UPDATE 획득 → `completePayment()` → save (version=2) → commit
  - t3: 만료 batch가 `cancel()` → save 시도 → `WHERE version=1` update 0 rows → **OLE**
  - batch retry → load (version=2, status=PAID) → `cancel()` 호출 → 도메인 예외 (`status != INIT`) → skip 정책으로 단건 보류
- 이 race가 *실제로 발생 가능*하며, `@Version`이 *만료/결제 race를 안전하게 catch*하는 핵심 메커니즘.

**비관락만으로는 안 되는 이유**:
- 만료 batch도 FOR UPDATE로 잡으면 결제 흐름과 *직렬화*되어 race가 사라짐.
- 하지만 chunk 단위 FOR UPDATE는 *chunk 처리 시간 내내 락 보유* → 정상 주문/결제 흐름이 차단됨.
- 그래서 만료는 *FOR UPDATE 안 잡고 낙관락 의존*, 결제는 *FOR UPDATE 비관락*. 두 전략이 만나는 race window를 `@Version`이 잡음.

**`OrderConcurrencyService.createOrderWithOptimisticLock`** — 동시성 비교 실험용. production은 비관락 정렬을 쓰지만, 낙관락 전략을 *비교 가능하게* 남겨둠.

### 7. `OrderConcurrencyService` — production 흐름과 격리된 학습 코드

**결정 요약**: 동시성 전략 7개(`withoutLock`, `synchronized`, `synchronizedAndTransaction`, `reentrantLockAndTransaction`, `optimisticLock`, `pessimisticLock`, `pessimisticLockOrdered`, `pessimisticLockBatch`)를 별도 Service로 분리. production `createOrder`는 `OrderCreateService` → `OrderCreateProcessor` (pessimisticLockOrdered와 동등)만 사용.

**현재 사용처 — production 0, 테스트 5개 클래스만** (grep 확정):
- `OrderApplicationServiceTest`
- `OrderConcurrencyServiceDebugTest`
- `OrderConcurrencyServiceTest`
- `OrderConcurrencyServiceDeadlockTest`
- `OrderConcurrencyServiceDeadlockMysqlTest`

**왜 분리인가 — 학습 의도**:
- DDD 회고: "동시성 전략 비교와 검증 목적의 생성 흐름은 production 주문 생성 흐름과 섞지 않고 `OrderConcurrencyService`로 분리".
- 실험 코드를 production service에 두면 public API가 비대해지고 호출자 혼란.
- 사용자가 *각 락 전략의 동작·문제·트레이드오프를 학습하기 위해* 명시적으로 보존한 비교 코드. 부하 테스트/벤치마크에서 strategy별 호출도 가능.

**현재 위치 — 학습 흔적**:
- *운영 진입 전 단계*라 학습 코드를 production에 의도적으로 남겨둠.
- 운영 진입 시 → 학습 내용을 문서로 옮기고, 코드는 ADR-009 패턴(미사용 추상화 제거 + git history 활용)으로 정리 예정.
- 즉 현재는 *임시 보존 상태*이고 *제거 예정*.

**네이밍 예외 — 의도적**:
- `decreaseWithPessimisticLock`처럼 *구현 전략을 public 메서드 이름에 노출*하지 말자는 stock 도메인 회고 원칙 vs `createOrderWithPessimisticLock` / `pessimisticLockOrdered` 등의 strategy 노출 네이밍.
- 실험·비교가 *목적*인 service라 strategy를 메서드 이름으로 드러내는 것이 *오히려 명확함*. 일관성 원칙의 의도적 예외.

---

## 도메인 경계에서 배운 것

### DDD 이관 회고 (`docs/ddd/order-ddd-migration-retrospective.md`)

- *DDD 구조 추가* 와 *legacy 삭제* 를 분리. legacy controller 는 bean 등록만 끊고 코드는 일단 둠. 이후 별도 작업으로 production/test 참조 제거.
- `OrderCommandService` 한 덩어리 ✗ → 유스케이스 단위(`OrderCreateService`, `OrderCancelService`, `OrderExpirationService`, `OrderQueryService`, `OrderConcurrencyService`) 분리.
- `batch` 패키지는 *Job/Step/Reader/Writer 구성만*. 비즈니스 유스케이스는 application service 에. 재사용성 + 단일 책임.
- `OrderItem` 은 독립 유스케이스 없음 → 별도 도메인 패키지 X, `order.domain.OrderItem` 으로 통합.
- domain repository(`OrderRepository` interface) ← infrastructure (`JpaOrderRepository`, `OrderRepositoryAdapter`) 의 표준 패턴.

### order-idempotency 작업 회고 (`docs/tasks/order-idempotency/retrospective.md`)

회고 내용 자체는 *harness execute.py 도구 운영* 이슈가 주제 — order 도메인 결정과는 무관.
- harness execute.py 가 worktree 안 파일을 못 찾는 ROOT 계산 버그. 수동 구현으로 우회.
- 도구 사용 전에 동작 방식(ROOT, worktree lifecycle)을 미리 검증.

### 코드 곳곳의 *작은 신호*

- `OrderItem` 에 *가격 컬럼 없음* — 주문 시점 가격 스냅샷이 아님. 현재는 `product.getPrice() * quantity` 를 totalPrice 에 누적. 주문 후 상품 가격 변경 시 totalPrice 와 product price 가 불일치.
  - 코드 주석: `// 가격도 넣어야 하나? (구매했을때 기준의 가격이 있어야 하지 않나.. 세일,, 등등.. product의 price는 변동하지 않나) 추후 고려하기`
  - 이 TODO는 쿠폰/정산/리뷰 기능 도입 전에 반드시 정리해야 하는 부채.
- `OrderStatus.RECEIVED`, `COMPLETED` 가 enum 에 정의됐지만 실제 전이 코드 없음 — 향후 단계용 placeholder.
- `Order.assignMerchantPayKey(...)` 는 *null 일 때만 set* 하는 멱등 setter. 결제 흐름에서 동일 주문에 여러 번 호출돼도 첫 값만 유지.

---

## 다시 본다면

- (사용자 작성 영역)

---

## 다음 단계 / 미해결

- **Issue #171** (fix: PROCESSING 상태 동시 요청 안전망 500) — 결정 1번에서 발견. 빠른 시일 내 개선 예정.
- **Issue #172** (feat: FAILED 캐싱 정책 결정 및 구현) — 결정 1번에서 발견. 정책 결정부터 필요.
- **Issue #173** (refactor: AFTER_COMMIT 리스너 비동기 전환) — 결정 2번에서 발견. ADR-019 fallback 검증 함께.
- **`OrderConcurrencyService` 처리**: 학습 단계 흔적. 운영 진입 시 학습 내용 문서로 옮기고 코드 제거 (ADR-009 패턴).
- **`OrderItem` 가격 스냅샷**: 코드 주석 TODO 정리. 쿠폰/정산 도입 전 처리 필요.
- **만료 batch 숫자 튜닝**: chunk 100 / retry 3 / skip 10 / cron 5분 / TTL 10분 / 만료 cutoff 60분 — 부하 테스트 후 조정.
- **`Order.@Version` 의 실제 race 사례 정리**: 어떤 흐름에서 OLE 가 실제로 발생하는지 데이터 수집.
- **traceId 전파**: Spring Event 5개 이상 늘면 Multicaster wrapping 으로 전환 (ADR-019 약속).
- **`@Profile("!test")` 구조 개선**: 스케줄러마다 반복 부착 패턴을 (B) `@TestConfiguration` mock override 또는 (C) 별도 `SchedulerConfig` 분리로 정리.
- **Writer lambda 정리**: 학습 완료 후 주석 처리된 `Chunk<>` 방식 제거하고 lambda로 일관 정리.

---

## 인용

- `[[commerce-backend/docs/ADR.md#ADR-002]]` — 멱등성 Redis + RDB unique 이중 보장
- `[[commerce-backend/docs/ADR.md#ADR-005]]` — Redis 캐싱은 AFTER_COMMIT
- `[[commerce-backend/docs/ADR.md#ADR-008]]` — 트랜잭션 분리 (`NOT_SUPPORTED` 패턴)
- `[[commerce-backend/docs/ADR.md#ADR-011]]` — find-first 정책 (멱등 catch 폐기)
- `[[commerce-backend/docs/ADR.md#ADR-017]]` `[[commerce-backend/docs/ADR.md#ADR-019]]` — traceId 전파 (Kafka / Event / Outbox)
- `[[commerce-backend/docs/tasks/order-idempotency/prd.md]]` — 멱등성 task PRD
- `[[commerce-backend/docs/tasks/order-idempotency/adr.md]]` — 멱등성 task ADR (Processor 분리 근거)
- `[[commerce-backend/docs/tasks/order-idempotency/architecture.md]]` — 멱등성 흐름도
- `[[commerce-backend/docs/ddd/order-ddd-migration-retrospective.md]]` — DDD 이관 회고 (Service 분리, batch 경계)
- `[[commerce-backend/docs/architecture.md]]` — order 데이터 흐름·서비스 목록
- `[[commerce-backend/docs/db-schema.md#tbl_order]]` — 스키마 + 복합 unique
- `[[raw/sessions/backend/2026-05-29-stock-domain-overview]]` — 비관락·outbox 비동기 사슬 (order 와 짝)
