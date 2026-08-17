---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, idempotency, redis, event-listener, after-commit, trace-id, transaction-boundary]
created: 2026-05-29
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# 주문 멱등성 캐싱 — @TransactionalEventListener(AFTER_COMMIT)로 Redis 분리

## 컨텍스트·문제(트랜잭션 안 Redis 호출의 위험)

주문 저장이 성공하면 Redis에 멱등 완료 마커(complete)를 남겨 다음 재요청이 DB를 거치지 않고 캐시에서 흡수되게 한다. 이 Redis 쓰기를 Order 저장 트랜잭션 **안에서** 하면, Redis 장애가 트랜잭션 전체 롤백을 일으켜 주문 자체가 사라진다. 멱등성 캐싱은 정합성이 아니라 편의이고 최종 보장은 RDB([[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]])인데, 편의 계층의 장애가 정본 데이터를 되돌리는 건 본말전도다.

## 결정과 선택 이유(AFTER_COMMIT·레이어 경계)

Order 저장 트랜잭션 안에서 `OrderIdempotencyCacheEvent`를 발행하고, `RedisOrderIdempotencyStore.handle()`이 `@TransactionalEventListener(AFTER_COMMIT)`으로 받아 RDB 커밋 이후 Redis complete를 수행한다.

- **Redis 작업을 RDB 커밋 이후로 미룬다** — Redis 장애가 나도 이미 커밋된 주문은 무사하다.
- **왜 `TransactionSynchronizationManager` 직접 사용이 아니라 이벤트 리스너인가**: `@TransactionalEventListener`가 DDD 레이어 경계 유지에 자연스럽다. Application이 Infrastructure(Redis adapter)를 직접 알지 않아도 되고, 커밋 이후 후처리라는 의도가 애너테이션으로 드러난다.

## traceId 명시 동봉(pushTraceIdIfMissing)

- 이벤트 객체가 publisher 시점의 `LogContext.getTraceId()`를 필드로 동봉하고, listener가 MDC에 push한다.
- AFTER_COMMIT은 기본 동기 실행이라 호출 스레드 MDC에 traceId가 살아있으면 그대로 보존하고, 없을 때만 이벤트의 traceId를 fallback으로 push(`pushTraceIdIfMissing`)한다.
- 이 명시적 동봉은 Outbox 이벤트가 traceId를 컬럼으로 저장했다가 relay 시 MDC로 복원하는 [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]의 traceId 전파와 같은 결 — 비동기/이벤트 경계는 traceId가 자동 전파되지 않으므로 명시 동봉이 필요하다.

## 미해결(동기/비동기 오인 #173)

솔직한 메모: `@TransactionalEventListener(AFTER_COMMIT)`가 **비동기로 동작한다고 잘못 인식**하고 도입했다. 실제는 **동기** — 호출 스레드에서 commit 직후 그대로 실행된다(코드 주석에도 명시). 즉 클라이언트 응답 latency에 Redis `complete()` 호출 1회가 포함된다. 영향 자체는 작지만(Redis 호출이 빠름) AFTER_COMMIT의 원래 의도(트랜잭션과 분리된 별개 흐름)는 비동기까지 가야 완성된다. 위 traceId 동봉이 사실 이 비동기 전환을 대비한 설계이며, `pushTraceIdIfMissing` fallback은 비동기 전환 후에야 진가를 발휘한다. **Issue #173**(AFTER_COMMIT 리스너 비동기 전환, `@Async`/`ApplicationEventMulticaster`에 TaskExecutor 주입)로 등록.

## 트레이드오프(커밋~캐싱 gap, 이벤트별 traceId 동봉 반복)

- RDB 커밋 ~ Redis 캐싱 사이 짧은 gap에 동일 키 요청이 오면 MISS → INSERT 시도 → unique 위반 → DB 재조회(find-first로 흡수). 정확성은 OK.
- 이벤트 객체마다 `traceId` 필드를 추가하는 반복이 든다. Spring Event 사용처가 5개 이상 되면 Multicaster wrapping으로 재검토하기로 했다(현재는 이 이벤트 1개뿐이라 동봉으로 충분).

> [!warning] EVOLUTION — 05-29 스냅샷의 완료캐싱 설계는 이후 뒤집힘
> 이 노트의 AFTER_COMMIT "완료 캐싱" 설계(성공 후 Redis에 완료 마커를 남겨 재요청을 캐시에서 흡수)는 이후 pr-180에서 **inflight 차단 전용**으로 단순화됐다 — [[주문-멱등-캐시-inflight-차단-전용]]. 즉 Redis 캐시의 역할이 "완료 결과 캐싱"에서 "동시 진행 중(inflight) 요청 차단"으로 축소됐고, 완료 판정은 RDB unique + find-first에 더 명확히 위임됐다. AFTER_COMMIT으로 트랜잭션과 분리한다는 근간 아이디어는 남지만, 무엇을 캐싱하는지는 pr-180 노트를 정본으로 본다. 회원가입에서 AFTER_COMMIT을 진지 검토했다가 strict 정책과 양립불가로 제외한 [[회원가입-not-supported-트랜잭션-분리]]와 대비해 읽으면 AFTER_COMMIT의 적용 조건이 선명해진다.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
