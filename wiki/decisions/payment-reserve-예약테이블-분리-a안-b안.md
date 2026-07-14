---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, reservation, schema-design, append-only, not-null, mysql]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]]"
  - "[[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]"
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
---

# RESERVE 거주지 분리 — 단일 결제 테이블(A안)에서 예약/사건 두 테이블(B안)로

## A안 배경 — 단일 테이블 시도

결제-주문 재설계 초기엔 `tbl_payment` 한 테이블에 `type ∈ {RESERVE, APPROVE, CANCEL}`로 모든 결제 데이터를 담는 A안이었다. 같은 날 재설계 논의([[payment-order-도메인분리와-pg격리]])에서 "RESERVE 저장 위치를 A/B 중 어디로 할지"는 정합성 문제가 아니라 노이즈 정리 취향이라며 **미정**으로 열어뒀는데, task 문서를 A안으로 써 내려가는 내내 위화감이 반복돼 그 열린 질문을 여기서 B안으로 닫았다(supersede가 아니라 후속 확정).

## 위화감 4가지

1. **RESERVE와 APPROVE/CANCEL은 본질이 다르다.** APPROVE/CANCEL은 "PG에 실제로 보낸 요청 사건"으로 결과가 확정되면 불변·영구 보존이다. RESERVE는 "결제창을 띄우기 위한 준비물"로 PG 요청 사건이 아니며, 상태(RESERVED→USED/EXPIRED)가 계속 변하고 임시·만료성이며 결제창 따닥/이탈로 대량 생성된다. 한 테이블에 두면 RESERVE의 성격이 사건 테이블을 오염시킨다.
2. **A안은 `pg_payment_id` NOT NULL을 포기시킨다.** RESERVE 단계엔 pgPaymentId가 아직 없어 컬럼을 NULL 허용으로 완화해야 한다. APPROVE/CANCEL만 보면 항상 존재해야 하는 값인데 RESERVE 때문에 보장을 포기하는 손해.
3. **`status` 컬럼에 두 의미가 혼재한다.** 예약 상태(RESERVED/EXPIRED)와 사건 상태(REQUESTED/SUCCEEDED/FAILED/UNKNOWN)가 한 컬럼에 섞여, "결제 시도 몇 건?" 같은 조회마다 `type != 'RESERVE'` 필터를 달아야 한다.
4. **"단일 테이블이 단순"은 착시다.** 테이블 수는 줄지만 NULL 컬럼(`expires_at`, `reserved_key`)이 늘고 status 의미가 혼재하며 조회 분기가 생긴다. 테이블 내부 복잡도·쿼리 복잡도가 오히려 오른다.

## B안 결정과 두 테이블 목표 구조

RESERVE를 별도 `tbl_payment_reservation`(임시·만료·상태 변화)으로 떼고, `tbl_payment`는 "PG에 실제로 보낸 사건"(APPROVE/CANCEL)만 담는 순수 append-only로 정리한다. 그러면 `pg_payment_id NOT NULL`이 복원되고, `Payment.status`(REQUESTED/SUCCEEDED/FAILED/UNKNOWN)와 `PaymentReservation.status`(RESERVED/USED/EXPIRED)가 각자 깔끔한 enum으로 갈린다. 코드보다 문서(재설계 task 세트)를 먼저 B안으로 맞춘 뒤 구현으로 넘어갔다.

```
PaymentReservation (tbl_payment_reservation)  — 결제창 준비물
 ├─ order_id / provider / merchant_pay_key(UNIQUE) / amount
 ├─ status            RESERVED / USED / EXPIRED
 ├─ expires_at
 └─ reserved_key      RESERVED일 때만 "{order_id}:{provider}", 그 외 NULL → uk_reserved

Payment (tbl_payment)  — PG에 보낸 사건 (영구·append-only)
 ├─ order_id / provider / merchant_pay_key
 ├─ pg_payment_id     NOT NULL ← 복원
 ├─ type              APPROVE / CANCEL  ← RESERVE 제거
 ├─ status            REQUESTED / SUCCEEDED / FAILED / UNKNOWN
 ├─ amount / fail_code / fail_detail
 └─ approved_order_key  APPROVE+SUCCEEDED일 때만 order_id, 그 외 NULL → uk_approved
                        + (merchant_pay_key, provider, pg_payment_id, type) UNIQUE
```

옛 "성공 결제 1:1" 테이블(`uk_payment_order_id`로 주문당 1개 강제)은 폐기했다 — 그 사실(order_id·pgPaymentId·amount·approvedAt·provider)은 성공 APPROVE 사건 행이 이미 다 가진 *복사본*에 불과했기 때문. 완료 판단은 "마지막 행"이 아니라 EXISTS(성공 APPROVE 존재)로 두었다. 상세는 [[payment-append-only-원장과-exists-완료판단]].

## 두 테이블 관계 — merchant_pay_key 값 연결(FK 없음)

연결 고리는 `merchant_pay_key` **값**이고 FK 물리 제약은 두지 않는다(도메인 간 결합·마이그레이션·삭제 순서·픽스처 부담 회피). 관계는 `예약 1 : N Payment`(한 예약에서 승인 재시도 + 취소까지 여러 사건). FK를 값 참조로 두는 프로젝트 전반 컨벤션은 [[cross-aggregate-fk-to-id-참조-컨벤션]] 참조.

## reserved_key unique 거주지 이동

reserve 따닥(동시 중복요청)을 막는 unique(작업명 `uk_reserved`)의 거주지가 `tbl_payment` → `tbl_payment_reservation`으로 이동했다. "조건 맞을 때만 값, 아니면 NULL" 트릭 자체는 동일하다 — RESERVED일 때만 `"{order_id}:{provider}"`, USED/EXPIRED면 NULL. reserved_key는 provider를 포함해 (주문, 수단)당 RESERVED 1개를 보장하고, 이중결제용 approved_order_key는 반대로 provider를 안 섞어 전 PG 통틀어 주문당 성공 승인 1개를 보장한다(방어 대상이 다르다). NULL 트릭 구현·캡슐화 규칙은 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]].

## 흐름 변화 — 전이가 아니라 새 APPROVE 행 생성

거주지가 바뀌며 redirect 흐름도 달라졌다. RESERVE 행을 APPROVE로 "전이"시키는 게 아니라, **예약 행을 읽어 새 APPROVE 행을 생성**하고 예약을 USED로 마킹한다.

```
ready    → PaymentReservation(RESERVED) 생성/재사용/만료
redirect → merchant_pay_key로 예약 조회 → order_id 확보
           → Payment(type=APPROVE, REQUESTED) "새 행" 생성  ← 예약 행 전이 아님
           → [tx 밖] NaverPay 승인 → [tx 안] succeed + Order PAID + 예약 USED 마킹
```

예약 행 생성/재사용/만료 흐름(expiresAt 재사용, EXPIRED lazy 회수)은 [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]], 예약 엔티티의 `use`/`expire` 도메인 메서드 네이밍은 [[payment-attempt-네이밍-정리와-refactor-경계]]에 이어진다.

> [!note] 초기 설계 맹점 정정 (#205)
> B안 확정 당시엔 "만료 예약은 EXPIRED로 마킹하지 않고 재사용 판단 시 만료 시각 필터로만 거른다"였다. 그러나 만료 행은 상태가 여전히 RESERVED라 `reserved_key`를 계속 점유해, 만료 필터가 *재사용*만 막고 *새 발급*은 같은 키로 unique 위반을 일으켜 같은 주문이 영원히 재예약 불가가 됐다. 리뷰에서 드러나 EXPIRED를 재도입하되, reserve 진입 요청이 자기 자리를 lazy 회수(EXPIRED + reserved_key=NULL)한 뒤 새로 발급하도록 고쳤다("단순화 영향을 모든 경로에서 따져라"). 상세는 ready 흐름 노트.

## 트레이드오프 — 테이블 +1

치르는 비용은 테이블 수 +1 뿐이고, 의미·스키마 단순화 + 쿼리 명확성 + `pg_payment_id NOT NULL` 회복의 이득이 더 크다. 운영 데이터 없음 전제라 backfill 없는 단순 schema 변경으로 갔다. 나머지 재설계 결정(NULL 트릭 unique, EXISTS 완료 판단, unique vs lock 전략, append-only, 외부 호출 tx 밖)은 거주지가 바뀌어도 그대로 유효하다.

## 근거

- [[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]]
- [[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]
- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]
