---
platform: backend
author: KimYeonWook511
created: 2026-06-10
origin:
  - { type: pr, repo: commerce-backend, ref: 237 }
---

## 맥락
- PR #237 대사/보상 코드에서 드러난 결제-주문 결합 문제와 해소 방향. 후속 #240으로 분리.

## 문제 — 결제가 주문 상태 머신을 대신 돌림
- 조합 폭발의 원인은 모놀리식이 아니라 결제가 주문 상태에 결합된 설계. 결제 안에서 order.getStatus() 보고 if 분기 → 주문 상태 가짓수 × 결제 상태 가짓수.
- 현재 안티패턴: 대사 서비스 handleOrderNotCompletable이 order를 재조회해 취소/PAID/없음/기타 4분기로 환불을 판단. order.completePayment()가 이미 INIT(주문 결제 전 초기 상태) 가드로 판단하는데 뒤에서 또 재분석 = 판단 중복.

## 방향
- Tell-Don't-Ask: 결제가 주문 상태 묻지 말고 order.completePayment()로 시키고 판단은 주문 안에서.
- 단방향 결합: 결제→주문 결과 통보(양방향 끊으면 조합 절반).
- 조율은 facade: 여러 도메인 엮는 흐름은 결제도 주문도 아닌 facade. 각 도메인은 자기 일만.
- 보상(결제 뒤늦게 성공 ↔ 주문 이미 취소): 결제=성공 사실 확정만, 환불 필요 판단=주문/facade, 환불 실행(PG 취소)=다시 결제. 판단과 실행 분리(Saga 보상).
- 이벤트여야 하나? 아니다 — 결합 끊는 핵심은 facade로 조율 빼기. 모놀리식은 facade 직접 호출로 충분. 반응자 여럿(주문+알림+적립…)/MSA 분리 때 이벤트.
- facade까지 안 가는 중간 정리: "승인 확정 시도 → 성공이면 확정, 반영 불가 확정 예외(주문 종결/없음/중복)면 환불 종착, 일시적 예외면 재시도"로 order 4분기 제거. 단 일시적 실패(DB 오류)는 환불 금지 — 예외 종류로 구분.

## 다음 단계
- facade 도입(#240) — 다음 결제 작업 전 우선. dead 분기·order 상태 4분기 제거는 거기서.

## 관련
- 같은 PR: [[raw/sessions/backend/2026-06-10-pr-237-payment-status-fact-not-classification]], [[raw/sessions/backend/2026-06-10-pr-237-reconciliation-review-findings]]
