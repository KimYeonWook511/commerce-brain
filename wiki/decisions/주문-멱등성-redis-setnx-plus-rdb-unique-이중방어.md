---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, idempotency, redis, concurrency, db-unique, race-condition]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# 주문 멱등성 — Redis SETNX 1차 + RDB 복합 unique 2차 이중방어

## 컨텍스트·문제(핵심 기능의 멱등성 요구)

주문 생성은 `Idempotency-Key` 헤더를 필수로 받는다. 같은 키로 온 재요청이 새 주문을 만들면 안 되고, 동시 요청·재시도·네트워크 중복 어느 경우에도 정확히 한 번만 생성돼야 한다. 핵심은 **주문이 서비스의 핵심 기능이라 Redis가 죽어도 멱등성이 깨지면 안 된다**는 요구다. 캐시 계층의 가용성에 정합성을 의존시킬 수 없다.

## 검토한 대안(Redis 단독 / RDB 단독)

- **Redis 단독(SETNX만)** — 빠르고 정상 흐름 대부분을 흡수하지만, TTL 만료·Redis 장애라는 구조적 빈틈에서 멱등성이 통째로 무너진다. 캐시가 유일한 진실이면 캐시 장애 = 정합성 장애.
- **RDB unique 단독** — 정합성은 확실하지만 정상 재요청까지 매번 DB를 때려 부하가 오르고, hit 응답이 느리다.
- **이중방어(채택)** — Redis `setIfAbsent`(SETNX)로 1차 reserve해 정상 흐름을 흡수하고, `(member_id, idempotency_key)` RDB 복합 unique로 최종 보장. Redis 1차는 DB 부하 절감·빠른 hit, RDB 2차는 TTL 만료/Redis 장애의 구조적 빈틈을 메운다.

## 결정과 선택 이유(이중방어·복합키·NULL 허용)

- **이중방어를 택한 결정적 동기**: Redis가 죽어도 주문은 동작해야 하고 멱등성도 유지돼야 한다. RDB unique가 그 최종 보루다. 캐싱은 정합성이 아니라 편의이고 RDB가 최종 보장이라는 원칙은 [[주문-멱등성-캐싱-after-commit-이벤트-분리]]와 짝을 이룬다.
- **왜 `(member_id, idempotency_key)` 복합 키인가**: 클라이언트가 생성한 UUID의 충돌 가능성을 멤버 격리로 한 겹 더 낮추기 위함. UUID 자체 충돌 확률은 극히 낮지만 안전 마진을 더한다. (복합 unique의 컬럼 length 명시 등 스키마 컨벤션은 [[multi-column-unique-length-명시-컨벤션]] 참조.)
- **왜 `idempotency_key` NULL 허용인가**: 멱등성 없는 학습용 경로([[order-concurrency-service-학습코드-격리]])와 호환하기 위함 + MySQL이 NULL을 unique 비교에서 제외한다는 동작을 활용. 락이 아니라 DB unique로 동시성을 잡는 이 방향은 [[재고차감-동시성-비관락과-productid-정렬]]·payment 도메인의 통일 원칙과 같은 결이다.

## 현재 흐름과 숨은 디테일(find-first 통일, NOT_SUPPORTED, clear on rollback)

- `reserve()`가 false를 반환하는 경로는 둘: Redis에 이미 마커가 있어 `setIfAbsent`가 false거나, Redis 장애로 adapter가 `DataAccessException`을 catch해 false로 fallback할 때. 둘 다 DB 경로로 진입한다.
- **`OrderCreateService`가 `NOT_SUPPORTED`인 이유**: `DataIntegrityViolationException`을 catch한 뒤 같은 트랜잭션에서 DB 재조회가 불가하다(rollback-only 마킹). 그래서 실제 저장은 `OrderCreateProcessor`를 별도 빈(`@Transactional`)으로 떼어 돌리고, Service는 트랜잭션 없이 분기만 한다 → Processor 실패 시 새 트랜잭션으로 재조회 가능. 회원가입에서 트랜잭션을 떼어낸 [[회원가입-not-supported-트랜잭션-분리]]와 동일 패턴.
- **find-first 통일**: 초기 설계의 `DuplicateKeyException` catch 분기는 "DB unique 위반은 안전망 500으로 위임하고 정상 흐름은 사전 `find`로 처리한다"([[find-first-write-not-check-db-unique-멱등]])로 폐기됐다. 사전 `find`가 정상 멱등 재요청을 흡수하고, race window 충돌만 안전망 500으로 보낸다.
- **`clear()` on rollback**: race 충돌 RuntimeException 시 Redis의 `PROCESSING` 마커를 지워 다음 정당한 재시도가 막히지 않게 한다.

## 미해결·후속(#171 PROCESSING 500, #172 FAILED 캐싱, TTL 10분 vs 만료 60분)

- **Issue #171 — PROCESSING 동시 요청이 안전망 500을 받음**: 같은 키로 Req A·B가 거의 동시 진입 시, Req B가 `getCompletedOrderId()` miss(COMPLETED만 hit으로 봄) → 사전 find empty(A commit 전) → INSERT → unique 위반 → `clear()` 후 rethrow → 500. 게다가 Req B의 `clear()`가 Req A의 PROCESSING 마커까지 통째로 지워 후속 Req C도 같은 패턴으로 500이 반복된다. 이상적으로는 PROCESSING을 인식해 "처리 중"(예: 409) 응답 + 조건부 clear로 가야 한다.
- **Issue #172 — FAILED 캐싱 미구현**: `OrderIdempotencyStatus.FAILED`가 정의만 있고 `parseCompletedOrderId()`가 `COMPLETED:` 접두사만 인식한다. 실패 캐싱 정책 결정부터 필요.
- **두 만료의 공존**: TTL 10분(`order.idempotency.ttl-seconds:600`)은 PG(네이버페이) 결제 처리 시간 기반 멱등 키 활성 기간이고, 만료 cutoff 60분(`order.expiration.minutes:60`)은 좀비 주문을 정리하는 배치 기준([[주문만료-spring-batch-chunk-retry-skip]])이다. 의미가 다른 두 숫자이며, TTL 동기가 기억 기반 재구성이라 코드 개선 시 의도를 다시 정리할 예정.

## 트레이드오프

- TTL 만료 후 재요청 시 재고 차감 → unique 위반 → 롤백이 드물게 발생한다(정확성엔 문제 없음).
- `idempotency_key` NULL 허용은 학습 경로 호환을 위해 MySQL의 NULL-unique 제외 동작에 의존한다.

> [!note] 이후 진화
> 이 이중방어(SETNX 1차 + RDB unique 2차)의 근간은 유지되지만, 캐시 부분은 pr-180 계열에서 정제됐다 — [[주문-멱등-캐시-inflight-차단-전용]](완료캐싱 → inflight 차단 전용)와 [[redis-장애-멱등캐시-fallback-boolean-예외분리]](장애 fallback을 boolean/예외로 분리). 최종 보장이 RDB unique라는 원칙은 그대로다.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
