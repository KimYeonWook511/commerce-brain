---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, append-only, event-sourcing, order, fail-code, ledger]
created: 2026-06-04
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# 결제 시도 append-only 원장과 EXISTS 기반 완료 판단

결제 재설계([[payment-order-도메인분리와-pg격리]])의 핵심 데이터 모델 결정이다. 시도를 불변 이벤트로 쌓고, "현재 상태"는 저장하지 않고 시도들로부터 도출한다.

## 문제 — 마지막 행 ≠ 현재 상태

status 하나에 `RESERVED → COMPLETED → (CANCELED/FAILED)` 전이를 다 태우면 한 컬럼이 두 정보를 떠안는다 — (a) 결제의 현재 유효 상태, (b) 마지막 시도의 결과. 특히 "마지막 행"은 "마지막 시도 결과"지 "현재 상태"가 아니다.

예: `#1 APPROVE SUCCEEDED`, `#2 CANCEL FAILED`(마지막). 마지막 행만 보면 취소 시도 실패로 오판하지만 실제 상태는 여전히 PAID다.

## 결정 — append-only 시도 + EXISTS 완료 판단

시도(Payment 행)는 **불변 이벤트**다. 한 번 일어나면 바뀌지 않는다 — 재시도로 성공해도 "실패했던 그 시도"가 성공으로 변하는 게 아니라 *새 시도가* 성공한 것이다. 그래서 시도를 append-only로 쌓고, 현재 상태는 조건의 존재로 도출한다.

```text
결제 완료 = (성공한 APPROVE 존재) AND (그것을 무효화한 성공 CANCEL 부재)
```

| 성공 APPROVE | 성공 CANCEL | 현재 상태 |
|---|---|---|
| 없음 | - | 미결제(진행중/실패) |
| 있음 | 없음 | **PAID** |
| 있음 | 있음(전액) | 취소됨 |
| 있음 | 있음(부분) | 부분취소(일부 유효) |

외부 PG 연동에서는 "결과 모름(UNKNOWN)"도 한 시도로 남겨야 하므로 시도 단위 기록이 필수다 — [[payment-unknown-결과불명-처리와-예외분류]]. 환불도 append-only 원장이라 승인 행을 mutate하지 않고 **별도 CANCEL 레코드 append**로 표현한다(승인의 SUCCEEDED 상태는 보존) — 실제 구현은 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]].

이 EXISTS 완료 판단을 "정책 계산이 아닌 단순 사실 조회"로 캡슐화한 도메인 메서드가 `hasCompletedPayment`다 — [[payment-완료여부-사실조회-hascompletedpayment-srp]]. 그리고 이 불변식("주문당 성공 APPROVE 1개")을 DB가 물리적으로 강제하는 장치가 NULL 트릭 unique다 — [[payment-이중결제-reserve따닥-mysql-null트릭-unique]].

## 완료상태 테이블 제거(복사본)

재설계 전 코드에는 결제 테이블이 둘이었다 — 시도 로그 `tbl_payment_attempt`와 완료 상태를 담는 별도 `tbl_payment`(`uk_payment_order_id`로 "주문당 결제 하나" 강제). 그런데 승인 성공 시도가 이미 완료에 필요한 모든 사실(order_id, pgPaymentId, 금액, 시각, 수단)을 들고 있어, 별도 완료-상태 테이블은 **"성공 시도의 복사본"일 뿐**이었다.

- 별도 완료-상태 테이블을 폐기하고 시도 단일 테이블로 통일한다. "결제 완료 = 성공한 APPROVE 행이 존재함"으로 정의한다.
- "주문당 하나" 보장은 완료-상태 테이블의 `uk_payment_order_id`가 아니라 시도 테이블의 NULL 트릭 unique로 옮긴다.
- 집계 테이블(`PaymentSummary` 등)이 정당해지는 유일한 경우는 부분취소처럼 여러 시도를 집계해야 잔액이 나오고 그 계산이 잦고 무거울 때다. 지금은 불필요.

> 이후 예약/사건 두 테이블 분리([[payment-reserve-예약테이블-분리-a안-b안]])로, `tbl_payment`는 APPROVE/CANCEL "PG 사건"만 담는 순수 append-only 테이블이 됐다.

## 실패사유는 DB 컬럼·원문은 로그로(승격 트리거)

추상화 수준이 다른 두 가지를 구분한다.

- **원문 전체(PaymentEvent류)** = PG 응답 날것 JSON → 무거움 → **로그로 미룸**(YAGNI). 단 원문 JSON은 분쟁 시 유일한 증거라 로그에라도 반드시 남기고, 승격 트리거(분쟁/CS 증가, 대사 자동화 필요, 결제량 규모 초과 → `PgTransactionLog`류 테이블)를 정해둔다. 로그 대체의 취약점(조건 쿼리 불가, 로테이션 소실, 정합성 약함)은 인지하고 감수한다.
- **실패 사유** = 해석된 요약 → 쿼리·안내·분석·재시도 분기에 직접 쓰임 → **DB(Payment 행) 컬럼으로 유지**. `failCode`(정규화 enum: 잔액 부족/시간 만료/중복 결제/사용자 취소/결과 모름 …) + `failDetail`(PG 원문 메시지 요약, length 255 제한). PG마다 에러 코드가 달라 분석·분기는 내 코드 기준이어야 일관되며, UNKNOWN도 사유를 남겨 대사 단서로 쓴다.

"현재 상태엔 FAILED를 안 넣는다"와 모순 아니다 — 현재 결제 *상태*엔 FAILED가 없지만, 개별 *시도*의 결과로서 FAILED + 사유를 남기는 건 정상이다. 금액 컬럼은 `int`(원 단위 정수, 상한 약 21.4억 원만 의식)로 통일한다 — 주문 스냅샷 단가도 같은 맥락 [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]].

## 트레이드오프 — 집계 조회 비용

"이 주문 결제가 지금 유효한가"를 알려면 APPROVE·CANCEL 레코드를 집계해야 한다. 단일 row를 mutate하는 모델보다 조회가 복잡하다. 그러나 append-only 원장이 감사·분쟁 대응·미래 부분취소 확장에서 주는 일관성이 그 비용보다 크다고 봤다. 잔액도 저장하지 않고 `SUM(APPROVE) − SUM(CANCEL)`로 도출한다 — [[payment-부분취소-모델만-열고-구현-보류]].

## 미해결 — PaymentSummary·원문테이블 승격

| 항목 | 현재 | 승격 조건 |
|---|---|---|
| 집계 테이블 `PaymentSummary` | 안 만듦 | 부분취소 도입 + 잔액 SUM이 잦고 무거울 때 |
| PG 원문 보관 테이블 | 로그로 대체(원문 보존 필수) | 분쟁/CS 증가, 대사 자동화, 결제량 규모 초과 |

> [!note] 두 승격 조건이 모두 충족됐다 (2026-08)
> - **집계는 별도 테이블이 아니라 결제 행의 누적 환불액 컬럼으로 갔다.** 근거가 성능이 아니라 **동시성**이었다 — 합계를 조회하면 두 요청이 각자 한도를 통과해도 부모 버전이 안 올라 둘 다 커밋된다([[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]). 그리고 잔액의 정본은 금액 합이 아니라 주문 품목의 취소수량이 됐다([[부분취소-잔액-정본-수량기준-상태무관-누적]]).
> - **PG 원문 보관 테이블이 만들어졌다** — 회수 배치가 결제사 응답을 근거로 결과를 확정해야 해서다. **aggregate 밖에 두고 낙관 락을 갖지 않는 쌓기 전용**이다([[외부-호출기록-aggregate-밖-낙관락-없는-쌓기전용]]).
> - **환불을 승인 행 mutate가 아니라 별도 레코드로 표현한다**는 아래 원칙은 그대로 유지되며, 시도마다 새 행을 쌓는 이유(이력 보존)가 유일 슬롯 재점유 설계에서 다시 결정적 근거로 쓰였다([[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]).

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 이력/현재 상태 분리, EXISTS 완료 판단, 완료상태 테이블 제거, failCode/failDetail, 원문 로그 대체.
- [[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]] — 옛 완료-상태 `Payment` 폐기가 성공 APPROVE 복사본이라는 확인, EXISTS 조회로 정착.
- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]] — 환불을 승인 행 mutate가 아니라 CANCEL 레코드 append로 표현.
