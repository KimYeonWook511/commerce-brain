---
type: tradeoff
status: decided
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, partial-cancel, refund, idempotency, unique-constraint, yagni]
created: 2026-06-04
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# 부분취소 — 모델만 열어두고(amount+SUM) 로직은 전액취소만 구현

부분취소를 지금 구현하지 않되, 나중에 켤 때 모델 수술·마이그레이션이 없도록 데이터 모델만 미리 열어둔 결정이다.

> [!note] 닫힘 (2026-08) — 예약해둔 설계가 실제로 이뤄졌고, 이 노트가 예고한 것 중 절반이 맞았다
> 부분취소 설계가 진행돼 이 tradeoff는 `decided`로 전환한다. **맞은 것:** "취소요청 단위 고유키 + 한도 검증"이라는 해법 방향, "amount는 신원이 아니다", 4컬럼 unique로는 표현 불가.
> **틀린 것:** 아래 "데이터 모델 변경 0, 잔액 조회 변경 0"은 성립하지 않았다. 잔액의 정본이 `SUM(APPROVE) − SUM(CANCEL)`(금액 기준)이 아니라 **주문 품목의 취소수량**(수량 기준)으로 갔다 — 금액 기준은 접수 시점 검증에 쓸 수 없기 때문이다([[부분취소-잔액-정본-수량기준-상태무관-누적]]). 유일 제약도 문자열 단일 컬럼으로 교체됐다([[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]]).
> 스코프·입력 형태는 [[부분취소-스코프-배송전-품목수량-기반-금액입력-기각]], 멱등키 스코프는 [[취소요청키-유일범위-주문단위-와-같은키-다른내용-거부]], 최종 모델은 [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]].

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
