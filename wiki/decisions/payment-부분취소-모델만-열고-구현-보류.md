---
type: tradeoff
status: open
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, partial-cancel, refund, idempotency, unique-constraint, yagni]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# 부분취소 — 모델만 열어두고(amount+SUM) 로직은 전액취소만 구현

부분취소를 지금 구현하지 않되, 나중에 켤 때 모델 수술·마이그레이션이 없도록 데이터 모델만 미리 열어둔 결정이다. 아직 부분취소 설계 자체는 안 했으므로 **열린 tradeoff**로 둔다.

## 방침 — 전액만 구현·모델만 개방(YAGNI)

우선 전액취소만 운영한다. 부분취소 로직(입력 파라미터, 과다취소 검증, PG cancelAmount, UI)까지 미리 만들지 않는다 — YAGNI. 다만 확장 시 모델 수술이 지옥이 되지 않도록 **모델만 열어둔다**. 부분취소 요구가 실제로 생기면 이 tradeoff는 별도 설계 raw와 함께 decided로 전환한다.

## 분기점 — 금액 가진 CANCEL 행으로 모델링

취소를 boolean/status로 모델링하면 부분취소 추가가 모델 수술 + 마이그레이션이 된다. 대신 **금액을 가진 CANCEL Payment 행으로** 모델링하면 전액취소는 부분취소의 특수 케이스(취소액 = 원금)라 기능 추가 수준으로 끝난다. 이는 append-only 원장 모델의 자연스러운 귀결이다 — [[payment-append-only-원장과-exists-완료판단]].

## 지금 해두는 것 — amount 기록 + SUM 잔액 도출

1. CANCEL Payment 행에 `amount`를 둔다. 전액취소라도 실제 금액을 적는다.
2. 잔액은 저장하지 않고 SUM으로 도출한다.

```sql
SELECT SUM(CASE WHEN type='APPROVE' THEN amount ELSE 0 END)
     - SUM(CASE WHEN type='CANCEL'  THEN amount ELSE 0 END)
FROM payment
WHERE order_id = :orderId AND status = 'SUCCEEDED';
```

각 시도는 "그 시도가 움직인 금액"을 적고(APPROVE=+, CANCEL=−), 계산값(현재 잔액)은 행에 박지 않는다. 이중결제 제약("주문당 성공 APPROVE 1개")은 APPROVE 성공에만 걸리므로 CANCEL이 여럿 SUCCEEDED여도 무관하다 — [[payment-이중결제-reserve따닥-mysql-null트릭-unique]].

## 켤 때 변경 범위(파라미터·과다취소 lock·PG·UI)

데이터 모델 변경 0, 잔액 조회 변경 0. 추가되는 것은:

- 취소금액 파라미터
- **과다취소 검증** — order PK 단일 행을 `FOR UPDATE`로 잠근 위에서 "취소액 합 ≤ 승인액". 여러 행을 합산해 판단하는 계산성 검증이라 unique로 표현 불가, 한 트랜잭션 안에서 끝나므로 lock이 적합하다 — [[payment-동시성-unique-vs-lock-gap-lock회피]].
- PG cancelAmount, UI

## 부분취소 멱등은 4컬럼 unique로 표현 불가 → 취소요청 키 + 한도검증

> [!warning] 전액취소 멱등 장치가 부분취소로는 확장 불가
> 전액취소의 멱등은 기존 4컬럼 unique `(merchant_pay_key, provider, pg_payment_id, type)`가 진다 — `type`에 CANCEL이 들어 한 (merchantPayKey, pgPaymentId)에 CANCEL 하나만 존재할 수 있어 이미 하드 보장이다. 그러나 부분취소는 **같은 (merchantPayKey, pgPaymentId)에 금액만 다른 CANCEL이 여럿**이라 이 unique로 표현할 수 없다. amount를 unique에 끼워 넣는 것도 답이 아니다 — 같은 금액을 두 번 취소할 수 있어 **amount는 신원이 아니다**.

올바른 부분취소 멱등 모델은 두 장치의 조합이다(성격이 다른 문제를 각각 막음).

- **취소 요청마다 고유 키**(클라이언트 발급 또는 결정론적 파생)를 두고 그 키에 unique — 재시도 중복(같은 취소 두 번 환불) 방지.
- **잠금 + "취소액 합 ≤ 승인액" 한도 검증** — 동시에 다른 키로 같은 물건을 두 번 환불하는 것 방지(멱등키로는 못 막음).

이 취소요청 키 + 한도 검증 모델은 별도 설계가 필요하며, 전액취소 구현·4컬럼 unique 발견의 상세 맥락은 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]에 있다.

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 전액만 구현·모델만 개방(YAGNI), 금액 가진 CANCEL 행, amount+SUM 잔액 도출, 켤 때 변경 범위.
- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]] — 4컬럼 unique의 부분취소 표현 불가, amount는 신원 아님, 취소요청 키 + 한도 검증 모델.
