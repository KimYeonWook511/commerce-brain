---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, order, transaction-boundary, cross-aggregate, outbox, saga, consistency]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
  - "[[raw/sessions/backend/2026-06-11-payment-order-facade-decoupling]]"
---

# 트랜잭션 경계 — 외부 호출은 밖, order·payment 두 aggregate 쓰기는 한 트랜잭션

결제 확정·환불에서 트랜잭션 경계를 어디에 긋는가에 대한 원칙 결정이다. 재설계 허브는 [[payment-order-도메인분리와-pg격리]].

## 문제 — 분리 커밋 불일치 vs 외부호출 커넥션 점유

승인 성공 시 `payment(SUCCEEDED)`와 `order(PAID)` 두 쓰기가 따로 커밋되면 "결제됐는데 주문 미결제"(또는 반대) 불일치가 난다. 반대로 외부 API 호출을 트랜잭션 안에 넣으면 커넥션을 수 초~수 분 오래 점유한다(동시 사용자 많으면 커넥션 풀 고갈).

## 결정 — 외부호출은 밖·DB쓰기는 한 트랜잭션

```text
[트랜잭션 밖]  네이버 승인/취소 API 호출 → 결과 수신
[트랜잭션 안]  payment SUCCEEDED(+ approved_order_key set) + order PAID   ← 같은 트랜잭션
```

외부 호출(승인/취소)은 트랜잭션 밖에서, 그 결과를 받아 **DB 반영(payment + order)은 하나의 짧은 트랜잭션**으로 묶는다. 승인 성공 후 DB 반영 실패("박제")도 이 경계에서 발생하므로, 반영 실패 시 UNKNOWN 흔적으로 처리한다 — [[payment-unknown-결과불명-처리와-예외분류]].

## 단일 DB로 cross-aggregate 정합(이벤트/Outbox 불필요)

order와 payment는 다른 aggregate지만, **단일 DB라는 조건을 활용하면 이벤트/Outbox 없이 트랜잭션 하나로 두 aggregate에 걸친 정합(cross-aggregate consistency)을 확보**할 수 있다. order·payment 참조는 FK 물리 제약 없이 order_id 값으로만 잇는다 — [[cross-aggregate-fk-to-id-참조-컨벤션]].

모놀리식에선 조율자(facade)가 각 도메인을 직접 호출하는 것으로 충분하고, 결제→주문 단방향 결합을 facade 한 점에 격리한다 — [[payment-order-facade-결합끊기-tell-dont-ask]]. 두 도메인을 엮는 판단은 누군가 반드시 지는데, 그게 도메인이 아니라 조율자인 게 핵심이다.

## Outbox 승격 여지(결합 커질 때)

반응자가 여럿이거나 MSA로 결제를 분리하는 시점이 이벤트 도입 시점이다.

- 결합이 커지면(부분취소·다채널) 동기 호출을 이벤트(결과적 일관성)로 바꾸고 Outbox/Saga로 신뢰성·보상을 보강한다.
- 전환은 자연스럽다 — **영속된 REQUESTED(환불 의도) 행이 이벤트와 사실상 같은 역할**을 하기 때문이다. FK 물리 제약을 이미 안 쓰므로 참조 방식은 그대로다.

## 취소 시 order 상태전이도 같은 트랜잭션

PAID 주문 취소·환불에서도 같은 경계가 적용된다 — PG 취소 호출은 트랜잭션 밖(best-effort), 그 전에 **한 RDB 트랜잭션에서 `CANCEL 결제 REQUESTED(=환불 의도) 영속화 + order 취소 전이 + 재고 복구`를 원자적으로 커밋**한다. 어느 시점에 중단돼도 "이 결제는 환불해야 함"이라는 durable 기록이 DB에 남고, 그 마무리는 standalone CANCEL 대사가 진다 — 상세 구현·안전망 발견은 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]].

- **검토한 대안 — Outbox 이벤트**: 주문 트랜잭션에 "환불 필요" 이벤트를 원자적으로 적고 별도 consumer가 집행. aggregate 경계는 더 깨끗하나 환불 전용 이벤트·consumer를 새로 만들어야 해 현재 스코프에 과하다고 봐 기각(미래 탈출구로 남김).
- **트레이드오프**: 조율 service의 단일 트랜잭션이 order와 payment 두 aggregate 테이블을 함께 쓴다(경계 침범). 위 Outbox 승격을 탈출구로 두고 지금은 감수했다.

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 외부호출 밖·DB쓰기 한 트랜잭션 원칙, 박제 반영 실패의 UNKNOWN 처리.
- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]] — 단일 tx cross-aggregate 정합, Outbox 대안 기각과 승격 여지, 취소 시 order 전이 동일 트랜잭션.
- [[raw/sessions/backend/2026-06-11-payment-order-facade-decoupling]] — 모놀리식 facade 조율로 이벤트 불필요 판단.
