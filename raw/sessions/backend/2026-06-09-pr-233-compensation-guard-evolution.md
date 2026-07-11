---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 233 }
---

# 이중결제 보상의 "결제 완료 가드" 제거 — 보상 대상 pgPaymentId 무조건 취소로 전환

이중결제가 감지되면 이미 승인 요청까지 나간 PG 결제를 다시 취소(환불)하는 보상 흐름이 있다. 이
보상은 PG 취소를 실제로 쏘기 전에 "이 merchantPayKey(결제창을 여는 예약 단위 식별자)로 완료된
결제가 이미 있나"를 조회해, 있으면 취소를 skip하는 가드를 두고 있었다. 그런데 결제 도메인 재설계로
한 merchantPayKey에 서로 다른 PG 결제(pgPaymentId)가 여러 건 매달릴 수 있게 되면서, 이 가드가
"보상 대상 자신"이 아니라 "형제 결제의 성공"을 잡아 중복 결제의 환불을 조용히 건너뛰고 이중청구를
냈다. 이 세션은 가드를 pgPaymentId 단위로 고쳐 다는 대신 아예 제거하는 쪽을 택했다. 이슈 #230, PR #233.

## 막힌 점·해결

### 가드가 "조용히" 오작동으로 바뀌어 이중청구를 냈다 — 세 단위의 어긋남

이 완료 가드는 한 번에 설계된 게 아니라 데이터 모델이 바뀔 때마다 재정의돼 왔고, 마지막에는 전제
자체가 깨진 채로 남아 있었다. 진화사를 따라가면 왜 결국 제거가 답이었는지가 드러난다.

- **최초:** 보상을 진행할지 말지를 `PaymentAttempt.status`(결제 시도 엔티티의 상태)로 판단했다. 그런데
  attempt에 락이 없어, race window에서 이미 성공한 결제에 취소가 호출되는 결함이 있었다(이슈 #114).
- **완료 Payment 존재 기준으로 전환:** 판단 기준을 "완료된 Payment row가 있나"로 바꿨다. 근거는
  "merchantPayKey = 결제 1건"이라는 당시 모델 전제였다 — 그 merchantPayKey에 완료 결제가 있으면 곧
  "이 결제가 이미 성공"이니 취소하면 안 된다. 이 판단은 `hasCompletedPayment(merchantPayKey)`
  (해당 merchantPayKey로 완료된 결제 존재 여부를 묻는 조회)로 표현됐다.
- **결제 도메인 재설계(#205)에서 전제가 조용히 깨짐:** Order와 Payment 경계를 분리하고 예약
  (reservation)을 별도 테이블로 두면서, 한 merchantPayKey(= 하나의 예약)에 서로 다른 pgPaymentId
  (PG 결제 식별자)의 결제가 여러 건 존재할 수 있게 됐다. 여기서 "merchantPayKey = 결제 1건"이라는
  가드의 전제가 명시적 재검토 없이 무너졌다.
- **이중결제 보상을 fail-first 단일 경로로 통합(#226)하며 버그가 발현:** 그 전까지 이중결제 보상은
  두 갈래였고 실제 실행되는 쪽은 이 완료 가드를 타지 않았다. #226에서 이중결제 실패 케이스를
  하나의 보상 경로로 모으면서, 원래 이 가드를 안 거치던 `PAYMENT_DUPLICATE`(이중결제 도메인 오류)
  경로도 이제 이 가드를 통과하게 됐다. 그 결과 잠복해 있던 결함이 실제 이중청구로 터졌다.

**버그 메커니즘:** 같은 merchantPayKey에 두 PG 결제 pgA·pgB의 승인이 경합한다. pgA가 먼저 성공하며
주문 단위 unique 제약 `uk_payment_approved_order_key`(한 주문에 성공 APPROVE 하나만 허용하는 제약,
approved_order_key에 orderId를 점유)를 차지한다 → pgB는 같은 제약 위반으로 `PAYMENT_DUPLICATE`가 되어
보상에 진입한다. 이때 가드가 `hasCompletedPayment(merchantPayKey)`를 보는데, 형제 pgA가 이미 성공했으니
결과는 true → pgB의 PG 취소를 skip → pgB는 환불되지 않음 → 이중청구.

근본 원인은 **세 가지가 서로 다른 단위로 동작한다는 어긋남**이다:

- 제약(`uk_payment_approved_order_key`)은 **orderId(주문) 단위**
- 가드(`hasCompletedPayment`)는 **merchantPayKey(예약) 단위**
- 보상이 취소하려는 대상은 **pgPaymentId(개별 PG 결제) 단위**

가드가 merchantPayKey 단위라, "보상 대상 자신(pgB)"이 아니라 "형제(pgA)의 성공"을 잡아 항상 잘못
발동했다. 세 단위를 맞대어 보면 결함이 곧바로 드러난다.

## 결정한 것 — 왜 "재정의"가 아니라 "제거"인가

이중결제 보상이 취소할 PG 결제를 결정하기 전에 완료 여부를 확인하던 가드를, 보상을 담당하는
`PaymentApprovalCompensationService`의 공용 취소 실행부(`runPgCancel`)에서 제거했다. 이제 이중결제
(`compensateDuplicatePayment`)·금액 불일치(`compensateAmountMismatch`) 보상 모두 이 공용부를 거쳐
보상 대상 pgPaymentId를 **무조건 PG 취소**한다.

- **검토한 대안 — 가드를 pgPaymentId 단위로 좁히기:** merchantPayKey가 아니라 "보상 대상 자신(pgB)이
  성공(SUCCEEDED)했나"를 보게 시그니처를 좁히는 안을 먼저 검토했다.
- **기각 근거 — 그 가드는 항상 false인 dead 코드가 된다:** 보상 진입 경로를 끝까지 따지면, 보상
  대상 pgPaymentId 자신은 절대 SUCCEEDED로 커밋될 수 없다. 현재 보상 진입 경로는 금액 불일치
  (amount-mismatch)와 이중결제(duplicate) 둘뿐인데 — amount-mismatch는 승인 저장(`saveApproved`)에
  도달하기 전 검증 단계에서 막히고, duplicate는 자기 자신의 `succeed`가 `uk_payment_approved_order_key`
  위반으로 롤백된다. 어느 경로든 보상 대상 pgPaymentId는 성공 상태로 남지 못하므로, pgPaymentId 단위
  가드는 언제나 false를 돌려주는 죽은 조건이 된다. 사용처 없는 코드를 남기지 않는 원칙에 따라
  재정의가 아니라 제거를 택했다.
- **제거가 안전한 근거:** 보상이 생성·조회하는 취소 결제(cancel payment)는 항상 보상 대상
  pgPaymentId(실패한 그 결제)로 만들어진다. 그러니 그 pgPaymentId를 취소하는 건 언제나 옳다. 형제의
  성공(pgA)은 별도의 Payment row이고 보상이 아예 건드리지 않으므로, 완료 가드가 없어도 이 가드가
  원래 막으려던 위험 — "실제로 성공한 결제를 보상이 잘못 취소" — 은 발생하지 않는다.
- **금액 불일치 보상도 같은 결함을 공유했다:** amount-mismatch 보상도 이 공용 취소부를 함께
  쓰므로 같은 merchantPayKey 단위 가드로 형제 성공을 오탐할 수 있었고, 마찬가지로 보상 대상 자신은
  SUCCEEDED가 될 수 없어 가드가 무용했다. 가드 제거가 두 경로 모두에 옳다는 걸 확인하고 함께 정리했다.
- **멱등 안전망은 유지:** 취소 대상 결제(cancelPayment)가 이미 `REQUESTED`(PG 취소 요청만 만들어진
  초기 상태)가 아니면 취소를 skip하는 멱등 가드는 그대로 뒀다. 이미 처리된 취소를 다시 쏘는 걸 막는
  건 완료 가드와 별개의 안전망이다.
- **dead가 된 조회 체인도 함께 정리:** 가드가 사라지면서 이 가드에서만 쓰이던 조회 메서드 체인
  (`hasCompletedPayment` → 리포지토리의 `existsApproveSucceeded`(해당 merchantPayKey에 성공 APPROVE가
  있는지) → JPA 존재 조회)이 통째로 dead가 되어 함께 제거했다.

## 배운 것

- **과거의 안전장치는 그 시점의 데이터 모델 전제 위에 서 있다.** 모델이 1:1에서 1:N으로 바뀌면
  (여기선 merchantPayKey : 결제) 과거 가드가 코드 한 줄 안 바뀌고도 "조용히" 오작동으로 변한다.
  모델을 바꾸는 작업(결제 도메인 재설계)에서 그 모델에 기대던 기존 가드들의 전제를 함께 점검했어야 했다.
- **버그를 만나면 안전장치를 "추가"하려는 본능이 있는데, 이번엔 반대로 "전제가 깨져 dead가 된
  가드를 식별해 제거"하는 게 정답이었다.** 방향을 정하기 전에 "가드가 보호하려던 위험(성공한 결제를
  잘못 취소)이 현재 모델에서 실제로 발생 가능한가", "보상 대상 자신이 성공 상태로 커밋될 경로가
  있는가"를 진입 경로별로 먼저 따졌다. 그 확인 덕에 pgPaymentId 단위 재정의가 dead 가드가 됨을 결정
  전에 알았고, "더 남기는" 대안에 끌리지 않을 수 있었다.
- **한 동작이 여러 단위(주문 / 예약키 / PG결제)에 걸칠 때, 가드·제약·보상 대상이 각각 어느 단위로
  동작하는지 맞대어 보면 이런 종류의 버그가 드러난다.** 여기서도 제약(orderId)·가드(merchantPayKey)·
  대상(pgPaymentId)의 단위 차이가 결함의 근원이었다.
