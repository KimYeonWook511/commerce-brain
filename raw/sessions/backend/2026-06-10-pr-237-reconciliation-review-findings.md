---
platform: backend
author: KimYeonWook511
created: 2026-06-10
origin:
  - { type: pr, repo: commerce-backend, ref: 237 }
---

## 맥락
- PR #237 대사/만료 가드에서 코드 리뷰로 발견한 정합성 버그와 테스트 철학.

## 막힌 점
### 중복 결제 환불 누락(돈 박제)
- 중복 결제 시 승인 확정이 order.completePayment() 도달 전 저장 단계의 unique 제약(주문당 성공 승인 1개) 위반으로 결제 도메인 예외(PaymentException, 코드 PAYMENT_DUPLICATE — adapter가 DuplicateKeyException을 도메인 예외로 번역)를 던지는데, 대사는 주문 도메인 예외(OrderException, ORDER_PAID_NOT_ALLOWED)만 catch → 결제 예외가 안 잡혀 일반 catch로 빠짐 → 결제 미확정으로 남고 매 주기 재스캔. PG에 청구된 중복 결제가 영구 미환불 = 돈 박제.
- 단위 테스트가 승인 확정을 mock으로 "주문완료 불가 예외"를 강제해 비현실 경로를 검증 → 못 잡음. 통합 테스트(실 DB)였으면 잡혔다. (GitHub 자동 리뷰어는 못 잡았고 독립 코드리뷰 에이전트와 codex가 발견)
- 수정: 대사에 결제 도메인 예외(PAYMENT_DUPLICATE) catch 추가, 단위 테스트 현실 경로 교정.

### 대사 승인 확정 시 키/금액 검증 누락
- 실시간 승인은 PG 응답의 가맹점결제키/금액을 검증하고 불일치면 보상하는데, 대사는 검증 없이 확정 → 불일치 박제 위험. 실시간과 대칭으로 검증 추가.

### 만료 차단·스캔 하한 추가 발견 (codex 리뷰)
- 만료 차단 쿼리가 결과불명만 보는데, 미확정 요청중(승인 호출 후 결과 저장 전 중단되어 과금됐을 수 있음)도 차단해야 만료-대사 경합 방지 → 요청중 포함.
- 스캔 쿼리가 요청중에도 1분 하한인데 정책은 15분 → 1~15분 요청중이 첫 페이지 차지하고 매번 버려져 뒤의 실제 대상이 고사 → 요청중은 15분 하한으로 분리(결과불명은 1분 유지).

## 배운 것 — 테스트 철학
- 도달 불가 분기(주문 PAID인데 이 결제가 성공 주체 — 승인 확정이 결제성공+주문완료를 원자로 하므로 불가)를 "혹시 모르니" 방어 코드로 두지 마라. 검증 안 되고 거짓 안전감만.
- 대신 "발생 안 함"을 불변식 테스트로 못 박고(주문 PAID면 항상 성공한 승인이 존재), 코드는 단순화하고, 실제 발생하는 상황(중복 결제 환불)의 돈 정합성을 통합 테스트로 검증.

## 관련
- 같은 PR: [[raw/sessions/backend/2026-06-10-pr-237-payment-status-fact-not-classification]], [[raw/sessions/backend/2026-06-10-pr-237-payment-order-decoupling]]
