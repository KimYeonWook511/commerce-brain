---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [stock, pessimistic-lock, concurrency, optimistic-lock, redis-lock, domain-exception]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-stock-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# 재고 차감 기본 전략 비관적 락 — 낙관락·Redis락·atomic UPDATE 기각

주문 경로의 재고 차감 동시성 전략과, 여러 상품을 한 주문에서 차감할 때의 데드락 회피를 한 노트에 묶는다. 앞부분은 "왜 비관적 락인가"(전략 선택), 뒷부분은 "왜 productId 정렬인가 / 왜 batch decrease가 아닌가"(구현 형태)다. 재고 차감 자체는 [[stock-도메인-구조-개요]]의 세 축 중 하나이고, order는 이 락 전략에 **의존만** 한다([[order-도메인-구조-개요]]).

## 컨텍스트·문제 — 동시 차감 전략

재고는 인기 상품에 동시 주문이 몰려 같은 row에 차감 경합이 잦은 특성이다. 그래서 "차감 실패의 원인(상품 없음 vs 재고 부족)을 서로 다른 도메인 예외로 정확히 갈라 던지면서, 동시성 하에서도 안전하게" 차감할 전략을 골라야 했다.

## 후보 비교 (낙관/Redis/atomic/비관)

- **낙관적 락 ✗** — 충돌이 잦은 특성이라 충돌 시 재시도로 처리하면 retry가 빈발해 서버 부담만 커진다.
- **Redis 분산 락 ✗** — 지금은 단일 DB 환경이라 DB 밖의 외부 동기화 지점이 불필요하다. DB 부하가 심해지는 시점의 도입 후보로만 남긴다.
- **atomic UPDATE(`WHERE quantity >= N`) ✗** — affected rows가 0일 때 "상품 없음"과 "재고 부족"을 **구분할 수 없어** 둘을 다른 도메인 예외로 못 던진다. 구분하려 사전 `SELECT`를 붙이면 결국 같은 row에 락이 걸려 atomic의 이점이 사라진다.
- **비관적 락 ✓** — `SELECT ... FOR UPDATE`로 row를 잡은 상태에서 "상품 존재 여부 → 재고 충분 여부"를 도메인 객체 안에서 명시 검증하고 각각 다른 예외로 던질 수 있다. 같은 트랜잭션이라 실패 시 자동 롤백된다.

## 비관락 선택 이유 — 도메인 예외 정밀 구분

비관락을 고른 결정적 근거는 성능이 아니라 **실패 원인을 도메인 예외로 정밀하게 구분**할 수 있다는 점이다. row를 잡은 채 도메인 객체 안에서 존재·충분을 단계별로 검증하니, "상품이 없다"와 "재고가 부족하다"가 서로 다른 예외로 갈려 나가고, 상품별 부족 사유까지 에러 메시지에 담긴다. atomic UPDATE는 affected rows라는 단일 신호로 이 둘을 합쳐버려 이 정밀도를 낼 수 없다.

이 판단은 [[payment-동시성-unique-vs-lock-gap-lock회피]]나 [[보상판단-payment-존재-lock-대신-db-unique]]처럼 "락 대신 unique로 동시성을 막자"던 결제·주문 쪽 선택과 대비된다 — 그쪽은 유일성 보장이 목적이라 락을 피했지만, 재고 차감은 **예외 구분**이 목적이라 오히려 락을 택했다. 목적이 다르면 결론이 갈린다.

## 사실관계 정정 — 보상 트랜잭션 오해 교정

초기에 "atomic UPDATE는 실패 시 보상 트랜잭션이 필요하다"고 생각했으나 틀렸다. RDB 트랜잭션 안의 UPDATE는 다른 SQL과 똑같이 트랜잭션이 롤백되면 함께 되돌아간다. 보상 트랜잭션이 진짜 필요한 경우는 "트랜잭션 경계 밖 외부 시스템 호출"이나 "트랜잭션 없이 즉시 반영되는 작업(Redis INCR 등)"이다. 따라서 비관적 락이 atomic UPDATE보다 나은 진짜 이유는 *원자성/롤백*이 아니라 앞서 말한 *도메인 예외 정밀 구분*이다.

## productId 정렬 — batch decrease 기각

한 주문에 여러 상품이 있을 때 `OrderCreateProcessor.execute()`는 items를 **`productId` 오름차순으로 정렬**한 뒤 stock을 단건씩 차감한다.

**왜 정렬인가:** 같은 두 상품을 서로 다른 순서로 차감하면 두 트랜잭션이 `SELECT ... FOR UPDATE`로 서로의 락을 잡아 상호 대기 데드락이 난다. 모든 트랜잭션이 동일 순서(productId 오름차순)로 락을 잡으면 이 데드락을 회피한다(전형적 lock ordering).

**왜 batch decrease(IN 절)가 아니라 단건 정렬인가 — 측정 기반:**

1. **InnoDB IN 절은 atomic lock acquisition이 아니다.** 테스트로 확인한 사실 — `SELECT ... FOR UPDATE`의 IN 절도 모든 락을 한 번에 잡는 게 아니라 옵티마이저가 인덱스를 보고 정한 순서로 순차적으로 row lock을 획득한다. 즉 batch도 단건 정렬과 사실상 같은 lock acquisition 패턴이라, batch의 이점은 네트워크 round-trip(N→1)·SQL parsing 절감뿐으로 좁아진다. **락 보유 시간 우위는 없다.**
2. **batch의 락 순서는 옵티마이저·인덱스 구조에 의존 → 잠재적 데드락.** IN 절의 락 획득 순서는 우리가 적은 순서가 아니라 옵티마이저가 정한다. 인덱스 변경·MySQL 업그레이드·통계 변화로 의도한 락 순서가 깨지면 데드락이 날 수 있다. IN 절 정렬을 명시할 수도 있으나 native query 문자열 안에 정렬 로직을 박아야 해, 컴파일러·IDE refactoring으로 잡히지 않는 hidden coupling이 된다.
3. **단건 정렬의 진짜 우위 — 추상화 깊이와 안정성.** 정렬이 Java 코드(`List.sort()`)라 컴파일러 검증·IDE refactoring이 되고, DB 옵티마이저 동작과 무관하며, 각 차감 사이에 도메인 로직(단위 검증·예외 분기)을 끼울 수 있고, 상품별 부족 사유를 에러 메시지에 명시할 수 있다.

`OrderConcurrencyService.createOrderWithPessimisticLockBatch`는 비교 실험용으로만 남기고 production은 단건 정렬을 채택했다(격리 근거는 [[order-concurrency-service-학습코드-격리]]).

## 트레이드오프 — 락 대기·DB 부담, Redis 전환 임계

- 경쟁이 심해지면 락 대기와 DB 부담이 커진다. 임계에 닿으면 Redis 분산 락으로 전환할 예정이나, 그 임계가 언제인지는 아직 미측정 — **락 경쟁 모니터링이 열린 과제**다([[stock-도메인-구조-개요]]).
- 이 비관락 전략은 order의 `@Version` 낙관락과 공존한다. 만료 배치는 FOR UPDATE를 안 잡고 낙관락에 의존하고, 결제 흐름은 비관락을 잡으며, 둘이 만나는 race를 `@Version`이 잡는다 — 자세한 공존 근거는 [[order-version-낙관락-비관락-공존]].

## 근거

- [[raw/sessions/backend/2026-05-29-stock-domain-overview]]
- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
