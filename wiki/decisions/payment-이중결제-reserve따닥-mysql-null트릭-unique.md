---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, idempotency, mysql, innodb, unique-constraint, double-payment, reservation, concurrency]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]]"
  - "[[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]]"
  - "[[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]"
  - "[[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]]"
---

# 이중결제·reserve 따닥 멱등 — MySQL NULL 트릭 partial unique

결제 재설계([[payment-order-도메인분리와-pg격리]])에서 정합성의 최종 방어선을 DB 제약으로 세운 결정이다. 앱/Redis/UI의 1차 필터는 없어도 시스템이 돌아가야 하고, 정합성·중복의 최종 보장은 DB에 둔다("1차 필터 + 최종 방어선" 이중 구조).

## 불변식 — 성공승인 1개·(주문,수단) RESERVED 1개

DB로 강제해야 하는 두 불변식이다.

- **이중결제**: 한 주문에는 성공한 승인이 **전 PG를 통틀어 최대 1개**만 존재한다.
- **reserve 따닥**: (주문, 수단)당 진행 중 RESERVED는 최대 1개만 존재한다.

## PostgreSQL partial unique가 원안, MySQL 한계

가장 직관적인 원안은 조건부(partial) unique 인덱스다.

```sql
-- PostgreSQL이라면
CREATE UNIQUE INDEX uk_order_approved ON payment (order_id)
  WHERE type = 'APPROVE' AND status = 'SUCCEEDED';
```

성공한 APPROVE만 주문당 1개로 제한하고 실패한 APPROVE는 조건 밖이라 여럿 OK. 그러나 **MySQL InnoDB는 `WHERE` 절을 가진 partial(조건부) unique index를 지원하지 않는다.** `UNIQUE(order_id)`를 통째로 걸면 실패·취소 시도까지 막혀 안 된다(주문당 시도가 여러 개여야 하므로).

## NULL 트릭 우회(InnoDB NULL 중복 허용)

InnoDB unique index가 **NULL을 중복으로 치지 않는(여러 NULL 허용)** 성질을 이용한다 — "조건 만족 시에만 값, 아니면 NULL" 컬럼 + 일반 unique.

```sql
-- 이중결제: 성공 APPROVE일 때만 order_id, 그 외 NULL
ALTER TABLE payment ADD COLUMN approved_order_key BIGINT NULL;
CREATE UNIQUE INDEX uk_payment_approved_order_key ON payment (approved_order_key);

-- reserve 따닥: RESERVED일 때만 "orderId:provider", 그 외 NULL
ALTER TABLE payment_reservation ADD COLUMN reserved_key VARCHAR(96) NULL;
CREATE UNIQUE INDEX uk_payment_reservation_reserved_key ON payment_reservation (reserved_key);
```

NULL은 중복 허용이라 실패·취소·만료 행은 제약을 안 받고, 조건을 만족하는 행끼리만 유일성이 걸린다. 효과는 partial unique와 **동일**하며 동시 승인의 race condition도 차단된다. 실제 V6 마이그레이션과 예약 테이블에 그대로 반영됐다(검증). reserved_key의 거주지는 이후 예약 테이블로 이동했다 — [[payment-reserve-예약테이블-분리-a안-b안]].

> 왜 lock이 아니라 unique인지(트랜잭션 수명·gap lock 회피)는 별도 노트 [[payment-동시성-unique-vs-lock-gap-lock회피]]에서 상술한다. 계산 기반 판단(과다취소 검증 등)만 PK 단일 행 lock으로 좁게 건다.

## cross-provider 이중결제 vs (주문,수단) reserve — 방어 대상이 다름

두 unique는 컬럼 구성이 반대인데, 이는 **방어하려는 대상이 다르기 때문**이다.

- `approved_order_key`는 `order_id`만 넣고 **provider를 안 섞는다.** 사용자가 네이버·카카오에 동시 결제를 보내 둘 다 승인되는 *cross-provider 이중결제*를 막는 게 목적이다. merchantPayKey 멱등은 "한 PG 안에서 같은 키 중복"만 막을 뿐, "여러 PG에 걸친 같은 주문 중복"은 원리적으로 못 막아 멱등을 주문 단위로 끌어올려야 한다.
- `reserved_key`는 반대로 **`"{order_id}:{provider}"`로 provider를 포함**한다. (주문, 수단)당 RESERVED 1개를 보장해야 같은 수단 따닥만 재사용으로 흡수하고 다른 수단은 새 예약이 되기 때문이다.

## 캡슐화(같은 UPDATE) + saveAndFlush load-bearing

NULL 트릭은 "SQL 마법"이라, **상태 전이와 트릭 컬럼 set/NULL을 같은 도메인 메서드의 같은 UPDATE에서** 처리해야 정합성이 산다. 둘이 따로 변경되면(예: 상태만 바꾸고 트릭 컬럼은 안 비움) unique가 무력화된다.

- `Payment.succeed()` — `status=SUCCEEDED`와 `approved_order_key=orderId`를 함께.
- `PaymentReservation.use()`/`expire()`(옛 `markUsed`/`markExpired`) — `status`와 `reserved_key=NULL`을 함께.

도메인 테스트로 "한 메서드 호출에서 두 필드 동시 변경"을 단언으로 박았다. 이 메서드 rename 경계는 [[payment-attempt-네이밍-정리와-refactor-경계]] 참조.

**flush 타이밍이 load-bearing이다.** 승인 확정 오케스트레이션은 성공 전이 뒤 명시적 `save()`를 호출하는데, infra adapter의 `saveAndFlush`(즉시 flush) 덕에 제약 위반이 **트랜잭션 커밋 전, 승인 흐름의 try-catch 안에서** `DataIntegrityViolationException`으로 확정된다. 이걸 같은 트랜잭션 catch에서 잡아 이중결제 보상취소를 트리거한다. flush를 커밋 시점으로 미루면 예외가 try-catch 밖에서 터져 보상 catch를 빠져나가므로, 이 명시 save는 제거 금지다. unique 위반의 예외 번역 경계는 [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]].

## 중복 콜백 멱등 반환 + 보상취소 사슬

- **중복 콜백**: 에러가 아니라 **기존 결과를 반환**한다(진짜 멱등). 앱 레벨 체크만 믿지 말고 DB unique를 최종 방어선으로 둔다.
- **보상취소**: unique가 막는 두 번째 승인은 이미 PG에서 돈이 빠졌으므로, 위반 감지 시 그 승인을 즉시 CANCEL(환불)한다(CANCEL 행이 보상취소 기록). 방어선 사슬은 **(1) 진행 중 결제 락(1차, 불완전) → (2) unique(최종 방어선) → (3) 보상 취소**다. 보상 판단을 lock이 아니라 payment 존재로 하는 후속 정리는 [[payment-order-facade-결합끊기-tell-dont-ask]]와 이어진다.

전액취소 CANCEL의 멱등은 별개다 — 기존 4컬럼 unique `(merchant_pay_key, provider, pg_payment_id, type)`가 `type`에 CANCEL을 포함해 "한 결제당 CANCEL 하나"를 이미 하드 보장한다. 그 발견과 부분취소에서의 한계는 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]·[[payment-부분취소-모델만-열고-구현-보류]]에 상세하다.

> [!note] 진화 (2026-08) — 두 unique가 활성 슬롯 하나로 합쳐졌다
> 예약 테이블이 폐지되면서 `reserved_key`와 `approved_order_key` 두 방어가 **결제 행의 활성 슬롯 하나**로 모였다([[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]). NULL 트릭 자체는 그대로 쓰이고, **비웠다가 같은 값으로 다시 잡는 흐름**이 새로 생겨 쓰기 순서 문제가 따라왔다([[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]) — 그 노트에 이 트릭의 DBMS 이식 한계(SQL Server·PostgreSQL 15+)와 조건부 유일을 생성 컬럼으로 표현하는 미검토 대안이 정리돼 있다.
> **주의 — 같은 NULL 성질이 반대로 함정이 되는 자리가 있다.** 제약으로 지켜야 할 값을 비우면 그 행이 제약의 보호 밖으로 나간다([[멱등키-세-값-분리와-요청멱등키는-호출자가-발급]]).
> 종류마다 다른 컬럼을 쓰는 조합 제약 대신 **문자열 한 컬럼**을 쓰는 것도 같은 NULL 성질에서 나온 판단이다([[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]]).

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 불변식, partial unique 원안·MySQL 한계, NULL 트릭, 방어선 사슬, 중복 콜백 멱등.
- [[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]] — NULL 트릭이 실제 마이그레이션에 반영됐다는 검증.
- [[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]] — reserved_key 거주지의 예약 테이블 이동.
- [[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]] — cross-provider 차단 의도, 캡슐화·markUsed/markExpired 동시 set 단언.
- [[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]] — 명시 save/saveAndFlush load-bearing과 flush 타이밍·보상 catch.
