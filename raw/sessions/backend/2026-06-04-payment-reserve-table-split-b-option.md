---
platform: backend
author: KimYeonWook511
created: 2026-06-04
---

# RESERVE 거주지 분리 — 단일 결제 테이블(A안)에서 별도 예약 테이블(B안)로 전환

결제-주문 재설계를 진행하며, 결제 데이터를 단일 `tbl_payment` 한 테이블에 `type ∈ {RESERVE, APPROVE, CANCEL}` 로 모두 담는 A안으로 task 문서를 쓰고 있었다. 쓰는 내내 반복적으로 위화감이 나왔고 그 원인이 전부 "RESERVE 를 같은 테이블에 둔 것"으로 수렴했다. 그래서 RESERVE 를 별도 `tbl_payment_reservation` 테이블로 떼어내고 `tbl_payment` 는 APPROVE/CANCEL 만 담는 구조(B안)로 전환하기로 한 세션이다. 같은 날 결제-주문 재설계 논의에서 "RESERVE 저장 위치를 A/B 중 어디로 할지 미정"으로 남겨둔 항목을 여기서 닫는다. 함께 정한 동시성 전략(존재 보장은 unique 제약, 여러 행을 합산해 계산하는 판단은 단일 행 FOR UPDATE, gap lock 회피)은 거주지가 바뀌어도 그대로 유효하다.

## 결정한 것

RESERVE 를 `tbl_payment` 에서 떼어 새 `tbl_payment_reservation` 테이블로 옮기고, `tbl_payment` 는 "PG 에 실제로 보낸 요청 사건"(APPROVE/CANCEL)만 담는 순수 append-only 테이블로 정리한다. 코드에 손대기 전에 재설계 task 의 문서 세트부터 B안 기준으로 다시 쓰고 합의한 뒤 구현으로 넘어가기로 했다.

### 왜 바꿨나 — 위화감 4가지

1. **RESERVE 와 APPROVE/CANCEL 은 본질이 다르다.**
   - APPROVE/CANCEL: "PG 에 실제로 보낸 요청 사건". 결과가 확정되면 그 행은 불변, 영구 보존.
   - RESERVE: "결제창을 띄우기 위한 준비물". PG 요청 사건이 아니다. 내용 자체는 불변이지만 **상태(RESERVED → USED/EXPIRED)는 계속 변하고**, 임시·만료성이며 결제창 따닥/이탈로 대량 생성된다.
   - 성격이 다른 둘을 한 테이블에 두면 RESERVE 의 성격이 APPROVE/CANCEL 테이블을 오염시킨다.

2. **A안은 `pg_payment_id` 를 NULL 허용으로 완화시킨다.**
   - RESERVE 단계엔 pgPaymentId 가 아직 없기 때문. 하지만 APPROVE/CANCEL 만 놓고 보면 항상 존재해야 하는 값이다. RESERVE 때문에 NOT NULL 보장을 포기하는 건 손해.

3. **A안은 `status` 컬럼에 두 가지 의미를 섞는다.**
   - RESERVE 행의 `RESERVED/EXPIRED` 와 APPROVE/CANCEL 행의 `REQUESTED/SUCCEEDED/FAILED/UNKNOWN` 이 한 컬럼에 공존한다.
   - "결제 시도 몇 번?" 같은 조회마다 `type != 'RESERVE'` 필터를 달아야 한다.

4. **"테이블 하나"의 단순함은 착시다.**
   - 테이블 수는 줄지만 NULL 컬럼(`expires_at`, `reserved_key`)이 늘고, status 의미가 혼재하고, 조회 분기가 생긴다. 테이블 내부 복잡도와 쿼리 복잡도가 오히려 올라간다.

→ RESERVE 의 "상태 변화 + 임시 + 만료 + 대량" 성격을 예약 테이블이 자기 책임으로 가져가면, `tbl_payment` 는 "PG 사건"만 담는 순수 append-only 가 되고 `pg_payment_id` 도 NOT NULL 로 복원된다. 트레이드오프는 테이블 수 +1 뿐이고, 의미·스키마 단순화 + 쿼리 명확성 + NOT NULL 회복의 이득이 그보다 크다.

### 바뀌는 것 vs 유지되는 것

**바뀌는 것 — RESERVE 의 거주지 하나**
- `tbl_payment_reservation` 신설. RESERVE 가 그쪽으로 이주.
- `tbl_payment` 에서 RESERVE 관련 제거(type=RESERVE, status=RESERVED/EXPIRED, expires_at, reserved_key).
- `tbl_payment.pg_payment_id` NOT NULL 복원.
- reserve 따닥을 막는 unique 제약(작업명 `uk_reserved`)의 거주지: `tbl_payment` → `tbl_payment_reservation` 으로 이동. "조건 맞을 때만 값, 아니면 NULL" 트릭 자체는 동일.
- 흐름: redirect 단계에서 RESERVE 행을 APPROVE 로 "전이"시키는 게 아니라, **예약 행을 읽어 새 APPROVE 행을 생성**하고 예약을 USED 로 마킹한다.

**유지되는 것 — 나머지 결정 전부**
- 시도 단위 append-only(사건은 새 행, 상태 전이를 덮어쓰기 금지).
- merchantPayKey 를 서버가 발급하고 Order 에서 제거.
- 결제 완료 판단 = EXISTS(성공 APPROVE 존재 ∧ 이를 무효화한 성공 CANCEL 부재). 마지막 행 기반 판단 금지.
- 이중결제 최종 방어선: 주문당 성공 APPROVE 1개를 강제하는 unique(작업명 `uk_approved`)의 NULL 트릭. MySQL InnoDB 가 partial unique index 를 미지원하기에 택한 우회.
- UNKNOWN 은 마킹까지만 이번 task 에 포함, 대사(해소)는 후속 task.
- 외부 PG 호출은 트랜잭션 밖, DB 쓰기는 한 트랜잭션 안.
- 존재 보장은 unique, 계산 판단은 PK 단일 행 FOR UPDATE, gap lock 회피.
- MySQL 8 InnoDB, 운영 데이터 없음 전제 → backfill 없는 단순 schema 변경.

### B안 목표 구조

두 테이블로 나뉜다. `PaymentReservation` 은 결제창 준비물(임시·만료·상태 변화), `Payment` 는 PG 에 보낸 실제 요청 사건(영구·append-only)이다.

```
PaymentReservation (tbl_payment_reservation)  — 결제창 준비물
 ├─ id                PK
 ├─ order_id          BIGINT NOT NULL   소속 Order PK. FK 없음
 ├─ provider          VARCHAR(32) NOT NULL
 ├─ merchant_pay_key  VARCHAR(64) NOT NULL   서버 발급. redirect 역조회. UNIQUE
 ├─ amount            INT NOT NULL   결제 예정 금액 (승인 시 PG 응답 대조)
 ├─ status            VARCHAR(32) NOT NULL   RESERVED / USED / EXPIRED
 ├─ expires_at        DATETIME(6) NOT NULL
 ├─ reserved_key      VARCHAR(96) NULL   RESERVED 일 때만 "{order_id}:{provider}", 그 외 NULL
 ├─ created_at / updated_at
 └─ 인덱스: merchant_pay_key UNIQUE,
            reserved_key UNIQUE (작업명 uk_reserved)   ← reserve 따닥 차단이 이 테이블로

Payment (tbl_payment)  — PG 에 보낸 실제 요청 사건 (영구·append-only)
 ├─ id                PK
 ├─ order_id          BIGINT NOT NULL
 ├─ provider          VARCHAR(32) NOT NULL
 ├─ merchant_pay_key  VARCHAR(64) NOT NULL   어느 예약에서 비롯됐는지 (값 연결)
 ├─ pg_payment_id     VARCHAR(64) NOT NULL   ← NOT NULL 복원
 ├─ type              VARCHAR(32) NOT NULL   APPROVE / CANCEL   ← RESERVE 제거
 ├─ status            VARCHAR(32) NOT NULL   REQUESTED / SUCCEEDED / FAILED / UNKNOWN
 ├─ amount            INT NOT NULL
 ├─ fail_code         VARCHAR(32) NULL
 ├─ fail_detail       VARCHAR(255) NULL
 ├─ approved_order_key BIGINT NULL   APPROVE+SUCCEEDED 일 때만 order_id, 그 외 NULL
 ├─ responded_at      DATETIME(6) NULL
 ├─ created_at / updated_at
 └─ 인덱스: approved_order_key UNIQUE (작업명 uk_approved),
            (merchant_pay_key, provider, pg_payment_id, type) UNIQUE
```

**두 테이블 관계**
- 연결 고리는 `merchant_pay_key` **값**. FK 제약은 두지 않는다.
- `예약 1 : N Payment`(한 예약에서 승인 재시도 + 취소까지 여러 사건).

**흐름 (A안 대비 달라지는 핵심)**

```
ready    → PaymentReservation(RESERVED) 생성/재사용/만료
           재사용 조건: status=RESERVED ∧ expires_at>now ∧ provider 일치
           동시 따닥은 예약 테이블의 uk_reserved 가 차단
redirect → merchant_pay_key 로 예약 조회 → order_id 확보
           → Payment(type=APPROVE, status=REQUESTED) "새 행" 생성  ← RESERVE 행 전이 아님
           → [트랜잭션 밖] NaverPay 승인 API
           → [트랜잭션 안] 성공: Payment.succeed(approved_order_key=order_id 같은 UPDATE)
                          + Order PAID + 예약을 USED 로 마킹
           → uk_approved 위반: 보상 CANCEL
취소     → Payment(type=CANCEL) "새 행" 생성 → 네이버 취소 → SUCCEEDED (후속 task)
```

### 재설계 task 문서에 반영할 것

코드보다 문서를 먼저 B안으로 맞춘다. 성격상 기존 A안 결정 문서 몇 개는 본문을 고쳐야 하고, A안→B안 전환 자체는 별도 결정 문서로 신설한다.

- **DB 스키마 문서**: `tbl_payment_reservation` 신규. `tbl_payment` 에서 RESERVE 흔적 제거 + `pg_payment_id` NOT NULL 복원. reserve 따닥 방어 unique 를 예약 테이블로 이동. Flyway 마이그레이션 재작성.
- **결정 문서**: 결제를 시도 단위 append-only 로 정의한 결정에서 Payment 의 type 목록에서 RESERVE 를 뺀다. ready 흐름 결정을 예약 행 생성/재사용 기반으로 다시 쓴다. NULL 트릭 unique 결정에서 reserve 따닥 방어 unique 의 거주 테이블을 예약 테이블로 옮긴다. 그리고 위 위화감 4가지를 근거로 A안→B안 전환 결정을 새로 하나 신설한다.
- **아키텍처 문서**: 두 테이블 기준으로 도메인/엔티티/흐름 재작성. `PaymentReservation` 엔티티와 도메인 메서드(createReserved / 재사용 판단 / expire / markUsed) 도입. `Payment` 는 APPROVE/CANCEL 로 한정.
- **PRD**: ready/approve 흐름을 예약 분리 기준으로 갱신.
- **API 스펙**: 엔드포인트 시그니처는 동일하게 유지(외부 호환). 내부 동작 설명만 갱신. UNKNOWN 차단 시 주는 "결제 결과 확인 중" 응답 코드(`PAYMENT_RESULT_PENDING`)는 그대로 유지.
- **단계별 구현 문서(step1~5)**: B안에 맞게 재구성. 특히 엔티티/스키마 단계, reserve 재사용/만료가 이제 예약 행 대상이 되는 단계, 승인 안전망에서 예약을 읽어 APPROVE 새 행을 생성하는 단계.

## 미해결·열린 질문

- 예약 엔티티/패키지 위치, status enum 이름이 `RESERVED/USED/EXPIRED` 로 적정한지, USED 상태를 실제로 둘지 말지 등 세부는 task 문서 전체 일관성을 보고 조정한다.
- A안 문서에 있던 서술 중 B안에선 모순·불필요해지는 것(예: "RESERVE 단계는 pg_payment_id 가 NULL 이라 자연스레 unique 에서 빠진다")을 식별해 정리한다.

## 한 줄 요약

RESERVE 와 APPROVE/CANCEL 은 "상태 변하는 임시 준비물"과 "불변 PG 요청 사건"으로 본질이 달라, 같은 테이블에 두면 `pg_payment_id` NOT NULL · status 의미 단일성 · 쿼리 단순성이 모두 깎인다. 그래서 RESERVE 를 별도 `tbl_payment_reservation` 으로 분리(B안)하고 `tbl_payment` 는 APPROVE/CANCEL append-only 사건 테이블로 정리한다. 나머지 결정(NULL 트릭 unique, EXISTS 완료 판단, unique vs lock 전략, append-only)은 그대로 유효하다 — 거주지 하나만 바뀐다.
