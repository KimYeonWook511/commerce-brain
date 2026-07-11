---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 183 }
---

# 결제 보상 판단의 REQUIRES_NEW 트랜잭션 격리 제거 — 근거 사라진 방어벨트와 잘못 박힌 정책 문구 정정

결제 승인 실패 후 PG 보상 취소를 진행할지 판단하는 메서드(`PaymentApprovalService.isCompensationRequired`)에 걸려 있던 `@Transactional(readOnly = true, propagation = REQUIRES_NEW)` 중 `REQUIRES_NEW` 격리를 제거한 세션이다(이슈 #160, PR #183). 이 격리는 "외부 트랜잭션의 1차 캐시에 오염되지 않고 커밋된 DB 상태만 읽어 race-safe하게 판단한다"는 명분으로 붙어 있었는데, 실제 호출 경로에 외부 트랜잭션이 존재하지 않아 격리할 대상 자체가 없음을 확인했다. 더 문제였던 건 이 잘못된 격리 명분이 코드 주석뿐 아니라 여러 정책 문서에 "핵심 격리"로 박혀 잘못된 전제를 재생산하고 있었다는 점이라, 코드와 문서를 함께 정정했다.

## 결정한 것

### 1. `REQUIRES_NEW` 격리 제거 — 격리할 외부 트랜잭션이 애초에 없다

`isCompensationRequired`의 어노테이션을 `@Transactional(readOnly = true, propagation = REQUIRES_NEW)`에서 `@Transactional(readOnly = true)`로 낮췄다.

- **이 메서드의 역할:** 결제 승인이 실패해 PG 보상 취소를 진행하기 직전, "완료된 Payment(결제) row가 이미 존재하는가"를 조회해(`findPaymentByMerchantPayKey(...).isEmpty()`) 보상이 필요한지 판단한다. false면 이미 완료된 결제가 있으니 취소를 skip한다.
- **제거가 안전한 근거 — 외부 트랜잭션이 없다:** 이 메서드를 부르는 실제 경로는 `NaverPayApprovalService.completeVerifiedApproval`의 catch 블록 → `PaymentApprovalCompensationService.runPgCancel` → 그 안에서 `isCompensationRequired` 호출이다. 이 경로의 어느 클래스·메서드에도 `@Transactional`이 붙어 있지 않다(두 서비스 클래스 모두 `@Transactional` 어노테이션 0건, 확인함). 외부 트랜잭션이 없으니 오염될 1차 캐시 자체가 없고, `REQUIRES_NEW`가 지킬 격리 대상이 존재하지 않았다. `readOnly = true`만으로도 자기 트랜잭션에서 커밋된 상태를 읽으므로 race-safe 판단은 그대로 성립한다.
- **남아 있던 건 비용뿐:** 격리 이득이 공집합인 상태에서 `REQUIRES_NEW`는 순전히 비용만 남았다 — 이 전파 모드는 진행 중 트랜잭션을 suspend하고 별도 커넥션을 여는 성격이라, 나중에 상위에 트랜잭션이 끼는 순간 커넥션을 이중 점유해 풀 포화로 이어질 수 있는 잠재 위험만 남긴다. (지금은 외부 TX가 없어 당장 이중 점유가 나는 건 아니지만, 이득 없는 격리를 남겨 두면 미래에 상위 TX가 생겼을 때 잠복 위험이 된다.)

### 2. 여러 정책 문서에 박힌 "REQUIRES_NEW 격리 보존" 문구를 "단계별 독립 commit(부분 진행 보존)"으로 정정

코드보다 더 컸던 문제는, 사라진 방어벨트의 명분이 정책 문서에 근거로 못 박혀 있어 잘못된 전제를 살려 두고 있었다는 점이다. 보상 서비스의 트랜잭션 정책을 규정한 결정들이 "클래스 레벨 `@Transactional`을 두지 않는 이유"를 "`isCompensationRequired`의 `REQUIRES_NEW` 격리를 보존하기 위해"라고 적어 두고 있었다.

- **실제 진짜 이유로 교체:** 보상 서비스에 클래스 레벨 `@Transactional`을 두지 않는 진짜 근거는 격리가 아니라 **단계별 독립 commit**이다. 보상 흐름의 각 단계(`failIfRequested`, `isCompensationRequired`, `getOrCreate`, `succeed`/`fail`)가 각자 자기 트랜잭션을 열어 독립 커밋해야, 한 단계가 실패해도 앞서 진행한 부분(예: `failIfRequested`로 상태 전이된 approve attempt, `getOrCreate`로 생성된 cancel attempt)이 보존된다. 단일 트랜잭션으로 묶으면 한 단계 실패가 이전 단계 진행까지 롤백시켜 부분 진행 보존이 불가능해진다. race-safe 판단은 이제 `readOnly = true`가 커밋된 상태를 읽는 것으로 별도 성립하고, 트랜잭션 정책의 근거에서 분리했다.
- **정정 대상:** 보상 정책의 소유·트랜잭션 정책을 규정한 정본 결정 문서와, 보상 도메인 이관 작업(payment-compensation-to-domain)의 관련 문서들(트랜잭션 정책 ADR·PRD·아키텍처), 그리고 payment-attempt-service-split 작업의 아키텍처 문서를 갱신했다. 회고·step 문서는 **작업 당시의 기록(스냅샷)**이라 사후 수정하지 않고 그대로 뒀다.
- **주석도 정정:** 코드 주석 역시 "외부 트랜잭션 1차 캐시 오염 방지를 위해 REQUIRES_NEW로 격리한다"에서 "PG 보상 취소 처리 시작 전 커밋된 DB 상태를 기준으로 보상 필요 여부를 판단한다"로 바꿨다.

### 3. 판단 메서드의 위치 이동은 검토 후 이번 PR에서 보류 — 후속 이슈로 분리

`isCompensationRequired`를 호출 위치인 보상 서비스(`PaymentApprovalCompensationService`)로 옮기는 리팩터링을 검토했으나 이번 PR에서는 보류하고 후속 이슈 #182로 분리했다.

- **검토한 이동안(택한 쪽):** 보상 서비스의 `private` 메서드로 옮기되, 내부에서는 결제 조회를 **다른 application service를 경유**해서 한다(`paymentApprovalService.findPaymentByMerchantPayKey(...).isEmpty()`). 이걸 택한 결정적 근거는 보상 서비스가 이미 자신의 다섯 단계를 **전부 다른 application service를 거쳐** 수행하는 "정책 조립자" 패턴을 갖고 있다는 것이다 — 새 메서드도 그 패턴을 따라야 일관된다.
- **기각한 이동안:** 같은 메서드를 옮기되 Repository에 직접 의존시키는 안. 응집도·SRP만 보면 더 깔끔해 보이지만, 다섯 단계 중 이 하나만 다른 의존 방식(repository 직접)이 되어 "모든 도메인 조작을 service 경유로" 하던 패턴을 깬다. 그래서 기각했다.
- **왜 이번엔 보류했나 — 테스트가 막았다:** 동시성 통합 테스트(`NaverPayServiceConcurrencyTest`)의 mock 패턴이 위치 이동과 충돌한다. 현재 테스트는 단일 mock으로 `isCompensationRequired`가 **특정 시점에만** false를 반환하도록 제어해 race window를 재현한다. 이동 후 mock 대상이 `findPaymentByMerchantPayKey`가 되면, 이 메서드는 `approve()` 시작 시점에도 보상 시점에도 똑같이 호출되므로 두 호출을 시점으로 구분할 수 없게 된다. 멀티스레드 환경에서 chained `willReturn`이나 `AtomicReference` + `doAnswer`로 시점을 구분하려는 대안은 모두 fragile하다. 테스트 재설계 비용까지 포함해 후속 이슈 #182로 넘겼다.

## 막힌 점·해결

### 위치 이동 판단이 두 번 흔들렸다 — 응집도만 보다 패턴 일관성을 놓침

- **처음:** service 경유안을 "변경 영향이 적어서"라는 약한 근거로 택했다.
- **그다음:** repository 직접안으로 뒤집었다. 응집도/SRP만 보고 보상 서비스가 모든 단계를 "service 경유"로 하는 패턴 일관성을 놓쳤다.
- **최종:** 사용자의 지적("다른 단계는 다 service를 경유하는데 이것만 repository 직접 의존이냐?")으로 다시 service 경유안으로 정정했다. 결정 근거가 "변경 영향" 같은 약한 것에서 "구조 패턴 일관성"이라는 재사용 가능한 기준으로 바뀌었다.
- **안정화 계기:** 사용자가 "휘둘리지 말고 네 판단으로만 답하라"고 지시한 뒤에야 결론이 안정됐다. 근거의 강도로 판단을 고정하는 게 흔들림을 막았다.

## 배운 것

- **방어적 안전벨트가 후속 리팩터링으로 의미를 잃어도, 정책 문서까지 동기화되지 않으면 잘못된 전제가 살아남는다.** 여기서 `REQUIRES_NEW`의 격리 명분은 호출 경로에서 외부 트랜잭션이 사라지면서 무의미해졌지만, 문서에는 "핵심 격리"로 남아 잘못된 근거를 재생산하고 있었다. 코드만 보지 말고 정책 표현의 근거가 여전히 유효한지 확인해야 한다.
- **멀티스레드 mock chaining은 fragile하다.** 호출 시점에 의존해 반환값을 바꾸는 mock 패턴(같은 메서드가 여러 시점에 호출되는데 특정 시점만 다르게)은 단위 테스트에서나 통제된다. 동시성 통합 테스트는 진짜 race window 시나리오로 모델링하거나, 아예 단위 테스트로 분해해야 한다.
- **헥사고날 application service들의 의존 패턴 일관성**이 재사용 가능한 구조적 결정 기준이다. 보상 서비스처럼 "정책 조립자"가 모든 도메인 조작을 다른 service 경유로 하는 패턴이면, 새 메서드를 추가할 때도 그 패턴을 따라야 한다.
- **Spring Data JPA의 `SimpleJpaRepository`는 클래스 레벨 `@Transactional(readOnly = true)`를 기본 적용**한다. 즉 Repository 호출 자체가 자기 트랜잭션을 시작한다. self-invocation 한계를 우회할 때 이 성질을 활용할 수 있다.
- **PR 리뷰의 "트랜잭션" 용어 모호성:** "보상 트랜잭션"처럼 비즈니스 수준의 보상 처리를 가리키는 표현이 `@Transactional` DB 트랜잭션과 같은 단어를 쓰면 혼동을 부른다. 이번 gemini 리뷰 1건도 이 용어 명확화 지적이었고, "PG 보상 취소 처리"처럼 명시적으로 쓰기로 했다.

## 미해결·열린 질문

- **판단 메서드 위치 이동(후속 이슈 #182):** `isCompensationRequired`를 보상 서비스의 `private` 메서드(service 경유안)로 옮기는 작업. 동시성 테스트 재설계가 함께 필요하다. #160(이 PR) 머지 후 진행 예정.
- **약한 근거 문구 정리:** 보상 필요 여부를 Payment(결제) 존재 여부로 판단한다는 결정에는 "미래에 Payment 도메인을 분리하면 이 판단 메서드가 외부 API로 자연스럽게 승격 가능하다"는 취지의 정당화가 붙어 있는데, 이는 약한 근거로 판단된다. #182 진행 시 이 표현도 함께 정리할지 검토한다.
