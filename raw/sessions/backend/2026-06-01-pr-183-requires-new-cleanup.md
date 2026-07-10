---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 183 }
---

## 한 일

- 이슈 #160 (PaymentApprovalService.isCompensationRequired REQUIRES_NEW 제거) 작업
- 잘못된 트랜잭션 격리 정책이 ADR-015 / ADR-T2 / task docs 에 박혀있던 것을 발견하고 정리
- 위치 이동 (isCompensationRequired → PaymentApprovalCompensationService) 검토 후 보류, 후속 이슈 #182 로 분리
- PR #183 생성 → gemini review 1건 (주석 용어 명확화) accept 후 머지 대기

## 결정한 것

### REQUIRES_NEW 제거가 안전한 근거

호출 경로(`NaverPayApprovalService.completeVerifiedApproval` catch → `PaymentApprovalCompensationService.runPgCancel`) 모두 `@Transactional` 없음. 외부 TX 가 존재하지 않으므로 격리할 1차 캐시 자체가 없음. `REQUIRES_NEW` 가 커넥션 2개를 점유해 풀 포화 위험만 남아 있던 상태.

### 정책 표현 정정 — "REQUIRES_NEW 격리 보존" → "단계별 독립 commit (부분 진행 보존)"

`docs/ADR.md` ADR-015, `payment-compensation-to-domain/{adr,prd,architecture}.md`, `payment-attempt-service-split/architecture.md` 갱신. 회고/step 문서는 작업 당시 기록이라 사후 수정 안 함.

### 위치 이동 (B-2) 결정 + 보류

검토 결과: `PaymentApprovalCompensationService` 의 `private` 메서드로 옮기되, 내부에서 `paymentApprovalService.findPaymentByMerchantPayKey(...).isEmpty()` 호출 (B-2).

- Compensation 의 "다른 application service 경유" 패턴(5단계 모두 service 경유) 일관성 유지가 결정적 근거
- (B-1) Repository 직접 의존은 패턴 깸 — 5개 단계 중 1개만 다른 의존 방식

다만 이번 PR 에서 보류. `NaverPayServiceConcurrencyTest` 의 mock 패턴이 위치 이동과 충돌:
- 단일 mock 으로 `isCompensationRequired` 특정 시점만 false 처리하는 현재 패턴
- 위치 이동 후 mock target 이 `findPaymentByMerchantPayKey` 가 되면 `approve()` 시작 시점 / 보상 시점 같은 mock 호출 → 시점 구분 불가
- multi-thread 환경에서 chained `willReturn` / `AtomicReference` + `doAnswer` 모두 fragile

후속 이슈 #182 로 분리 (테스트 재설계 비용까지 포함).

## 막힌 점

- 위치 결정 과정에서 두 번 흔들림
  - 처음 (B-2) → "변경 영향 적어서" 라는 약한 근거
  - 그 다음 (B-1) → 응집도/SRP 만 보고 Compensation 의 "service 경유" 패턴 일관성 놓침
  - 최종 (B-2) → 사용자 지적("다른 단계는 다 service 경유인데 이거만 repository 직접?") 으로 정정
- 사용자가 "휘둘리지 말고 너 판단으로만 답해" 라고 지시한 후에야 안정적 결론

## 배운 것

- **방어적 안전벨트가 후속 리팩터링으로 의미를 잃어도 정책 문서까지 동기화되지 않으면 잘못된 전제가 살아남는다.** 코드만 보지 말고 정책 표현의 근거가 여전히 유효한지 확인 필요
- **multi-thread mock chaining 은 fragile.** 호출 시점에 의존하는 mock 패턴은 단위 테스트에만 적용 가능. 동시성 통합 테스트는 진짜 race window 시나리오로 모델링하든지 단위 테스트로 분해해야 함
- **헥사고날 application service 들의 의존 패턴 일관성**이 재사용 가능한 구조적 결정 기준. Compensation 처럼 "정책 조립자" 가 모든 도메인 조작을 다른 service 경유로 하는 패턴이면 새 메서드 추가 시도 그 패턴을 따라야 함
- **Spring Data JPA `SimpleJpaRepository` 가 클래스 레벨 `@Transactional(readOnly = true)` 를 기본 적용**하므로 Repository 호출 자체가 자기 TX 를 시작. self-invocation 한계 우회에 활용 가능
- **PR review 의 "트랜잭션" 용어 모호성**: "보상 트랜잭션" 같이 비즈니스 수준 보상 처리를 가리키는 표현이 `@Transactional` DB 트랜잭션과 같은 단어를 쓰면 혼동. "PG 보상 취소 처리" 같이 명확화 권장

## 다음 단계

- 후속 이슈 #182 — `isCompensationRequired` 위치 이동 (Compensation private). 동시성 테스트 재설계 동반 필요. #160 머지 후 진행
- ADR-014 의 "외부 API 자연 승격" 표현은 약한 근거 — #182 진행 시 함께 표현 정리 검토

## 참고

- 이슈: #160
- PR: #183
- 후속 이슈: #182
- 정본 ADR: `commerce-backend/docs/ADR.md` (ADR-014, ADR-015), `commerce-backend/docs/tasks/payment-compensation-to-domain/adr.md` (ADR-T2)
