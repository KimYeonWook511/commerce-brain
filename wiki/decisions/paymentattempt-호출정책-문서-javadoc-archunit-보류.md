---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, archunit, convention, javadoc, architecture-test]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# PaymentAttempt 호출 정책 — ArchUnit 대신 문서·JavaDoc·단일 호출처

## 컨텍스트·문제(public 메서드 호출 강제 불가)

`PaymentAttempt.succeed`/`fail`은 `PaymentApprovalAttemptService`/`PaymentCancellationAttemptService` **외부에서 직접 호출하지 않는다**는 정책이 있다([[paymentattempt-상태전이-도메인-검증-defensive]]의 상태 가드가 단일 호출처를 전제하기 때문). 그런데 Java `public` 접근자라 컴파일러로는 이 호출처 제약을 강제할 수 없다. 어떻게 정책을 강제할 것인가.

## 후보 비교(ArchUnit / package-private / 문서+JavaDoc)

- **ArchUnit CI 차단** — 가장 강한 보장(CI에서 위반 경로 차단). 별도 학습 + 룰 설계 + CI 통합 비용 발생.
- **패키지 구조 변경(package-private)** — `PaymentAttempt`와 `*AttemptService`를 같은 패키지로 이동. 대규모 변경 + 기존 패키지 분리 의도 훼손 → 제외.
- **문서 + JavaDoc + 단일 호출처 유지 ✓** — 현재 위반 경로 없음. 새 기여자 실수만 방지하면 충분. 채택 비용 0.

## ArchUnit을 지금 안 넣은 이유(학습 단계 우선순위)

결정적 이유는 **학습 단계의 우선순위**다. 현 시점은 개발 과정 자체가 주이고, 그 과정의 고민을 문서로 녹이는 게 부가가치인 단계다. ArchUnit은 학습·룰 설계·CI 통합 시간이 드는데 그 비용을 가늠하기 어렵다. 반면 문서 + JavaDoc은 *개발하면서 바로 기록 가능*해 채택 비용 0, 즉시 적용된다. "학습 단계라 도구를 더 늘리고 싶지 않음" + "개발 흐름을 끊지 않고 기록 가능한 방법 우선"의 결합이다 — 학습 단계라 새 도구/추상화를 늘리지 않는 판단은 [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]의 PgCanceller AI 추천 시도, [[redis-장애-strict-정책-soft-fail-기각]] 등과 같은 결이다.

## 미해결·후속(다른 도메인과 일관 도입)

- **향후 도입 시점** — 다른 도메인의 아키텍처 테스트와 함께 일관되게 도입 권장. **단일 도메인 단독 도입은 룰 운영 부담 대비 가치가 낮다.** 운영 진입 + 안정화 단계에 검토.
- ArchUnit 도입은 payment 여러 회고가 동일하게 제안한 미해결 클러스터다([[payment-도메인-구조-개요]]).

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
