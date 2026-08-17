---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, domain-service, srp, naming-convention, idempotency, compensation]
created: 2026-06-02
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-192-payment-completion-fact-query]]"
---

# 보상 스킵 판단을 Payment 도메인 사실 조회로 — hasCompletedPayment

## 컨텍스트·문제 — 이름에 보상 컨텍스트 누출, 본문은 단순 row 존재

결제 승인이 실패해 PG 취소(보상)로 넘어갈 때, "이미 결제가 완료됐으면 취소를 건너뛴다"는 가드가 있다. 이 가드가 `PaymentApprovalService.isCompensationRequired(merchantPayKey)`로 표현돼 있었는데, 이름은 "보상이 필요한가"를 묻지만 실제 하는 일은 "완료된 Payment row 가 있느냐"였다. 사실 조회 메서드가 자기를 부르는 쪽(보상 흐름)의 용도를 이름에 담고 있는 꼴 — 보상(compensation)이라는 상위 흐름 컨텍스트가 도메인 조회 메서드 이름으로 누출됐다.

게다가 초기 구현은 `findByMerchantPayKey(...).isPresent()`, 즉 status 를 보지 않는 단순 row 존재 여부였다. 이름("완료된 Payment 존재")과 본문("그냥 Payment row 존재")이 어긋나 있었다.

## 검토한 대안 A/B/C

- **A) `isCompensationRequired`를 public 도메인 메서드로 그대로 유지.** 판단을 Payment 도메인이 소유하고, 나중에 Payment 도메인이 별도로 분리될 때 외부 API 로 승격되는 경로가 보존된다. 다만 이름에 보상 흐름 컨텍스트가 계속 누출된다.
- **B) 판단을 보상 서비스(`PaymentApprovalCompensationService`)의 private 메서드로 이동.** 이슈가 제안한 방향. 보상 로직끼리 응집되고 캡슐화된다. 그러나 "결제 완료 여부"라는 도메인 사실 판단이 보상 서비스 안에 박혀, 판단의 근거가 바뀌면 엉뚱한 곳(보상 서비스)을 고치게 된다.
- **C) 이름을 `hasCompletedPayment`로 바꾸되 위치는 Payment 도메인 서비스에 유지.** 사실 조회(결제 완료됐나)는 도메인이 소유하고, 정책 적용(그러니 취소를 skip)은 보상 서비스가 `if (hasCompletedPayment) skip` 한 줄로 담당한다. 별도 정책 메서드는 만들지 않았다.

## 결정 — C 채택, 근거는 "도메인 단정의 소유"

C 를 최종 채택했다. 처음엔 C 를 약한 근거(A 의 이름 컨텍스트 누출을 피한다 정도)로 골랐는데, 추가 리뷰 관점에서 근거가 단단해졌다.

`exists` 한 줄은 단순한 Optional/존재 문법이 아니라 **"row 존재 = 결제 완료"라는 도메인 단정**이다. 그 단정이 성립하는 근거 — merchantPayKey unique 제약과, Order 를 FOR UPDATE 로 잠근 안에서 저장하므로 동시성 안전 — 이 전부 Payment 쪽에 있다. 근거가 Payment 에 있으니 단정도 소유자(Payment 도메인 서비스)에 박아 둬야, 완료의 정의가 바뀔 때 한 곳만 고친다. B 처럼 보상 서비스로 내리면 근거와 판단이 분리돼 유지보수가 엇나간다.

## 구현 — status 명시 exists 로 이름=의미 일치

AI 코드 리뷰(gemini)는 여기서 마이크로 최적화(`findBy...`로 엔티티를 끌어오지 말고 `existsBy...`로 존재만 확인)만 제안했다. 여기에 "메서드 이름은 `hasCompletedPayment`인데 status 를 안 보면 이름과 본문 의미가 어긋난다"는 지적을 더해, 최종적으로 `existsByMerchantPayKeyAndStatus(merchantPayKey, COMPLETED)`로 갔다. exists 로 효율을 얻으면서 status 조건까지 넣어 "완료된" 결제를 실제로 조회하게 만들어, 최적화와 의미 일치를 동시에 확보했다.

핵심은 안전망 관점이다. `hasCompletedPayment`가 status 를 안 볼 때도 우연히 맞게 동작한 건 "Payment 는 완료(COMPLETED) 상태로만 저장된다"는 불변식 덕분인데, 그 불변식은 코드로 강제되지 않는다. 상태 종류가 늘거나 미완료 Payment 를 저장하게 되는 순간 이름이 거짓이 된다. status 조건을 본문에 명시하면, 그 불변식이 깨져도 이름이 표현하는 의미를 코드가 실제로 보장한다.

## 사실 조회 vs 정책 적용 — SRP 를 변화의 축 단위로

`row 존재 → cancel skip` 한 줄에 두 책임이 섞여 있었다.

- **사실 조회**: 완료된 결제 row 가 존재하는가 — Payment 도메인의 지식.
- **정책 적용**: 그러니까 보상 취소를 skip 한다 — 보상 흐름의 결정.

이 둘을 다른 위치(사실은 도메인 서비스, 정책은 보상 서비스의 `if` 한 줄)에 두면 각 책임의 변화가 각자의 자리에 갇힌다. 결제 완료의 정의가 바뀌면 도메인 쪽만, 보상 정책이 바뀌면 보상 서비스 쪽만 고친다. 단일 책임 원칙을 "메서드 하나" 단위가 아니라 "변화의 축" 단위로 적용한 사례다. "사실만 조회하고 분류/정책 계산은 분리한다"는 결은 [[payment-status-사실만-분류는-정책계산-manual-review-철회]]와 같은 원칙이다.

## 트레이드오프·리스크

- 보상 스킵을 별도 정책 메서드 없이 호출부 `if` 한 줄로 두므로, 같은 판단을 여러 곳에서 쓰게 되면 정책이 흩어질 수 있다. 현재는 한 곳뿐이라 수용.
- `hasCompletedPayment`의 완료 판단은 이 시점의 `status = COMPLETED` 모델에 묶여 있다.

> [!warning] 시간 순서 — 완료 상태 모델의 이후 변화
> 이 결정(2026-06-02)은 Payment 가 완료 시 `status = COMPLETED` 로 저장되던 옛 모델을 전제한다. 이후 결제 도메인 재설계에서 완료 판단 모델이 append-only 원장의 row 존재 기준으로 바뀌면서 `COMPLETED` 상태 자체가 사라지는 방향으로 진화했다 — [[payment-append-only-원장과-exists-완료판단]] 참조. 따라서 여기 서술한 `existsByMerchantPayKeyAndStatus(..., COMPLETED)`의 구체 형태는 이후 재설계 노트가 정본이며, 이 노트는 그 이전 스냅샷으로 읽어야 한다. merchantPayKey unique 를 근거로 삼은 부분은 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]와, 보상 흐름의 이후 재편은 [[payment-order-facade-결합끊기-tell-dont-ask]]·[[보상판단-payment-존재-lock-대신-db-unique]]로 이어진다.

## 미해결·후속

- **Payment 도메인이 실제로 분리될 때 `hasCompletedPayment`의 외부 API 승격을 다시 검증해야 한다.** 지금은 같은 서비스 안에서 부르지만, 도메인 경계를 실제로 그으면 이게 도메인 간 조회 API 가 되므로 그 시점에 계약을 다시 봐야 한다. (도메인 분리 자체는 [[payment-order-도메인분리와-pg격리]] 참조.)
- **이 결정의 후속 노트가 임계(대략 3~4개)에 이르면 별도 결정 문서로 졸업시킨다.** 한 결정에 후속 노트가 계속 쌓이면 원 결정과 진화가 뒤엉키므로, 임계에 닿으면 분리하겠다는 예고. 언제·어떻게 졸업시킬지는 아직 미확정. (이 운영 판단은 [[backend-완료된-task-문서-불변-원칙]]의 "후속 노트 임계 졸업"과 같은 논의다.)

## 근거

- [[raw/sessions/backend/2026-06-02-pr-192-payment-completion-fact-query]]
