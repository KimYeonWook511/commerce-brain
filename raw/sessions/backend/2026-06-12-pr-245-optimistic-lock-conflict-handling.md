---
platform: backend
author: KimYeonWook511
created: 2026-06-12
origin:
  - { type: pr, repo: commerce-backend, ref: 245 }
---

# Payment 낙관 락 충돌 처리 — 구현 실패와 올바른 구조 확정

## 배경
#243(Payment 엔티티에 @Version 낙관 락 도입) 코드를 harness로 구현(version 컬럼 추가 / 종착 전이 충돌 흡수 / escalation을 도메인 메서드로)하고 PR #245 오픈. PR 리뷰 단계에서 흡수 구현이 잘못됐음을 발견. 선행 메모 [[raw/sessions/backend/2026-06-11-why-payment-optimistic-lock]]는 낙관 락 채택까지만 정했고 "충돌 흡수를 어떻게 하나"는 미정이었다 — 이 메모가 그 흡수 구조를 처음 확정한다.

## 막힌 점 — 흡수 구현이 세 정책 위반
구현된 방식: `markUnknownIfRequested`(REQUESTED일 때만 UNKNOWN 마킹, 아니면 skip)/`failIfPending`(보상 흐름에서 미확정이면 FAILED, 아니면 skip) — 둘 다 보상·best-effort 전이 메서드 — 에서 application 서비스가 `ObjectOptimisticLockingFailureException`(Spring DAO 예외)을 직접 catch해서 skip하고, `@Transactional`을 제거함.

세 위반:
1. application이 Spring DAO 예외(`org.springframework.orm.*`)를 직접 catch — 코드베이스 정책은 "인프라 예외는 adapter에서 도메인 예외로 변환, application/domain은 DAO 타입에 의존 안 함"(정본 docs/exception-strategy.md). 같은 Payment 도메인 선례 `PaymentReservationRepositoryAdapter.saveUsed`(예약 소비 전용 저장 경로 — @Version 충돌을 adapter에서 PaymentException으로 변환 throw)와 어긋남.
2. `@Transactional` 제거 — 보상 흐름은 "각 단계가 자기 @Transactional로 독립 commit"이 정책(정본 docs/tasks/payment-compensation-to-domain/adr.md). 제거하면 단계별 독립 commit이 깨져 한 단계 실패가 이전 단계를 롤백시킴.
3. 흡수(skip)를 트랜잭션 안에서 함 — @Version 충돌은 flush 시점 발생 → 그 순간 트랜잭션은 rollback-only로 마킹됨 → 같은 메서드 안에서 catch하고 정상 리턴하면 commit 시점에 UnexpectedRollbackException. `saveUsed` 주석이 이미 "충돌 후 tx는 rollback-only"라 인지하고 변환 throw만 한다(흡수 안 함).

코드 리뷰어(Gemini)는 2번만 부분 지적(@Transactional 대신 REQUIRES_NEW 제안). 그러나 REQUIRES_NEW를 메서드에 붙여도 같은 메서드 안에서 catch하면 그 새 트랜잭션이 rollback-only가 되어 UnexpectedRollbackException이 여전하다. 검증을 시도했으나 동시성 테스트가 타이밍 의존이라 흡수 경로(충돌 진 쪽)를 결정적으로 안 타서 "통과"로 위장됐다(검증 불충분).

## 결정한 것 — 올바른 구조
외부에서 작성된 설계 문서(payment-optimistic-lock-design.md — 아직 외부 초안, repo 미커밋. worktree로 옮길 예정)로 확정. 한 줄 요지: "문제는 흡수한 게 아니라 흡수를 트랜잭션 안에서 한 것."

- transition (별도 빈, public @Transactional): find + 도메인 전이 + saveChecked 호출. 충돌도 가드 위반도 catch 안 함 → 도메인 예외 전파 → 트랜잭션 깨끗이 rollback.
- adapter saveChecked (신규): saveAndFlush로 flush를 adapter 프레임 안으로 당겨와 ObjectOptimisticLockingFailureException을 PaymentException(PAYMENT_CONCURRENTLY_MODIFIED, 신규 충돌 전용 코드)으로 변환 throw.
- useCase(=orchestrator: 보상·실시간 승인·대사 흐름) (트랜잭션 없음): transition을 호출하고, skip이 필요하면 useCase의 private 래퍼 메서드에서 도메인 예외를 catch해 skip. catch가 트랜잭션 경계 밖이라 rollback-only와 무관.
- 함정 2개: transition은 반드시 useCase와 별도 빈의 public 메서드여야 한다(private이면 @Transactional 무효, 같은 빈 self-call이면 프록시 우회로 무효). useCase에는 @Transactional을 달지 않는다(달면 흡수가 다시 트랜잭션 안으로 들어가 원래 문제로 회귀).
- saveAndFlush의 의미: "실패할 save를 성공시키는 도구"가 아니라 "충돌을 잡을 수 있는 위치(adapter 프레임)로 flush를 당겨오는 도구". 선례 saveUsed/saveApproved가 saveAndFlush인 이유가 이것.

### 예외 코드 granularity (의미 코드 vs 일반 코드)
- 충돌은 일반 코드(PAYMENT_CONCURRENTLY_MODIFIED = "다른 처리가 먼저 상태를 바꿈")로 던지고, "이미 무엇이 됐는지"가 필요하면 재조회로 판정. unique 위반(PAYMENT_DUPLICATE — 주문당 SUCCEEDED 중복을 막는 unique 위반 매핑 코드)과 version 충돌은 정책이 달라(중복→보상 vs 재시도/skip) 절대 한 코드로 합치지 않음.
- reservation의 PAYMENT_RESERVATION_ALREADY_USED 같은 의미 코드는 "version 충돌 = 이미 사용됨"이 1:1로 성립하는 전제 위에서만 정직하다. 같은 행에 다른 동시 쓰기 경로가 하나만 늘어도 거짓 양성이 되고 컴파일로 안 잡힌다. 새로 짜는 전이는 일반 코드 + 재조회.

### 낙관 락 vs 조건부 UPDATE (트레이드오프)
- 둘은 어느 게 기본이 아니라 전이 성격으로 고른다. 낙관 락+catch = 전이가 도메인 로직을 품을 때(이 코드베이스 주력). 조건부 UPDATE(WHERE 가드 + InnoDB 행 락) = 단순 멱등 status 플립일 때(충돌이 예외 대신 affected rows=0, 트랜잭션 딜레마가 소멸, 단 전이 로직이 SQL로 내려가 DDD 냄새).
- 이번엔 낙관 락 주력으로 결정. escalation도 도메인 메서드(escalate()) 유지(조건부 UPDATE로 되돌리지 않음). 여러 애그리거트를 한 트랜잭션에 묶는 승인 확정(succeedApproval: payment 전이 + order 완료)만 낙관/비관 락 혼용.

## 다음 단계
- 코드 구현(transition 재구성 + useCase private 래퍼 + escalation + 결정적 충돌 테스트)은 별도 집중 세션으로 분리(이 세션이 너무 길어짐). 기반 변경(PAYMENT_CONCURRENTLY_MODIFIED 에러코드 + adapter saveChecked)은 이미 worktree에 반영.
- 루트 문서 보강은 별도: docs/exception-strategy.md에 "낙관 락 충돌 처리" 섹션을 정본으로 신설(메커니즘 / 처리 위치 / 충돌 정책 3종 전파·skip·retry / 코드 granularity / 조건부 UPDATE 대안). 전체 architecture 재정비는 또 다른 별도 PR.
- 미결: PAYMENT_STATUS_TRANSITION_NOT_ALLOWED(가드 위반 시 던지는 코드, 현재 HTTP 500)가 가드 위반이 skip 대상이 되는 경로에서는 부적절할 수 있어 재검토 필요.
- 추적: 이슈 #243, PR #245. AI 협업 교훈은 같은 PR의 [[raw/sessions/backend/2026-06-12-pr-245-design-late-context-fatigue-ai]].
