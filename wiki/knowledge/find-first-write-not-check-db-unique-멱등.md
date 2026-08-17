---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [find-first, db-unique, idempotency, concurrency, exception-handling, convention]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
  - "[[raw/sessions/backend/2026-06-08-payment-idempotency-write-not-check]]"
---

# find-first 패턴 — 사전 find로 정상 멱등 흡수, unique 위반 race만 안전망 500

## 한 줄 정의

payment·order 공통으로 정착한 멱등 정책. **멱등은 "확인 후 생성"이 아니라 "INSERT 시도 + unique 위반 시 기존 반환"이다.** 정상 멱등 재요청은 사전 `find`로 값싸게 흡수하고, 진짜 보장은 DB unique 제약이 한다. Application/Adapter 어디서도 `DuplicateKeyException`을 catch하지 않고, unique 위반 race window 충돌만 `GlobalExceptionHandler`의 `DataAccessException` 부모 핸들러가 500으로 처리한다.

## 동작(사전 find 흡수 / unique 위반 안전망 500)

**왜 "확인 후 생성"이 뚫리는가 (MVCC 스냅샷):**

```
T1: findByKey → 없음
T2: findByKey → 없음   (T1 미커밋 + REPEATABLE READ 스냅샷이라 T1의 INSERT 안 보임)
T1: save 성공 / T2: save 성공  → 둘 다 "없음"을 봐서 둘 다 생성
```

SELECT는 MVCC 스냅샷이라 서로의 미커밋 INSERT를 못 본다. 반면 **write(INSERT)는 스냅샷이 아니라 현재 커밋 상태와 충돌**하므로 unique 제약이 "이미 있음"을 정확히 잡는다. 멱등의 진짜 보장은 여기에 있다.

정석 `getOrCreate`:

```java
public Payment getOrCreate(String key, ...) {
    return repo.findByKey(key)                 // 1. 흔한 경우(이미 있음) 빠르게 흡수
        .orElseGet(() -> insertOrReread(key)); // 2. 없으면 생성, 3. race면 안전망
}
```

`find`는 흔한 시간차 중복을 값싸게 흡수하는 **1차 필터**일 뿐, 최종 보장은 unique다.

## 적용 조건 두 가지

find-first는 **조건부 패턴**이다. 다음 둘이 만족될 때만 race window 비용이 안전망 500으로 값싸게 흡수된다:

1. **트랜잭션이 짧다** — race window가 좁다.
2. **정상 흐름의 동시 충돌 확률이 낮다** — 사용자 입력 식별자 / idempotency key 기반 unique라 같은 순간 겹치는 동시 race가 극히 드물다.

## 조건이 깨지면(try-save-catch)

이 조건이 깨지면(충돌이 잦은 곳, 예: 캐시 미스 후 동시 다발 insert) try-save-catch 패턴이 더 적합하다.

또한 **위반난 같은 트랜잭션에서 재조회하지 않는다.** (a) MVCC — 위반은 났어도 같은 트랜잭션 SELECT는 시작 시점 스냅샷이라 그 행을 여전히 못 볼 수 있다. (b) 세션 오염 — flush 중 예외가 나면 트랜잭션이 rollback-only로 마킹되고 Hibernate 세션은 "더 쓰지 마라"가 기본. 재조회로 수습하려면 새 트랜잭션(새 스냅샷)이라야 하고, 그 곡예가 싫으면 위반을 그냥 던진다.

**위반의 의미로 응답이 갈린다:** 노이즈성 위반(예약 따닥, 돈 안 걸림)은 그냥 500 — 재시도하면 find가 흡수([[payment-이중결제-reserve따닥-mysql-null트릭-unique]]). 돈 걸린 멱등(승인)은 사용자에게 500을 주면 재시도 위험이 있어, 진입 사전 체크로 시간차 중복을 멱등 성공으로 흡수하고 드문 동시 race만 보상·별도 트랜잭션으로 격리한다.

## 적용처(payment 3곳 + order 멱등)

- **payment** — find-first 6곳 중 3곳 차지: `PaymentApprovalService`·`PaymentApprovalAttemptService`·`PaymentCancellationAttemptService`. unique 위반 race는 `GlobalExceptionHandler`의 `DataAccessException` 부모 핸들러가 500(`COMMON-500-2`)으로. [[payment-amount-mismatch-이중검증-409-vs-400-분리]]의 catch 검증이 이 패턴의 인스턴스.
- **order** — 주문 생성 멱등키 `(member_id, idempotency_key)` unique 위반 fallback 재조회. 초기 `DuplicateKeyException` catch 분기를 폐기하고 이 정책으로 통일([[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]).

## 관련 링크

- [[보상판단-payment-존재-lock-대신-db-unique]] — "lock 대신 DB unique"라는 같은 결의 데이터 모델 동시성. 이 패턴의 상위 통일 원칙.
- [[payment-도메인-구조-개요]] — payment 도메인 통일 원칙의 진입점.
- [[persistence-exception-노출-경계-추상수준]]·[[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]] — 안전망 500의 예외 노출 경계·번역 지점.
- [[조회-또는-생성-메서드-해체-조회·생성·기존요청대조-셋]] — 위 `getOrCreate` 형태를 한 메서드로 묶지 않기로 한 후속. **묶어서 얻는 원자성이 없다**는 것이 이 노트의 "최종 보장은 unique"와 같은 사실의 다른 면이다.
- [[멱등키-세-값-분리와-요청멱등키는-호출자가-발급]] — 이 패턴이 성립하는 전제("정상 경로는 조회가 먼저 걸러 준다")가 깨지는 조건. **유일 제약의 범위가 조회 범위보다 넓으면 안전망이 정상 흐름에서 터진다.**
- [[동시도착은-선점층이-받고-처리중-전용응답]] — 아래 열린 질문("돈 걸린 곳의 동시 race에서 멱등 성공 응답을 만들지")에 대한 답. 성공이 아니라 "처리 중"으로 답한다.

## 열린 질문

- 돈 걸린 곳의 동시 race(2차)에서 멱등 성공 응답을 *만들지* 여부는 정책 결정으로 남았다. 안 만들고 재시도에 넘기는 게 단순하고, 만들려면 새 트랜잭션 재조회가 필요하다.
- 보상(PG 환불)이 또 실패하는 경우 — 배치 재처리 후보(별도 논의).

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
- [[raw/sessions/backend/2026-06-08-payment-idempotency-write-not-check]]
