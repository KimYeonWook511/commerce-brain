---
platform: backend
author: KimYeonWook511
created: 2026-06-12
origin:
  - { type: pr, repo: commerce-backend, ref: 245 }
---

# Payment 낙관 락 충돌 처리 — 흡수를 트랜잭션 밖으로 빼는 3계층 구조 확정

Payment 엔티티에 `@Version` 낙관 락을 도입(이슈 #243)해 같은 결제 행의 동시 상태 전이에서 생기는 lost update를 막기로 했다. 그 결정을 harness로 구현(version 컬럼 추가 / 종착 전이의 충돌 흡수 / escalation을 도메인 메서드로 환원)해 PR #245를 열었는데, 리뷰 단계에서 "충돌을 흡수하는" 구현이 잘못됐음을 발견했다. 낙관 락 채택 자체는 바로 앞 세션에서 정했지만 "충돌을 어떻게 흡수하나"는 미정으로 남겨뒀던 부분이라, 이 세션이 그 흡수 구조를 처음으로 확정한다.

## 막힌 점·해결 — 흡수 구현이 정책 3개를 위반

harness가 만든 방식은 이랬다. 보상·best-effort 성격의 종착 전이 메서드 두 개 — `markUnknownIfRequested`(REQUESTED일 때만 UNKNOWN 마킹, 아니면 skip)와 `failIfPending`(보상 흐름에서 미확정이면 FAILED, 아니면 skip) — 안에서 application 서비스가 Spring DAO 예외인 `ObjectOptimisticLockingFailureException`을 직접 catch해 skip하고, 그 메서드의 `@Transactional`을 제거했다. 세 가지를 한꺼번에 어겼다.

1. **application이 Spring DAO 예외를 직접 catch했다.** 코드베이스 예외 처리 정책은 "인프라 예외는 adapter에서 도메인 예외로 변환하고, application/domain은 DAO 타입에 의존하지 않는다"이다. 같은 Payment 도메인에 이미 선례가 있다 — 예약 소비 전용 저장 경로인 `PaymentReservationRepositoryAdapter.saveUsed`는 `@Version` 충돌을 **adapter 안에서** `PaymentException`(`PAYMENT_RESERVATION_ALREADY_USED`)으로 변환해 던진다. harness 구현은 이 선례와 정반대로 갔다.
2. **`@Transactional`을 제거했다.** 보상 흐름은 "각 단계가 자기 `@Transactional`로 독립 commit"이라는 게 정책이다(보상 흐름을 도메인으로 옮긴 task의 결정). 보상은 PG 응답과 정합을 단계별로 맞추는 작업이라, 단일 트랜잭션으로 묶이면 뒤 단계 실패가 앞 단계까지 롤백시켜 부분 진행 보존이 깨진다. `@Transactional`을 떼면 그 단계별 독립 commit이 무너진다.
3. **흡수(skip)를 트랜잭션 안에서 했다 — 이게 진짜 버그다.** `@Version` 충돌은 flush 시점에 터지고, 그 순간 그 트랜잭션은 rollback-only로 마킹된다. 같은 메서드 안에서 그 예외를 catch하고 정상 리턴하면, commit 시점에 `UnexpectedRollbackException`이 난다. 앞의 선례 `saveUsed`의 주석은 이미 "충돌 후 tx는 rollback-only"임을 알고 있어서 흡수하지 않고 **변환 throw만** 한다.

- **AI 코드 리뷰는 2번만 부분적으로 짚었다.** `@Transactional`을 떼는 대신 `REQUIRES_NEW`를 쓰라는 제안이었다. 그러나 메서드에 `REQUIRES_NEW`를 붙여도 **같은 메서드 안에서 catch하면** 그 새 트랜잭션이 rollback-only가 되어 `UnexpectedRollbackException`이 여전히 난다. 문제의 뿌리는 트랜잭션 전파 옵션이 아니라 "catch가 트랜잭션 경계 안에 있다"는 위치였다.
- **검증이 통과로 위장됐다.** 이 구현을 동시성 테스트로 확인하려 했으나, 테스트가 타이밍 의존이라 흡수 경로(충돌에서 진 쪽)를 결정적으로 밟지 못했다. 충돌이 안 난 실행이 "통과"로 잡혀 버그가 가려졌다 — 검증이 실질적으로 불충분했다.

## 결정한 것 — 올바른 구조

구조는 이 PR을 위해 따로 작성한 낙관 락 충돌 처리 설계 문서로 확정했다(작성 시점엔 아직 repo 밖 초안이라 worktree로 들여올 예정이었다). 한 줄 요지: **"문제는 흡수한 게 아니라, 흡수를 트랜잭션 안에서 한 것이다."** 그래서 충돌 처리를 세 계층으로 가른다.

- **transition (별도 빈, public `@Transactional`):** `find → 도메인 전이 → saveChecked` 호출만 한다. 충돌도 가드 위반도 catch하지 않는다 → 도메인 예외가 그대로 전파되고 트랜잭션이 깨끗이 rollback된다.
- **adapter `saveChecked` (신규):** `saveAndFlush`로 flush를 adapter 프레임 안으로 당겨와, 진 쪽의 `ObjectOptimisticLockingFailureException`을 `PaymentException`(`PAYMENT_CONCURRENTLY_MODIFIED`, 충돌 전용으로 새로 만든 코드)으로 변환해 던진다.
- **useCase(=orchestrator: 보상·실시간 승인·대사 흐름을 조율, 트랜잭션 없음):** transition을 호출하고, skip이 필요하면 useCase의 private 래퍼 메서드에서 그 도메인 예외를 catch해 skip한다. 이 catch가 트랜잭션 경계 **밖**이라 rollback-only 문제와 무관하다.

이렇게 하면 세 위반이 동시에 풀린다 — DAO 예외 변환은 adapter가(1번), 보상은 트랜잭션 없는 useCase에서 단계별 독립 commit을 유지한 채(2번), 흡수는 트랜잭션 경계 밖에서(3번) 이뤄진다.

- **함정 2개.** transition은 **반드시** useCase와 별도 빈의 public 메서드여야 한다. private이면 `@Transactional`이 무효가 되고, 같은 빈에서 self-call하면 프록시를 우회해 역시 무효가 된다. 그리고 useCase에는 `@Transactional`을 달지 않는다 — 달면 흡수가 다시 트랜잭션 안으로 들어가 원래 문제로 회귀한다.
- **`saveAndFlush`의 의미를 다시 못박았다.** 이건 "실패할 save를 성공시키는 도구"가 아니라 "충돌을 잡을 수 있는 위치(adapter 프레임)로 flush를 당겨오는 도구"다. 선례인 `saveUsed`(예약 소비)·`saveApproved`(승인 저장, `uk_payment_approved_order_key` unique 위반을 이 프레임 안에서 확정)가 `saveAndFlush`를 쓰는 이유가 바로 이것이다.

### 예외 코드 granularity — 의미 코드 vs 일반 코드

- **충돌은 일반 코드로 던진다.** `PAYMENT_CONCURRENTLY_MODIFIED`("다른 처리가 먼저 상태를 바꿈")로 던지고, "그래서 지금 무엇이 됐는지"가 필요하면 재조회로 판정한다. 이 코드가 실린 `PaymentException`은 409(CONFLICT)로 매핑돼 있어, 전파 정책일 때 그대로 409 응답이 된다(adapter에서 변환되지 못하고 새어 나간 날 낙관 락 예외를 받는 전역 안전망 핸들러도 같은 409로 응답하므로, 어느 경로든 충돌은 409로 수렴한다).
- **unique 위반과 version 충돌은 절대 한 코드로 합치지 않는다.** 주문당 SUCCEEDED 중복을 막는 unique 위반은 `PAYMENT_DUPLICATE`로 매핑되는데, 두 충돌은 후속 정책이 다르다 — 중복은 보상(이미 다른 결제가 성공) 쪽, version 충돌은 재시도/skip 쪽. 의미가 반대라 합치면 안 된다.
- **의미 코드는 전제가 좁을 때만 정직하다.** 예약 소비의 `PAYMENT_RESERVATION_ALREADY_USED` 같은 의미 코드는 "version 충돌 = 이미 사용됨"이 1:1로 성립하는 전제 위에서만 참이다. 같은 행에 다른 동시 쓰기 경로가 하나만 늘어도 거짓 양성이 되고, 그건 컴파일로 안 잡힌다. 그래서 새로 짜는 전이는 의미 코드를 붙이지 않고 **일반 코드 + 재조회**로 간다.

### 낙관 락 vs 조건부 UPDATE (트레이드오프)

- 둘 중 하나가 기본인 게 아니라 **전이 성격**으로 고른다. 낙관 락 + catch는 전이가 도메인 로직을 품을 때(이 코드베이스의 주력 방식) 맞다. 조건부 UPDATE(WHERE 가드 + InnoDB 행 락)는 단순 멱등 status 플립일 때 맞다 — 충돌이 예외가 아니라 affected rows=0으로 나와서 앞의 트랜잭션 rollback-only 딜레마 자체가 소멸하지만, 대신 전이 로직이 SQL로 내려가 DDD 관점에서 냄새가 난다.
- **이번엔 낙관 락 주력으로 결정.** escalation도 조건부 UPDATE로 되돌리지 않고 도메인 메서드(`escalate()`)로 유지한다(escalation 가능 상태·멱등 가드를 엔티티 안에 모아 네 전이 `succeed`/`fail`/`markUnknown`/`escalate`가 한곳에서 일관되게 표현되고, 통지 정확히 1회는 `@Version`이 보장). 예외적으로, 여러 애그리거트를 한 트랜잭션에 묶는 승인 확정(`succeedApproval`: payment 전이 + order 완료)만 낙관 락과 order 행 비관 락(`findByIdForUpdate`)을 혼용한다.

## 배운 것

- **"흡수했다"와 "흡수를 트랜잭션 안에서 했다"는 전혀 다른 문제다.** 낙관 락 충돌 흡수가 UnexpectedRollbackException을 부르는 근본 원인은 catch의 존재가 아니라 catch의 위치(트랜잭션 경계 안/밖)다. 이 프레이밍이 잡히니 `REQUIRES_NEW` 같은 국소 수정이 왜 안 통하는지도 바로 설명된다.
- **`@Version` 충돌은 flush 시점에 그 트랜잭션을 rollback-only로 만든다.** 그래서 같은 트랜잭션 안에서 catch하고 정상 리턴하면 어떤 전파 옵션(`REQUIRES_NEW` 포함)을 써도 commit에서 터진다. 흡수는 반드시 트랜잭션 경계 밖에서.
- **타이밍 의존 동시성 테스트는 충돌 경로를 안 밟은 채로 "통과"할 수 있다.** 충돌에서 진 쪽 경로를 결정적으로 강제하지 못하면 버그가 그대로 초록불로 위장된다 — 충돌 경로 자체를 결정적으로 유발하는 테스트가 필요하다.
- **의미 코드의 거짓 양성은 컴파일로 안 잡힌다.** "version 충돌 = 특정 비즈니스 의미"라는 1:1 전제는 그 행의 동시 쓰기 경로가 늘어나는 순간 조용히 깨진다. 전제가 좁다는 확신이 없으면 일반 코드 + 재조회가 안전하다.

## 미해결·열린 질문

- **전면 코드 구현은 의도적으로 별도 세션으로 분리했다** — transition 재구성 + useCase private 래퍼 + escalation 전환 + 결정적 충돌 테스트. 이 세션이 이미 너무 길어졌기 때문이다. 다만 기반 변경(새 충돌 코드 `PAYMENT_CONCURRENTLY_MODIFIED` + adapter `saveChecked`)은 이미 worktree에 반영해 뒀다.
- **루트 예외 처리 정책 문서 보강도 별도로 남겼다** — 예외 처리 전략 문서에 "낙관 락 충돌 처리" 절을 정본으로 신설할 계획(메커니즘 / 처리 위치 / 충돌 정책 3종인 전파·skip·retry / 코드 granularity / 조건부 UPDATE 대안). 전체 architecture 재정비는 또 다른 별도 PR로 미룬다.
- **가드 위반 코드의 상태 코드가 skip 경로에 안 맞을 수 있다.** 가드 위반 시 던지는 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`는 현재 HTTP 500인데, 가드 위반이 skip 대상이 되는 경로에서는 부적절할 수 있어 재검토가 필요하다.
- 추적: 이슈 #243, PR #245. 이 과정에서 겪은 AI 협업·늦은 컨텍스트 피로 교훈은 같은 PR의 별도 세션으로 분리했다.
