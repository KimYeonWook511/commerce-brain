---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 192 }
---

# 보상 스킵 판단을 Payment 도메인 사실 조회로 정리 — 이름·의미·status를 일치시킴

결제 승인이 실패해 PG 취소(보상)로 넘어갈 때, "이미 결제가 완료됐으면 취소를 건너뛴다"는 가드가 있다. 이 가드가 `PaymentApprovalService.isCompensationRequired(merchantPayKey)`(보상 필요 여부 판단 메서드)로 표현돼 있었는데, 실제 하는 일은 "완료된 Payment row 가 있느냐"였다. 이슈 #182(PR #192)에서 그 이름·위치·구현을 도메인 사실 조회로 정리하고, 겸사겸사 완료된 task 폴더 문서를 사후 수정하는 게 맞는지에 대한 운영 원칙까지 정리한 세션이다.

## 결정한 것

### 1. 보상 스킵 판단의 위치는 유지, 이름만 사실 조회로 — `hasCompletedPayment`

보상 흐름(승인 실패 후 PG 취소를 진행하는 `PaymentApprovalCompensationService.runPgCancel`)이 취소 전에 "결제가 이미 완료됐는가"를 확인하는 부분을, `isCompensationRequired`에서 `hasCompletedPayment`로 바꿨다. 세 가지 배치를 저울질했다.

- **A) `isCompensationRequired`를 public 도메인 메서드로 그대로 유지** — 판단을 Payment 도메인이 소유하고, 나중에 Payment 도메인이 별도로 분리될 때 외부 API로 승격되는 경로가 보존된다. 다만 이름에 "보상(compensation)"이라는 보상 흐름 쪽 컨텍스트가 누출된다 — 사실 조회 메서드가 자기를 부르는 쪽의 용도를 이름에 담고 있는 꼴.
- **B) 판단을 보상 서비스(`PaymentApprovalCompensationService`)의 private 메서드로 이동** — 이슈가 제안한 방향. 보상 로직끼리 응집되고 캡슐화된다. 그러나 "결제 완료 여부"라는 도메인 사실 판단이 보상 서비스 안에 박혀, 판단의 근거가 바뀌면 엉뚱한 곳을 고치게 된다.
- **C) 이름을 `hasCompletedPayment`로 바꾸되 위치는 Payment 도메인 서비스에 유지** — 사실 조회(결제 완료됐나)는 도메인이 소유하고, 정책 적용(그러니 취소를 skip)은 보상 서비스가 `if (hasCompletedPayment) skip` 한 줄로 담당. 별도 정책 메서드는 만들지 않았다.

처음엔 C를 약한 근거(A의 이름 컨텍스트 누출 정도)로 골랐는데, 추가 리뷰 관점에서 근거가 단단해졌다. `.isPresent()`/exists 한 줄이 단순한 Optional 문법이 아니라 **"row 존재 = 결제 완료"라는 도메인 단정**이라는 것. 그 단정의 근거(merchantPayKey unique 제약 + Order 를 FOR UPDATE 로 잠근 안에서 저장하므로 동시성 안전)가 Payment 쪽에 있으니, 단정도 소유자(Payment 도메인)에 박아 둬야 정의가 바뀔 때 한 곳만 고친다. 이걸로 C를 최종 채택했다.

### 2. 구현은 status 명시 exists 쿼리로 — `existsByMerchantPayKeyAndStatus(..., COMPLETED)`

`hasCompletedPayment`의 초기 구현은 `findByMerchantPayKey(...).isPresent()`, 즉 그냥 row 존재 여부였다. AI 코드 리뷰(gemini)는 여기서 마이크로 최적화(`findBy...`로 엔티티를 끌어오지 말고 `existsBy...`로 존재만 확인)만 제안했다. 여기에 "메서드 이름은 `hasCompletedPayment`인데 status 를 안 보면 이름과 본문 의미가 어긋난다"는 지적을 더해, 최종적으로 `existsByMerchantPayKeyAndStatus(merchantPayKey, COMPLETED)`로 갔다. exists 로 효율을 얻으면서 status 조건까지 넣어 "완료된" 결제를 실제로 조회하게 만들어, 최적화와 의미 일치를 동시에 확보했다.

### 3. 완료된 task 폴더 문서는 불변 — 운영 원칙으로 정리

이 repo 는 작업을 task 폴더(`docs/tasks/<task-name>/`) 단위로 문서화한다. 이슈 #182 는 두 task 의 architecture 문서(`payment-compensation-to-domain`, `payment-attempt-service-split`)를 갱신하라고 명시했다. 처음엔 그대로 갱신했는데, **이미 머지가 끝난 task 폴더 문서를 사후에 수정하는 게 맞나**라는 의문이 들면서 운영 원칙 논의로 번졌다. 결국 갱신을 되돌리고 원칙부터 세웠다.

정리된 정책은 "task 폴더 문서는 PR 머지 시점부터 불변"을 뼈대로 한 다섯 갈래다.

- **완료된 task 폴더는 불변** — 머지된 task 의 문서는 그 시점 결정·의도의 기록으로 얼려 둔다.
- **진행 중 task 폴더만 자유 수정** — 단 "진행 중"은 현재 worktree 에서 본인이 작업 대상으로 삼은 그 task 하나로 한정. 리뷰 반영·설계 변경·단계 갱신 등 머지 전 변경은 본문에 반영 OK. 다른 worktree/브랜치의 진행 중 task 나 이미 머지된 task 는 손대지 않는다.
- **완료 후 변경은 루트 `docs/` 문서로 표현** — task 폴더가 아니라 살아있는 루트 정책 문서(예외 처리 전략 문서·아키텍처·API 스펙·DB 스키마)와 ADR 후속 노트로.
- **결정의 진화는 ADR 본문 갱신 또는 후속 노트** — task 폴더에 후속 노트를 흩뿌리지 않고, 결정 이력을 ADR 한 곳에서 추적할 수 있게 한다.
- **새 정책 문서는 최소화** — "결정 1개 = 문서 1개"가 아니라 "영역 1개 = 문서 1개". 기존 문서에 흡수하기 어려울 때만 새로 만든다.

이 원칙은 task 문서 운영 가이드(루트 `docs/tasks/README.md`)의 "완료된 tasks 불변 원칙" 섹션에 정본화하고, repo 루트 CLAUDE.md 에는 행동 지침 한 줄("머지된 task 폴더 문서는 이후 수정하지 않고, 이후 변경은 루트 docs 로만 표현")로 걸었다.

## 배운 것

### 메서드 이름과 본문 의미가 실제로 일치해야 안전망이 작동한다

`hasCompletedPayment`가 `findByMerchantPayKey(...).isPresent()`였을 때, 이름은 "완료된 Payment 존재"인데 본문은 "그냥 Payment row 존재"였다. 그런데도 우연히 맞게 동작한 건 "Payment 는 완료(COMPLETED) 상태로만 저장된다"는 불변식 덕분이다. 문제는 **그 불변식이 코드로 강제되지 않는다**는 것 — 상태 종류가 늘거나 미완료 Payment 를 저장하게 되는 순간 이름이 거짓이 된다. status 조건(`status = COMPLETED`)을 본문에 명시해 이름이 표현하는 의미를 코드가 실제로 보장하게 만들면, 그 불변식이 깨져도 안전망이 제대로 작동한다.

### "사실 조회" vs "정책 적용" 분리 — SRP 를 변화의 축 단위로

`row 존재 → cancel skip` 한 줄에 두 책임이 섞여 있었다.

- **사실 조회**: 완료된 결제 row 가 존재하는가 — Payment 도메인의 지식.
- **정책 적용**: 그러니까 보상 취소를 skip 한다 — 보상 흐름의 결정.

이 둘을 다른 위치(사실은 도메인 서비스, 정책은 보상 서비스의 if 한 줄)에 두면 각 책임의 변화가 각자의 자리에 갇힌다. 결제 완료의 정의가 바뀌면 도메인 쪽만, 보상 정책이 바뀌면 보상 서비스 쪽만 고친다. 단일 책임 원칙을 "메서드 하나" 단위가 아니라 "변화의 축" 단위로 적용한 사례다.

## 미해결·열린 질문

- **Payment 도메인이 실제로 분리될 때 `hasCompletedPayment`의 외부 API 승격을 다시 검증해야 한다.** 지금은 같은 서비스 안에서 부르지만, 도메인 경계를 실제로 그으면 이게 도메인 간 조회 API 가 되므로 그 시점에 계약을 다시 봐야 한다.
- **보상 존재 판단 결정(이 세션이 정리한 그 결정)의 후속 노트가 임계(대략 3~4개)에 이르면 별도 결정 문서로 졸업시킨다.** 한 결정에 후속 노트가 계속 쌓이면 원 결정과 진화가 뒤엉키므로, 임계에 닿으면 별도 결정 문서로 분리하겠다는 예고. (이건 원저자의 운영 판단 기준이고, 언제·어떻게 졸업시킬지는 아직 미확정.)
