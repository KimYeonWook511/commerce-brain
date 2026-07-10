---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 233 }
---

## 한 일
- 이중결제 보상에서 "결제 완료 가드"를 제거했다. 이슈 #230, PR #233. 정본: `docs/adr.md`(ADR-035), `docs/tasks/compensation-completed-guard-removal/retrospective.md`.
- 가드는 PG 취소 보상 전에 `hasCompletedPayment(merchantPayKey)`(해당 merchantPayKey로 완료된 결제가 있나)를 확인해, 있으면 PG 취소를 skip하던 코드였다.
- 가드 제거로 dead가 된 조회 메서드 체인(`hasCompletedPayment` → `existsApproveSucceeded` → JPA 존재 조회)도 함께 정리했다.

## 보상 가드 정책의 진화사 (왜 결국 제거됐나)
이 가드는 한 번에 만들어진 게 아니라 모델이 바뀔 때마다 재정의돼 왔고, 결국 전제가 깨져 제거됐다:
1. 최초: 보상 진행 여부를 `PaymentAttempt.status`로 판단 → attempt에 락이 없어 race window에서 이미 성공한 결제에 취소가 호출되는 결함(이슈 #114).
2. ADR-014: 판단 기준을 "완료된 Payment row 존재 여부"로 바꿈. 근거는 "merchantPayKey = 결제 1건"이라는 옛 모델 전제 — 그 merchantPayKey에 완료 결제가 있으면 곧 "이 결제가 이미 성공" = 취소하면 안 됨.
3. payment-order-redesign(#205): 한 merchantPayKey(=하나의 reservation)에 서로 다른 pgPaymentId(PG 결제 식별자)의 결제가 여러 건 가능해짐. 여기서 "merchantPayKey = 결제 1건" 전제가 조용히 깨졌다.
4. ADR-033(#226): 이중결제 보상을 fail-first 단일 경로로 통합 — 실패 케이스를 먼저 하나의 보상 경로로 모아 처리하게 바꾸면서, 기존엔 이 가드를 안 타던 PAYMENT_DUPLICATE 오류 경로도 이 가드를 거치게 됐다. 그 결과 잠복했던 버그가 발현.
5. ADR-035(이번, #230): 가드 제거.

## 버그 메커니즘
- 같은 merchantPayKey에 pgA, pgB 승인이 경합. pgA가 먼저 성공(주문 단위 unique 제약 `uk_payment_approved_order_key`에 orderId 점유) → pgB는 같은 제약 위반으로 PAYMENT_DUPLICATE → 보상 진입.
- 이때 가드가 `hasCompletedPayment(merchantPayKey)`를 보는데, 형제 pgA가 성공했으니 true → pgB의 PG 취소를 skip → pgB 환불 안 됨 → 이중청구.
- 근본 원인은 세 가지가 서로 다른 단위라는 어긋남이다: **제약은 orderId(주문) 단위, 가드는 merchantPayKey 단위, 보상 대상은 pgPaymentId 단위**. 가드가 merchantPayKey 단위라 "보상 대상 자신(pgB)"이 아니라 "형제(pgA)의 성공"을 잡아 항상 잘못 발동했다.

## 결정한 것 — 왜 "재정의"가 아니라 "제거"인가
- 검토한 대안: 가드를 pgPaymentId 단위로 좁혀 "보상 대상 자신(pgB)이 성공했나"를 보게 하는 것.
- 기각 근거: 보상 진입 경로를 따지면 보상 대상 pgPaymentId 자신은 절대 성공(SUCCEEDED) 상태로 커밋될 수 없다. (현재 보상 진입 경로는 amount-mismatch·duplicate 둘뿐이다.) amount-mismatch는 검증 단계에서 막혀 저장에 도달 못하고, duplicate는 자기 succeed가 제약 위반으로 롤백된다. 따라서 pgPaymentId 단위 가드는 항상 false인 dead 코드가 된다.
- 결론: 보상이 취소하는 대상은 항상 "실패한 그 pgPaymentId"이므로 무조건 취소가 옳다. 형제의 성공은 별도 row라 보상이 건드리지 않는다. amount-mismatch 보상도 같은 가드를 공유했으므로 함께 정리됐다.
- 멱등 안전망(취소 대상 결제가 이미 REQUESTED(PG 취소 요청만 만들어진 초기 상태)가 아니면 skip)은 유지했다.

## 배운 것
- 과거의 안전장치는 그 시점 데이터 모델 전제 위에 서 있다. 모델이 1:1 → 1:N으로 바뀌면(여기선 merchantPayKey:결제) 과거 가드가 "조용히" 오작동으로 바뀐다. 모델을 바꾸는 작업(payment-order-redesign)에서 기존 가드들의 전제를 함께 점검했어야 했다.
- 버그를 만나면 안전장치를 "추가"하려는 본능이 있는데, 이번엔 반대로 "전제가 깨져 dead가 된 가드"를 식별해 제거하는 게 정답이었다. 가드가 보호하려던 위험(성공한 결제를 잘못 취소)이 현재 모델에서 발생 가능한지부터 따졌다.
- 한 동작이 여러 단위(주문/결제키/PG결제)에 걸칠 때, 가드·제약·대상이 각각 어느 단위로 동작하는지 맞춰보면 이런 종류의 버그가 드러난다.
