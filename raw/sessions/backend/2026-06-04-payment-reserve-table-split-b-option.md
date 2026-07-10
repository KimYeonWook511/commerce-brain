---
platform: backend
author: KimYeonWook511
created: 2026-06-04
---

> RESERVE 저장 위치를 A안(Payment 단일 테이블 type=RESERVE)에서 B안(별도 PaymentReservation 테이블)으로 전환한 결정 기록.
> 같은 날짜 [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] 의 "RESERVE 저장 위치 A/B 미정" 열린 항목을 닫는 후속 결정이다.
> 동시성 근거 [[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]] 는 B안에서도 그대로 유효.

# RESERVE 테이블 분리 — A안에서 B안으로

## 결정

`docs/tasks/payment-order-redesign/` 의 task 문서를 작성하며 A안(`tbl_payment` 단일 테이블에 RESERVE/APPROVE/CANCEL 모두 담기)으로 진행 중이었다. 검토하면서 반복적으로 위화감이 나왔고, 그 원인이 모두 "RESERVE 를 같은 테이블에 둔 것"으로 수렴해 B안(RESERVE 를 별도 `tbl_payment_reservation` 테이블로 분리, `tbl_payment` 는 APPROVE/CANCEL 만)으로 전환한다.

코드 수정 전, task 문서부터 B안 기준으로 다시 정리한 뒤 합의하기로 함.

---

## 왜 바꿨는가 — 위화감 4가지

1. **RESERVE 와 APPROVE/CANCEL 은 본질이 다르다.**
   - APPROVE/CANCEL: "PG 에 실제로 보낸 요청 사건". 결과 확정 후 행은 불변. 영구.
   - RESERVE: "결제창을 띄우기 위한 준비물". PG 요청 사건 아님. 내용은 불변이지만 **상태(RESERVED → USED/EXPIRED)는 계속 변한다.** 임시·만료·따닥/이탈로 대량 생성.
   - 성격이 다른 둘을 한 테이블에 두니, RESERVE 의 성격이 APPROVE/CANCEL 테이블을 오염시킨다.

2. **A안은 `pg_payment_id` 를 NULL 허용으로 완화시킨다.**
   - RESERVE 단계엔 pgPaymentId 가 없어서. 하지만 APPROVE/CANCEL 만 보면 항상 존재해야 하는 값. RESERVE 때문에 not-null 보장을 포기하는 건 손해.

3. **A안은 `status` 컬럼에 두 가지 의미를 섞는다.**
   - RESERVE 행의 `RESERVED/EXPIRED` 와 APPROVE/CANCEL 행의 `REQUESTED/SUCCEEDED/FAILED/UNKNOWN` 이 한 컬럼에 공존.
   - "결제 시도 몇 번?" 조회마다 `type != 'RESERVE'` 필터를 달아야 함.

4. **"테이블 하나"의 단순함은 착시.**
   - 테이블 수는 줄지만 NULL 컬럼(expires_at, reserved_key) 늘고, status 의미 혼재, 조회 분기 발생. 테이블 내부 복잡도 + 쿼리 복잡도가 올라간다.

→ RESERVE 의 "상태 변화 + 임시 + 만료 + 대량" 성격을 Reservation 테이블이 자기 책임으로 가져가면, `tbl_payment` 는 "PG 사건"만 담는 순수 append-only 가 되고 `pg_payment_id` 도 NOT NULL 복원.

---

## 바뀌는 것 vs 유지되는 것

**바뀌는 것 — RESERVE 거주지 하나**
- `tbl_payment_reservation` 신설. RESERVE 가 그쪽으로 이주.
- `tbl_payment` 에서 RESERVE 관련 제거 (type=RESERVE, status=RESERVED/EXPIRED, expires_at, reserved_key).
- `tbl_payment.pg_payment_id` NOT NULL 복원.
- `uk_reserved` 의 거주지: `tbl_payment` → `tbl_payment_reservation` 으로 이동. NULL 트릭 자체는 동일.
- 흐름: redirect 단계에서 RESERVE 행을 "전이"시키는 게 아니라, **Reservation 을 읽어 새 APPROVE 행을 생성** + Reservation 을 USED 로 마킹.

**유지되는 것 — 나머지 결정 전부**
- 시도 단위 append-only (사건은 새 행, 상태 전이 덮어쓰기 금지)
- merchantPayKey 서버 발급, Order 에서 제거
- 완료 판단 = EXISTS(성공 APPROVE ∧ 무효화 CANCEL 부재)
- 이중결제 최종 방어선 `uk_approved` NULL 트릭 (MySQL InnoDB partial unique 미지원)
- UNKNOWN 마킹까지만 이번 task, 대사는 후속
- 외부 PG 호출 트랜잭션 밖, DB 쓰기 한 트랜잭션 안
- 존재 보장=unique, 계산 판단=PK 단일 행 FOR UPDATE, gap lock 회피 ([[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]])
- MySQL 8 InnoDB, 운영 데이터 없음 → backfill 없는 schema 변경

---

## B안 목표 구조

### 테이블 두 개

```
PaymentReservation (tbl_payment_reservation)  — 결제창 준비물 (임시·만료·상태 변화)
 ├─ id                PK
 ├─ order_id          BIGINT NOT NULL   소속 Order PK. FK 없음
 ├─ provider          VARCHAR(32) NOT NULL
 ├─ merchant_pay_key  VARCHAR(64) NOT NULL   서버 발급. redirect 역조회. UNIQUE
 ├─ amount            INT NOT NULL   결제 예정 금액 (승인 시 PG 응답 대조)
 ├─ status            VARCHAR(32) NOT NULL   RESERVED / USED / EXPIRED
 ├─ expires_at        DATETIME(6) NOT NULL
 ├─ reserved_key      VARCHAR(96) NULL   RESERVED 일 때만 "{order_id}:{provider}", 그 외 NULL
 ├─ created_at / updated_at
 └─ 인덱스: uk_reservation_mpk(merchant_pay_key) UNIQUE,
            uk_reserved(reserved_key) UNIQUE   ← reserve 따닥 차단이 이 테이블로

Payment (tbl_payment)  — PG 에 보낸 실제 요청 사건 (영구·append-only)
 ├─ id                PK
 ├─ order_id          BIGINT NOT NULL
 ├─ provider          VARCHAR(32) NOT NULL
 ├─ merchant_pay_key  VARCHAR(64) NOT NULL   어느 reservation 에서 비롯됐는지 (값 연결)
 ├─ pg_payment_id     VARCHAR(64) NOT NULL   ← NOT NULL 복원
 ├─ type              VARCHAR(32) NOT NULL   APPROVE / CANCEL   ← RESERVE 제거
 ├─ status            VARCHAR(32) NOT NULL   REQUESTED / SUCCEEDED / FAILED / UNKNOWN
 ├─ amount            INT NOT NULL
 ├─ fail_code         VARCHAR(32) NULL
 ├─ fail_detail       VARCHAR(255) NULL
 ├─ approved_order_key BIGINT NULL   APPROVE+SUCCEEDED 일 때만 order_id, 그 외 NULL
 ├─ responded_at      DATETIME(6) NULL
 ├─ created_at / updated_at
 └─ 인덱스: uk_approved(approved_order_key) UNIQUE,
            uk_payment_merchant_pay_key_provider_pg_payment_id_type
```

### 두 테이블 관계
- 연결 고리는 `merchant_pay_key` 값. FK 제약 없음.
- `Reservation 1 : N Payment` (한 예약에서 승인 재시도 + 취소까지).

### 흐름 (A안 대비 달라지는 핵심)

```
ready    → PaymentReservation(RESERVED) 생성/재사용/만료
           재사용: status=RESERVED ∧ expires_at>now ∧ provider 일치
           동시 따닥은 uk_reserved (reservation 테이블) 가 차단
redirect → merchant_pay_key 로 Reservation 조회 → order_id 확보
           → Payment(type=APPROVE, status=REQUESTED) "새 행" 생성  ← RESERVE 행 전이 아님
           → [트랜잭션 밖] NaverPay 승인 API
           → [트랜잭션 안] 성공: Payment.succeed(approved_order_key=order_id 같은 UPDATE)
                          + Order PAID + Reservation 을 USED 로 마킹
           → uk_approved 위반: 보상 CANCEL
취소     → Payment(type=CANCEL) "새 행" 생성 → 네이버 취소 → SUCCEEDED (후속 task)
```

---

## task 문서 재정리 범위

`docs/tasks/payment-order-redesign/` 의 다음을 B안 기준으로 다시 정리할 예정.

- **db-schema**: `tbl_payment_reservation` 신규. `tbl_payment` 에서 RESERVE 흔적 제거 + `pg_payment_id` NOT NULL 복원. `uk_reserved` 이동. Flyway 마이그레이션 재작성.
- **adr**: ADR-1(시도 단위)에서 RESERVE 를 Payment.type 에서 제외, ADR-5(ready 흐름)를 Reservation 기반으로, ADR-3(NULL 트릭)에서 `uk_reserved` 위치 변경. **A→B 전환 ADR 1건 신설** (위 위화감 4가지가 근거).
- **architecture**: 두 테이블 기준 도메인/엔티티/흐름. `PaymentReservation` 엔티티 + 도메인 메서드(createReserved/reuse판단/expire/markUsed). `Payment` 는 APPROVE/CANCEL.
- **prd**: ready/approve 흐름을 Reservation 분리 기준으로.
- **api-spec**: 엔드포인트 시그니처 동일(호환 유지). 내부 동작 설명만 갱신. PAYMENT_RESULT_PENDING 유지.
- **step1~5**: B안에 맞게 재구성. 특히 step1(엔티티/스키마)·step2(reserve 재사용/만료가 이제 Reservation 대상)·step3(승인 안전망에서 Reservation 읽고 APPROVE 새 행 생성).

---

## 열린 판단 — Claude 가 제안할 것

- Reservation 엔티티/패키지 위치, status enum 이름 (`RESERVED/USED/EXPIRED` 가 적정한지), USED 상태를 둘지 말지 등 세부는 task 문서 일관성 보고 조정.
- A안 문서에서 B안에선 모순·불필요해지는 서술(예: "RESERVE 단계는 pg_payment_id NULL 이라 자연스레 unique 빠짐") 식별·정리.
- 한꺼번에 고치지 말고 diff 개요부터 보여주고 사용자 확인 후 진행.

---

## 한 줄 요약

> RESERVE 와 APPROVE/CANCEL 은 "상태 변하는 임시 준비물"과 "불변 PG 요청 사건"으로 본질이 달라, 같은 테이블에 두면 `pg_payment_id` NOT NULL · status 의미 단일성 · 쿼리 단순성이 모두 깎인다. 그래서 RESERVE 를 별도 `tbl_payment_reservation` 으로 분리(B안)하고, `tbl_payment` 는 APPROVE/CANCEL append-only 사건 테이블로 정리한다. 나머지 결정(NULL 트릭 unique, EXISTS 완료 판단, unique vs lock 전략, append-only)은 그대로 유효 — 거주지 하나만 바뀐다.
