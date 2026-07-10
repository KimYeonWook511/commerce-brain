---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 192 }
---

## 한 일

- 이슈 #182 (PR #192) — `PaymentApprovalService.isCompensationRequired` → `hasCompletedPayment` 로 이름과 의미 정리
- 보상 정책 적용을 `PaymentApprovalCompensationService.runPgCancel` 호출 코드 한 줄(`if (hasCompletedPayment) skip`)로 분리. 별도 정책 메서드 만들지 않음
- 구현은 `existsByMerchantPayKeyAndStatus(merchantPayKey, COMPLETED)` 로 status 명시 exists 쿼리
- 완료된 tasks 폴더 문서 불변 원칙을 `docs/tasks/README.md` 와 `CLAUDE.md` 에 명문화
- ADR-014 / ADR-015 본문 정리 + 후속 (#182) 노트 추가, `docs/exception-strategy.md` 적용 예 갱신

## 결정한 것

### 보상 진행 여부 판단의 위치 vs 이름

- **A) `isCompensationRequired` (public) 유지**: 도메인 소유, API 승격 경로 보존. 이름 컨텍스트 누출
- **B) `PaymentApprovalCompensationService` private 이동** (이슈 제안): 응집, 캡슐화. 도메인 정책이 보상 service 에 박힘
- **C) `hasCompletedPayment` 로 이름 변경 + 위치 유지**: 사실 조회는 도메인, 정책 적용은 보상 service if 한 줄

claude 가 처음부터 C 를 약한 근거(이름의 컨텍스트 누출 정도)로 추천. 외부 claude 의견 — "`.isPresent()` 한 줄이 단순 Optional 문법이 아니라 'row 존재 = 결제 완료' 도메인 단정. 단정의 근거(merchantPayKey unique + Order FOR UPDATE)가 Payment 쪽이므로 단정을 소유자에 박아야 정의 변경 시 한 곳만 갱신" — 으로 C 의 근거가 단단해져 최종 C 채택. 정책 응집은 보상 service if 한 줄로 유지 (별도 메서드 X)

### exists 쿼리 + status 명시 (gemini review)

review 는 마이크로 최적화(`existsByMerchantPayKey`) 만 제안. 사용자가 "메서드 이름은 hasCompletedPayment 인데 status 안 보면 의미 어긋남" 지적. 최종 `existsByMerchantPayKeyAndStatus(..., COMPLETED)` — 효율 + 의미 일치 동시 확보

### task 폴더 운영 정책 — 완료된 tasks 불변

배경: 이슈가 두 task architecture.md (`payment-compensation-to-domain`, `payment-attempt-service-split`) 갱신을 명시했지만, "완료된 task 문서를 사후 수정하는 게 맞나" 의문이 들면서 원칙 논의 촉발. 처음엔 갱신했다가 원칙 정리 후 되돌림.

정리된 정책 5개 항목(불변 / 진행 중 자유 수정 / 후속 노트 표현 / 임계 시 새 ADR 졸업 / 새 정책 문서 최소화)은 `docs/tasks/README.md` "완료된 tasks 불변 원칙" 섹션에 정본화. `CLAUDE.md` 안전 규칙 한 줄로 행동 지침

## 배운 것

### 메서드 이름과 본문 의미의 일치

`hasCompletedPayment` 가 처음 `findByMerchantPayKey(...).isPresent()` 로 구현됐을 때, 이름은 "완료된 Payment 존재" 인데 본문은 "그냥 Payment row 존재". 우연히 동작한 이유는 "Payment 는 COMPLETED 상태로만 저장" 이라는 invariant 덕분. 그 invariant 는 코드에 강제 안 됨. 이름이 표현하는 의미를 본문에서 명시적으로 보장(`status = COMPLETED`)하면 안전망 실제 작동

### "사실 조회" vs "정책 적용" 분리 — SRP 를 변화의 축 단위로

같은 코드 한 줄(`row 존재 → cancel skip`) 안에 두 책임이 섞임:
- 사실 조회: row 존재 — Payment 도메인
- 정책 적용: 그러므로 skip — 보상 흐름

다른 위치에 두면 각 책임의 변화가 한 곳에 가둠. SRP 를 메서드 단위가 아니라 "변화의 축" 단위로 적용

## 다음 단계

- Payment 도메인 분리 시 `hasCompletedPayment` 외부 API 승격 검증
- ADR-014 후속 노트 임계(3~4개) 도달 시 새 ADR 졸업

## 외부 인용

- 정본: `commerce-backend/docs/ADR.md#ADR-014`, `#ADR-015`
- 살아있는 정책: `commerce-backend/docs/exception-strategy.md`
- 운영 가이드: `commerce-backend/docs/tasks/README.md` "완료된 tasks 불변 원칙"
- PR: #192, Closes #182
