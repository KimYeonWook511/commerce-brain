---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

# 주문(order) 도메인 개요 — 엔티티·멱등성·동시성·만료의 설계 결정 지도

주문 도메인의 구조와 그 안에 박혀 있는 핵심 설계 결정을 한 번에 훑는 세션이다. 주문 생성의 멱등성(Redis 1차 + RDB unique 2차), 재고 차감의 동시성(비관적 락 + 정렬), 취소·만료의 비대칭 처리, 만료 배치, 낙관락과 비관락의 공존, 그리고 production 흐름과 격리해 남겨둔 동시성 학습 코드까지 — 각 결정이 "왜 이 형태인가"와 트레이드오프·미해결을 함께 담는다. 코드를 직접 확인해 현재 동작을 앵커링했다.

## 도메인 구조

### 엔티티·테이블

- `Order : OrderItem = 1:N` (Aggregate). `OrderItem`은 독립 유스케이스가 없어 **별도 최상위 도메인 패키지를 만들지 않고** `Order` aggregate 내부 구성요소(`order.domain.OrderItem`)로 묶었다 (DDD 이관 회고에서 확정).
- `Order : Payment = 1:1` — 결제가 주문을 참조하는 방향. 주문은 결제 흐름에서 채워지는 `merchantPayKey`를 unique 로 보유한다.
- `Order : Member = N:1`.
- `tbl_order` 주요 컬럼: `id`, `version`(낙관락 `@Version`), `member_id`, `total_price`, `status`, `merchant_pay_key`(unique), `idempotency_key`(nullable).
- unique 제약 2개: `merchant_pay_key` 단일 unique, `uk_order_member_idempotency (member_id, idempotency_key)` 복합 unique. `idempotency_key`는 NULL 허용 — 멱등성 없는 경로(아래 동시성 학습용 `OrderConcurrencyService`)와 호환하기 위함 + MySQL 은 NULL 을 unique 비교에서 제외한다는 동작을 활용.

### 상태 (`OrderStatus`)

- enum 값은 `INIT`, `CANCELED`, `RECEIVED`, `PAID`, `COMPLETED` 5개. 실제 전이는 `INIT → CANCELED`(`order.cancel()`), `INIT → PAID`(`order.completePayment()`)만 코드에 존재한다. `RECEIVED`, `COMPLETED`는 enum 정의만 있고 이들로 전이하는 코드가 없다 — 향후 단계용 placeholder.
- 상태 전이·검증은 도메인 메서드에 캡슐화했다: `order.cancel()`, `order.completePayment()`, `order.checkPayable()`. 모두 `status != INIT` 이면 도메인 예외를 던진다(`checkPayable()`은 전이 없이 선조건만 검증). 결제(payment)의 `succeed`/`fail`이 상태 전이를 도메인 메서드 안에서 명시적으로 검증하는 것과 같은 결의 *명시적 선조건 검증* 패턴이다.

### application 서비스 (유스케이스 책임 단위 분리)

| Service | 책임 | 트랜잭션 |
|---|---|---|
| `OrderCreateService` | 멱등성 분기 only | `@Transactional(NOT_SUPPORTED)` (트랜잭션 없음) |
| `OrderCreateProcessor` | 실제 재고 차감 + 주문 저장 + 이벤트 발행 | `@Transactional` |
| `OrderCancelService` | 사용자 주도 취소 (동기 재고 복구) | `@Transactional` |
| `OrderExpirationService` | 만료 처리 (취소 + outbox 이벤트) | `@Transactional` |
| `OrderQueryService` | `merchantPayKey`(+`memberId`) 단건 조회 (결제 흐름용) | `readOnly` |
| `OrderConcurrencyService` | 동시성 전략 비교 *학습용*, production 흐름 아님 | 전략별 `@Transactional` |

- `OrderCommandService` 한 덩어리로 두지 않고 유스케이스 책임 단위로 쪼갠 것은 DDD 이관 회고에 명시된 결정이다. 한 서비스에 생성·취소·만료·조회·동시성 실험을 모두 두면 public API 가 과하게 넓어진다.
- `OrderConcurrencyService`는 production 의 `createOrder` 흐름과 *섞지 않고* 분리했다 — 학습용 strategy 노출이 *목적*이라 strategy 네이밍이 의도적으로 유지된다(뒤 "동시성 학습 코드 격리" 참조).

### 만료 배치

- 만료 처리는 `OrderExpirationBatchConfig` (Spring Batch, chunk = 100, default cron `0 */5 * * * *`).
- Reader: keyset 페이징 — `id > lastId` 로 다음 청크를 읽고 `status = INIT AND createdAt < cutoff` 조건(`findExpiredOrdersAfterId`). offset·count 없음. cutoff 는 job 파라미터로 reader 생성 시 1회 고정(`now - 60분`).
- Writer: chunk 안에서 `OrderExpirationService.expireOrder(orderId, requestedAt)` 를 단건씩 호출 = chunk 트랜잭션 안에서 N건 처리.
- fault tolerance: `faultTolerant` + `retry(OptimisticLockingFailureException, 3)` + `skip(OptimisticLockingFailureException | CustomException, 10)`.
- 비즈니스 유스케이스(`expireOrder`)는 batch 패키지에 두지 않고 application service 로 두어 *scheduler / admin / 수동 복구 흐름에서도 재사용 가능*하게 설계했다 (DDD 이관 회고 명시). batch 패키지는 Job/Step/Reader/Writer 구성만 담당한다.

### 외부 경계

- Stock: `StockInventoryService.decrease` / `increase` 동기 호출 (주문 생성 / 사용자 취소 경로).
- Outbox → Stock: `OutboxService.createStockRestoreOutboxEvent(...)` 비동기 발행 (만료 경로만).
- Redis: `OrderIdempotencyStore` port → `RedisOrderIdempotencyStore` adapter.
- Payment: `OrderQueryService`를 통해 결제가 `merchantPayKey`로 Order 를 가져간다.

## 결정한 것

### 1. 멱등성 — Redis 1차 방어선 + RDB unique 제약 최종 보장

**결정 요약:** 주문 생성은 `Idempotency-Key` 헤더 필수. Redis `setIfAbsent`(SETNX)로 1차 reserve, `(member_id, idempotency_key)` RDB unique 로 최종 보장.

**왜 이중 방어인가 — 결정적 동기:**
- **Redis 가 죽어도 주문은 동작해야 한다.** 주문은 핵심 기능이라 Redis 장애 시에도 멱등성 자체가 깨지면 안 된다. 그래서 DB 에서도 멱등성을 보장할 수단이 필요했고, RDB unique 제약이 그 역할.
- Redis 1차의 가치: 정상 흐름 대부분을 흡수 → DB 부하 절감, 빠른 hit 응답.
- RDB 2차의 가치: TTL 만료 / Redis 장애라는 *구조적* 빈틈을 메우고, Redis 가 죽어도 멱등성이 유지됨.

**왜 `(member_id, idempotency_key)` 복합 키인가:**
- 클라이언트가 생성한 UUID 의 충돌 가능성을 한 겹 더 낮추기 위함. UUID 자체로도 충돌 확률은 극히 낮지만 *멤버 격리*로 안전 마진을 더한다.

**흐름 (현재 코드 기준 — `OrderCreateService.createOrder`):**

```
createOrder(command)
 ├─ idempotencyKey 비어있음 → INVALID_REQUEST
 ├─ reserve(memberId, key, ttl)
 │   ├─ true  → attemptCreateOrder
 │   └─ false (이미 PROCESSING / COMPLETED 마커 존재, 또는 Redis 장애 fallback)
 │       ├─ getCompletedOrderId() hit → DB findById → 반환 (source=redis)
 │       │      (hit 인데 DB 에 없으면 ORDER_NOT_FOUND)
 │       └─ miss → attemptCreateOrder
attemptCreateOrder
 ├─ findByMemberIdAndIdempotencyKey 사전 체크 → 있으면 Redis complete 갱신 후 반환 (source=db)
 ├─ 없으면 OrderCreateProcessor.execute() (별도 @Transactional)
 │   ├─ 성공 → publishEvent(OrderIdempotencyCacheEvent) → AFTER_COMMIT 에서 Redis complete
 │   └─ RuntimeException (race 충돌 시 DataIntegrityViolationException 포함)
 │       → orderIdempotencyStore.clear() 후 rethrow (안전망 500)
```

`reserve()`가 false 를 반환하는 경로는 두 가지다: Redis 에 이미 마커가 있어 `setIfAbsent`가 false 이거나, Redis 장애 시 adapter 가 `DataAccessException`을 catch 해 false 로 fallback 시킬 때. 둘 다 DB 경로로 진입한다.

**숨은 디테일:**
- `OrderCreateService`가 `@Transactional(NOT_SUPPORTED)`인 이유: `DataIntegrityViolationException`을 catch 한 뒤 *같은 트랜잭션에서 DB 재조회가 불가*하다(rollback-only 마킹). 그래서 실제 저장은 `OrderCreateProcessor`를 별도 빈으로 떼어 `@Transactional`로 돌리고, Service 자체는 트랜잭션 없이 분기만 한다 → Processor 실패 시 Service 가 새 트랜잭션으로 재조회할 수 있다. 회원가입에서 트랜잭션을 `NOT_SUPPORTED`로 떼어낸 것과 동일한 패턴.
- 초기 멱등성 설계에 있던 `DuplicateKeyException` catch 분기는, 이후 "DB unique 위반은 안전망 500 으로 위임하고 정상 흐름은 사전 `find`로 처리한다"(find-first 패턴)로 통일되며 폐기됐다. 사전 `find`가 정상 멱등 재요청을 흡수하고, race window 충돌만 안전망 500 으로 보낸다.
- `clear()` on rollback: race 충돌로 인한 RuntimeException 시 Redis 의 `PROCESSING` 마커를 지워야 다음 정당한 재시도가 막히지 않는다.

**현재 코드의 미해결·이상함 (Issue 등록됨, 후속 개선 예정)**

코드 직접 확인 결과(`OrderCreateService`, `RedisOrderIdempotencyStore`, `OrderIdempotencyStatus`):

- **TTL = 600초 (10분)** — `@Value("${order.idempotency.ttl-seconds:600}")`.
  - *기억으로 재구성한 동기:* 네이버페이 결제 처리 시간(약 10분)에 맞췄다. PG 측에서 10분 안에 결제가 완료되지 않으면 그 주문은 PG 측 만료 대상이 되어 사용자도 새 시도로 넘어가므로, 같은 멱등 키 재사용을 차단할 이유가 사라지는 시점이 10분이라는 논리.
  - **두 개의 다른 만료가 공존한다:**
    - TTL 10분 = PG 결제 처리 시간 기반 *멱등 키 활성 기간*.
    - 만료 cutoff 60분 (`order.expiration.minutes:60`) = 별도 정책. PG 응답 없이 남은 좀비 주문을 정리하는 배치 기준.
  - 두 숫자의 의미가 다르다. TTL 동기가 기억 기반 재구성이라, 코드 개선 시점에 의도를 다시 정리하고 정확한 코멘트를 코드에 남길 예정.
- **`FAILED` enum 이 정의만 있고 어디서도 set 되지 않음** — `OrderIdempotencyStatus.FAILED`는 정의돼 있으나, `parseCompletedOrderId()`가 `COMPLETED:` 접두사만 인식한다. 실패 캐싱은 미구현이다. **Issue #172**(주문 멱등성 실패 캐싱 정책 결정 및 구현)로 등록.
- **PROCESSING 동시 요청 → 안전망 500:**
  - 시나리오: 같은 `idempotencyKey`로 Req A·B 가 거의 동시 진입.
  - Req A: reserve true → `Processor.execute()` 진행 중 (트랜잭션 commit 전).
  - Req B: reserve false(PROCESSING 마커) → `getCompletedOrderId()` miss(COMPLETED 만 hit 으로 봄) → `attemptCreateOrder` → 사전 find empty(Req A commit 전) → INSERT 시도 → unique 위반 → `clear()` 후 rethrow → **안전망 500**.
- **추가 race window — Req B 의 `clear()`가 Req A 의 PROCESSING 마커까지 지운다:**
  - `clear()` = `redisTemplate.delete(key)`로 키를 통째 삭제.
  - Req A 의 AFTER_COMMIT 이벤트가 commit 후 `complete()`로 키를 다시 만들지만, Req A commit 전에 Req C 가 또 오면 → reserve true(PROCESSING 마커 없음) → 같은 패턴 반복 → 또 500.
- **이상적 흐름과의 갭:**
  - PROCESSING 상태일 때 같은 키 요청은 *대기* 또는 *"현재 처리 중"* 응답이 자연스럽다.
  - 지금은 동시 요청이 모두 500 을 받는 구조라 클라이언트가 *"처리 중인지 실패인지" 구분 불가*.
  - 향후 개선 방향: `getCompletedOrderId`를 *PROCESSING 도 인식*하게 확장 + 적절한 응답(예: 409 Conflict + "처리 중" 메시지) + retry 정책 명확화 + Req B catch 의 `clear()` 동작 재검토(조건부 clear?). **Issue #171**(주문 멱등성 PROCESSING 상태 동시 요청 안전망 500 응답 개선)로 등록.

**트레이드오프:**
- TTL 만료 후 재요청 시 재고 차감 → unique 위반 → 롤백이 드물게 발생한다. 정확성에는 문제 없음.
- `idempotency_key` NULL 허용 → 멱등성 없는 학습용 경로와의 호환을 위함. MySQL 이 NULL 을 unique 비교에서 제외한다는 동작에 의존.

### 2. AFTER_COMMIT 이벤트로 Redis 캐싱 분리

**결정 요약:** Order 저장 트랜잭션 안에서 `applicationEventPublisher.publishEvent(OrderIdempotencyCacheEvent)`를 발행하고, `RedisOrderIdempotencyStore.handle()`이 `@TransactionalEventListener(AFTER_COMMIT)`으로 받아 Redis complete 를 수행한다. (Redis 캐싱은 RDB 커밋 이후에 실행한다는 결정의 구현.)

**왜 트랜잭션 안에서 Redis 를 호출하지 않나:**
- Redis 장애 시 RDB 롤백을 막아야 한다. 멱등성 캐싱은 *정합성이 아니라 편의*이고 RDB 가 최종 보장이다.
- Redis 호출이 동일 트랜잭션 안이면 Redis 장애 → 트랜잭션 전체 롤백 → 주문도 사라진다. Redis 작업을 RDB 커밋 이후로 미루는 것이 기본 정책.
- `TransactionSynchronizationManager`를 직접 쓰는 것보다 `@TransactionalEventListener`가 *DDD 레이어 경계 유지*에 자연스럽다 — Application 이 Infrastructure(Redis adapter)를 직접 알지 않아도 된다.

**traceId 전파 — 비동기/이벤트 경계는 명시적 동봉:**
- 이벤트 객체(`OrderIdempotencyCacheEvent`)가 publisher 시점의 `LogContext.getTraceId()`를 필드로 *동봉*한다. listener 가 MDC 에 push.
- AFTER_COMMIT 은 기본 동기 실행 → 호출 스레드 MDC 에 traceId 가 살아있으면 그대로 보존하고, 없을 때만 이벤트의 traceId 를 fallback 으로 push(`pushTraceIdIfMissing`).

**현재 상태의 미해결 — 동기/비동기 인식 실수 (솔직한 메모):**
- `@TransactionalEventListener(AFTER_COMMIT)`가 *비동기로 동작한다고 잘못 인식*하고 도입했다. 실제는 **동기** — 호출 스레드에서 commit 직후 그대로 실행된다(코드 주석에도 이 사실을 명시해 둠).
- 즉 *클라이언트 응답 latency* 에 Redis 호출 시간(`complete()` 1회)이 포함된다.
- 영향 자체는 작지만(Redis 호출이 빠름), AFTER_COMMIT 의 *원래 의도(트랜잭션과 분리한 별개 흐름)*는 *비동기까지 가야 완성*된다는 점에서 미완이다.
- 향후 비동기로 분리 예정(`@Async` 또는 `ApplicationEventMulticaster`에 TaskExecutor 주입).
- traceId 를 이벤트에 동봉해 둔 설계가 사실 *이 비동기 전환을 대비한 것* — `pushTraceIdIfMissing` fallback 로직은 비동기 전환 후에야 진가를 발휘한다. **Issue #173**(주문 멱등성 캐싱 AFTER_COMMIT 리스너 비동기 전환)로 등록.

**트레이드오프:**
- RDB 커밋 ~ Redis 캐싱 사이의 짧은 gap 에 동일 키 요청이 오면 MISS → INSERT 시도 → unique 위반 → DB 재조회(find-first 로 흡수). 정확성은 OK.
- 이벤트 객체마다 `traceId` 필드를 추가하는 반복 작업이 든다. Spring Event 사용처가 5개 이상 되면 Multicaster wrapping 으로 재검토하기로 했다 — 현재는 이 이벤트 1개뿐이라 동봉으로 충분.

### 3. 비관적 락 + productId 정렬 (batch decrease 기각) — `OrderCreateProcessor`

**결정 요약:** `OrderCreateProcessor.execute()`가 시작하자마자 items 를 `productId` 오름차순으로 정렬한 뒤 stock 을 단건씩 차감한다.

**왜 정렬인가:**
- 같은 두 상품을 서로 다른 순서로 차감하면 → 두 트랜잭션이 `SELECT ... FOR UPDATE`로 서로의 락을 잡아 **상호 대기 데드락**이 난다.
- 모든 트랜잭션이 동일 순서(productId 오름차순)로 락을 잡으면 데드락을 회피한다 (전형적인 lock ordering).

**왜 batch decrease 가 아니라 단건 정렬인가 — 측정 기반 + 안정성 의사결정:**

1. **InnoDB IN 절은 atomic lock acquisition 이 아니다.**
   - 직관적으로는 batch 가 락 보유 시간이 더 짧을 거라 추측했지만,
   - 테스트로 확인한 사실: `SELECT ... FOR UPDATE` 의 IN 절도 모든 락을 atomic 하게 한 번에 잡는 게 아니라, *옵티마이저가 인덱스를 보고 결정한 순서로 순차적으로 row lock 을 획득*한다. 즉 batch 도 단건 정렬과 사실상 같은 lock acquisition 패턴이다.
   - 그래서 batch 의 진짜 이점은 네트워크 round-trip 절감(N→1)과 SQL parsing 비용 절감뿐으로 좁아진다. 락 보유 시간 우위는 없다.
2. **batch 의 락 순서는 옵티마이저·인덱스 구조에 의존 → 잠재적 데드락 위험.**
   - IN 절의 락 획득 순서는 *옵티마이저가 인덱스를 보고 결정*한다. 우리가 IN 에 적은 순서 자체로 보장되는 게 아니다.
   - 인덱스 구조가 바뀌거나 옵티마이저 동작이 바뀌면(MySQL 업그레이드 / 인덱스 추가·삭제 / 통계 변화) **의도한 락 순서가 깨져 데드락이 날 수 있다.**
   - IN 절 정렬을 명시적으로 쓸 수도 있지만 → *native query 문자열 안에 정렬 로직*을 넣어야 하고 → 나중에 수정 시 *문자열을 찾아 고쳐야 하는* 불편 + 컴파일러·IDE refactoring 으로 잡히지 않는다.
3. **단건 정렬의 진짜 우위 — 추상화 깊이와 의존성 안정성.**
   - 정렬 로직이 *Java 코드*(`List.sort()`) — 컴파일러 검증, IDE refactoring 지원.
   - DB 옵티마이저 동작과 무관 — 인덱스 변경·MySQL 업그레이드에도 영향 없음.
   - 각 차감 사이에 도메인 로직을 끼울 수 있다(`Stock.decrease(n)` 단위 검증·예외 분기).
   - 에러 메시지 정밀도 — 상품별 부족 사유를 명시할 수 있다.

→ batch 는 *옵티마이저·인덱스 의존성*이라는 hidden coupling 을 안고, 그것이 *유지보수 시점에 깨질 수 있는 위험*이다. 단건 정렬은 모든 정렬 책임이 *Java 코드 안에 명시*되어 안정성 우위. `OrderConcurrencyService.createOrderWithPessimisticLockBatch`는 비교 실험용으로 남겨두고, production 은 단건 정렬을 채택했다.

**재고 차감의 기본 전략을 비관적 락으로 둔 stock 도메인 결정을 order 측에서 활용한다** — order 는 stock 의 락 전략에 *의존만* 한다. order 자체는 락을 잡지 않는다(단, 결제 흐름용 `findByMerchantPayKeyForUpdate`는 별도로 존재).

### 4. 사용자 취소(동기 복구) vs 만료(Outbox 비동기 복구) — 의도적 비대칭

**결정 요약:**
- `OrderCancelService`(사용자 주도 취소): `stockInventoryService.increase(...)`를 **같은 트랜잭션에서 동기 호출**.
- `OrderExpirationService`(배치 주도 만료): `outboxService.createStockRestoreOutboxEvent(...)`로 **outbox 이벤트만 발행** → Kafka → consumer 가 단건 복구.

**왜 비대칭인가 — 락 보유 시간의 차이:**
- 사용자 취소는 *단건 트랜잭션*. 동기 복구가 락을 잡는 시간 = 단건 차감 시간. 운영상 부담이 없다.
- 만료는 *chunk 트랜잭션*(chunk size 100). chunk 안에서 100건의 stock row 를 직접 `SELECT FOR UPDATE`로 잡으면 → chunk 가 끝날 때까지 모든 stock 락이 유지되고 → 정상 주문 흐름의 차감이 chunk 처리 시간 내내 대기하게 된다.
- 그래서 만료는 *명령만 발행*하고, consumer 가 *건당 별 트랜잭션*으로 풀어준다. 락 보유 시간 = 단건 처리 시간.
- Outbox + Kafka 를 쓰는 이유: "주문 취소 commit + Kafka 발행"의 원자성이 필요하기 때문. DB 트랜잭션과 메시지 발행을 원자적으로 묶으려면 outbox 테이블에 이벤트를 같은 트랜잭션으로 쓰고 relay 가 뒤이어 발행하는 구조가 필요하다.

**`expireOrder` 안에서 직접 stock 복구를 호출하지 않는 이유:** 재고 복구 outbox 이벤트가 *order 도메인의 책임 경계*(취소 사실만 알림)와 *stock 복구의 실제 수행*을 분리해, order 트랜잭션이 짧게 끝나도록 한다.

**사용자 취소를 outbox 로 통일하지 않은 이유 — 단순성 우선의 의도적 판단:**
- 사용자 취소는 *단건이라 락 부담이 없다* → 단순한 동기 호출이 더 낫다는 판단.
- *대칭성*(모든 취소를 outbox 로 통일)보다 *단순성*(부담 없는 흐름은 직관적 동기 호출)을 우선.
- 만료 경로는 chunk 단위 락 부담 때문에 outbox 가 필수, 사용자 취소는 그 부담이 없으므로 동기로 충분.

**관리자 증가 API 와 메서드 시그니처 레벨 분리** (관리자 재고 관리와 변경 이력을 분리한 결정의 보강):
- `AdminStockService.increaseByAdmin(...)` — 관리자 경로(변경 이력 기록 책임).
- `StockInventoryService.increase(...)` — 주문 취소 경로(이력 미기록).
- 이름부터 다른 메서드라 *호출 시점에 책임이 자연스럽게 강제*된다. 같은 이름의 분기 메서드가 아니라 *애초에 별도 메서드*다.

**트레이드오프:**
- 만료 경로는 eventual consistency. 만료 → 복구 사이에 짧은 gap 이 있다.
- 사용자 취소는 즉시 일관성. 두 흐름의 의미가 달라 사용자도 다른 경험을 받는다.

### 5. Spring Batch — chunk = 100, retry/skip 차등

**결정 요약:** 만료는 5분 주기 cron, chunk 100, `faultTolerant` + `retry(OptimisticLockingFailureException, 3회)` + `skip(OptimisticLockingFailureException | CustomException, 10건)`.

**왜 polling 이 아니라 batch 인가:**
- 시점 일관성 — cutoff(`now - 60분`) 기준이 명확.
- 운영 가시성 — Spring Batch metadata 로 진행·실패·재시작을 추적.
- peak 부하 회피 — 장애 복구 spike 가 정상 트래픽에 안 섞인다.

**retry/skip 정책:**
- `OptimisticLockingFailureException`은 retry + skip 둘 다 — 일시 충돌은 재시도하고, 그래도 안 되면 단건 skip 하고 다음 chunk 로 진행.
- `CustomException`은 skip 만 — 도메인 위반(주문이 이미 다른 상태로 전이된 경우 등)은 재시도해도 결과가 같아 skip.
- skip 한도 10 = "전체 실패 vs 개별 단건 보류"의 임계.

**ItemReader 의 keyset 페이징**(`id > lastId` + `chunkSize`): offset 페이징의 누적 cost 를 회피한다. cutoff 시점은 reader 생성 시 1회 고정.

**숫자(chunk 100 / retry 3 / skip 10 / cron 5분)는 모두 기본값** — 부하 테스트 결과나 운영 경험 기반이 아니다. 추후 부하 테스트(k6) 결과로 튜닝 예정.

**Writer 의 lambda vs `Chunk<>` 주석 — 학습 단계 흔적:**
- 주석 처리된 `Chunk<>`를 받는 `ItemWriter` 방식과, 현재 사용 중인 lambda 방식이 함께 남아 있다.
- lambda 표현이 아직 학습이 덜 돼 *직관적으로 와닿지 않아서*, 자기 스타일로 코드를 짠 뒤 lambda 표현을 주석으로 남겨뒀다. 학습이 충분해지면 lambda 로 정리할 예정.

**`@Profile("!test")` — 테스트 격리 목적이나 production 코드 책임 분리 측면의 자기 비판:**
- `OrderExpirationJobScheduler`에 `@Profile("!test")`를 붙여, test profile 일 때 스케줄러 자동 실행을 차단해 테스트를 격리한다. 의식적으로 적용한 패턴.
- 다만 *production 코드(스케줄러 클래스)가 테스트 격리 책임까지 떠안는 구조*가 된다. 다른 스케줄러가 추가될 때마다 같은 annotation 을 반복해야 해 코드 스멜로 인지.
- 향후 검토 대안: (B) `@TestConfiguration`에서 스케줄러 빈을 mock/no-op 로 override, (C) 별도 `SchedulerConfig` 클래스로 등록을 분리하고 그 config 에만 `@Profile("!test")`를 붙여 스케줄러 자체 코드는 깨끗하게 유지.
- 통합 테스트에서 만료 흐름 검증은 `JobLauncher` 수동 launch 로 수행한다.

### 6. `@Version` 낙관락 + 비관락 공존 (Order 엔티티)

**결정 요약:** `Order`에 `@Version private Long version`. 만료 batch 가 `retry(OptimisticLockingFailureException, 3)`로 OLE(낙관락 충돌)를 처리한다.

**현재 코드 동작:**
- **만료 흐름**(`OrderExpirationService.expireOrder`): `findByIdWithItems()` 일반 SELECT(FOR UPDATE 아님) → `order.cancel()` → outbox 이벤트 발행 → commit 시 Order update + `@Version` 검사.
- **결제 흐름**(결제 승인 서비스): `findByMerchantPayKeyForUpdate` 비관락 → `order.completePayment()` → save.
- **batch 정책**(`OrderExpirationBatchConfig`): `retry(OLE, 3)` + `skip(OLE, 10)`.

**`@Version`이 빛나는 race window — 실제 코드상 성립:**
- 시나리오: 만료 batch 와 결제 콜백이 같은 Order 를 동시에 다룬다.
  - t1: 만료 batch 가 Order load (version=1, status=INIT).
  - t2: 결제 콜백이 같은 Order 에 FOR UPDATE 획득 → `completePayment()` → save(version=2) → commit.
  - t3: 만료 batch 가 `cancel()` → save 시도 → `WHERE version=1` update 0 rows → **OLE**.
  - batch retry → load(version=2, status=PAID) → `cancel()` 호출 → 도메인 예외(`status != INIT`) → skip 정책으로 단건 보류.
- 이 race 는 *실제로 발생 가능*하며, `@Version`이 *만료/결제 race 를 안전하게 catch*하는 핵심 메커니즘이다.

**비관락만으로는 안 되는 이유:**
- 만료 batch 도 FOR UPDATE 로 잡으면 결제 흐름과 *직렬화*되어 race 가 사라지긴 한다.
- 하지만 chunk 단위 FOR UPDATE 는 *chunk 처리 시간 내내 락을 보유* → 정상 주문/결제 흐름이 차단된다.
- 그래서 만료는 *FOR UPDATE 를 안 잡고 낙관락에 의존*, 결제는 *FOR UPDATE 비관락*. 두 전략이 만나는 race window 를 `@Version`이 잡는다.

**`OrderConcurrencyService.createOrderWithOptimisticLock`** — 동시성 비교 실험용. production 은 비관락 정렬을 쓰지만, 낙관락 전략을 *비교 가능하게* 남겨뒀다.

### 7. `OrderConcurrencyService` — production 흐름과 격리된 동시성 학습 코드

**결정 요약:** 동시성 전략 8가지(`createOrderWithoutLock`, `createOrderWithSynchronized`, `createOrderWithSynchronizedAndTransaction`, `createOrderWithReentrantLockAndTransaction`, `createOrderWithOptimisticLock`, `createOrderWithPessimisticLock`, `createOrderWithPessimisticLockOrdered`, `createOrderWithPessimisticLockBatch`)를 별도 Service 로 분리했다. production `createOrder`는 `OrderCreateService` → `OrderCreateProcessor`(`createOrderWithPessimisticLockOrdered`와 동등)만 사용한다.

**현재 사용처 — production 0, 테스트 5개 클래스만** (grep 확정):
- `OrderApplicationServiceTest`
- `OrderConcurrencyServiceDebugTest`
- `OrderConcurrencyServiceTest`
- `OrderConcurrencyServiceDeadlockTest`
- `OrderConcurrencyServiceDeadlockMysqlTest`

**왜 분리인가 — 학습 의도:**
- DDD 이관 회고에서 "동시성 전략 비교·검증 목적의 생성 흐름은 production 주문 생성 흐름과 섞지 않고 `OrderConcurrencyService`로 분리"하기로 확정.
- 실험 코드를 production service 에 두면 public API 가 비대해지고 호출자가 혼란스럽다.
- 사용자가 *각 락 전략의 동작·문제·트레이드오프를 학습하기 위해* 명시적으로 보존한 비교 코드다. 부하 테스트/벤치마크에서 strategy 별 호출도 가능.

**현재 위치 — 학습 흔적:**
- *운영 진입 전 단계*라 학습 코드를 production 에 의도적으로 남겨뒀다.
- 운영 진입 시 → 학습 내용을 문서로 옮기고, 코드는 *미사용 추상화는 제거하고 그 존재·제거 이유는 git history 에 맡기는 패턴*(사용처 없는 인터페이스 메서드를 지운 결정과 같은 결)으로 정리 예정.
- 즉 현재는 *임시 보존 상태*이자 *제거 예정*이다.

**네이밍 예외 — 의도적:**
- `decreaseWithPessimisticLock`처럼 *구현 전략을 public 메서드 이름에 노출*하지 말자는 stock 도메인 회고 원칙 vs `createOrderWithPessimisticLock` / `pessimisticLockOrdered` 등 strategy 노출 네이밍의 충돌.
- 실험·비교가 *목적*인 service 라 strategy 를 메서드 이름으로 드러내는 것이 *오히려 명확*하다. 일관성 원칙의 의도적 예외.

## 배운 것

### DDD 이관 회고에서 굳힌 도메인 경계 기준

- *DDD 구조 추가*와 *legacy 삭제*를 분리한다. legacy controller 는 bean 등록만 끊고 코드는 일단 둔 뒤, 별도 작업으로 production/test 참조를 정리한다.
- `OrderCommandService` 한 덩어리 ✗ → 유스케이스 단위(`OrderCreateService`, `OrderCancelService`, `OrderExpirationService`, `OrderQueryService`, `OrderConcurrencyService`) 분리.
- `batch` 패키지는 *Job/Step/Reader/Writer 구성만*. 비즈니스 유스케이스는 application service 에 둔다 → 재사용성 + 단일 책임.
- `OrderItem`은 독립 유스케이스가 없음 → 별도 도메인 패키지 X, `order.domain.OrderItem`으로 통합.
- domain repository(`OrderRepository` interface) ← infrastructure(`JpaOrderRepository`, `OrderRepositoryAdapter`) 의 표준 패턴.

### order-idempotency 작업 회고 — 도메인 결정과는 무관한 도구 이슈

- 회고 내용 자체는 *개발 harness 의 `execute.py` 도구 운영* 이슈가 주제였다(order 도메인 결정과 무관).
- `execute.py`가 worktree 안 파일을 못 찾는 ROOT 경로 계산 버그가 있어 수동 구현으로 우회했다.
- 교훈: 도구를 쓰기 전에 그 동작 방식(ROOT 계산, worktree lifecycle)을 미리 검증해야 한다.

### 코드 곳곳의 작은 신호

- `OrderItem`에 *가격 컬럼이 없다* — 주문 시점 가격 스냅샷을 남기지 않는다. 현재는 `product.getPrice() * quantity`를 `totalPrice`에 누적한다. 주문 후 상품 가격이 바뀌면 `totalPrice`와 현재 product price 가 불일치한다.
  - 코드 주석: `// 가격도 넣어야 하나? (구매했을때 기준의 가격이 있어야 하지 않나.. 세일,, 등등.. product의 price는 변동하지 않나) 추후 고려하기`
  - 이 TODO 는 쿠폰/정산/리뷰 기능 도입 전에 반드시 정리해야 하는 부채.
- `OrderStatus.RECEIVED`, `COMPLETED`는 enum 에만 정의되고 전이 코드가 없다 — 향후 단계용 placeholder.
- `Order.assignMerchantPayKey(...)`는 *null 일 때만 set*하는 멱등 setter. 결제 흐름에서 같은 주문에 여러 번 호출돼도 첫 값만 유지한다.

## 미해결·열린 질문

- **Issue #171** (PROCESSING 상태 동시 요청 안전망 500 개선) — 결정 1번에서 발견. 빠른 시일 내 개선 예정.
- **Issue #172** (FAILED 캐싱 정책 결정 및 구현) — 결정 1번에서 발견. 정책 결정부터 필요.
- **Issue #173** (AFTER_COMMIT 리스너 비동기 전환) — 결정 2번에서 발견. traceId fallback(`pushTraceIdIfMissing`) 검증도 함께.
- **`OrderConcurrencyService` 처리** — 학습 단계 흔적. 운영 진입 시 학습 내용을 문서로 옮기고 코드는 제거(미사용 추상화 제거 + git history 활용 패턴).
- **`OrderItem` 가격 스냅샷** — 코드 주석 TODO 정리. 쿠폰/정산 도입 전 처리 필요.
- **만료 batch 숫자 튜닝** — chunk 100 / retry 3 / skip 10 / cron 5분 / TTL 10분 / 만료 cutoff 60분. 모두 기본값이라 부하 테스트 후 조정.
- **`Order.@Version`의 실제 race 사례 정리** — 어떤 흐름에서 OLE 가 실제로 발생하는지 데이터 수집.
- **traceId 전파** — Spring Event 가 5개 이상 늘면 Multicaster wrapping 으로 전환.
- **`@Profile("!test")` 구조 개선** — 스케줄러마다 반복 부착하는 패턴을 (B) `@TestConfiguration` mock override 또는 (C) 별도 `SchedulerConfig` 분리로 정리.
- **Writer lambda 정리** — 학습 완료 후 주석 처리된 `Chunk<>` 방식을 제거하고 lambda 로 일관 정리.
