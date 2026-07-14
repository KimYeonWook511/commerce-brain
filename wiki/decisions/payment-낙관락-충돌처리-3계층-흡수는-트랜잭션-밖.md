---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, optimistic-lock, concurrency, exception-handling, transaction-boundary, conflict-handling, compensation, escalation]
created: 2026-06-11
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-12-pr-245-optimistic-lock-conflict-handling]]"
  - "[[raw/sessions/backend/2026-06-12-pr-245-conflict-handling-implementation]]"
  - "[[raw/sessions/backend/2026-06-11-why-payment-optimistic-lock]]"
---

# Payment 낙관 락 충돌 처리 — 흡수를 트랜잭션 밖으로 빼는 3계층 구조

[[payment-낙관적-락-도입-왜-비관-아님]]에서 `@Version` 채택 자체는 정했지만 "충돌을 어떻게 흡수하나"는 미결로 남았다. 이 노트(PR #245)가 그 흡수 구조를 처음으로 확정한다.

## 컨텍스트 — @Version 채택 후 남은 '충돌 흡수' 미결

`@Version` 도입을 harness로 구현(version 컬럼 추가 / 종착 전이 충돌 흡수 / escalation 도메인 환원)해 PR #245를 열었는데, 리뷰 단계에서 "충돌을 흡수하는" 구현이 잘못됐음을 발견했다. 흡수 구조를 확정하는 게 이 세션의 본론이다.

## harness 초기 구현이 위반한 정책 3개

harness는 보상·best-effort 종착 전이 메서드(`markUnknownIfRequested`, `failIfPending`) 안에서 application이 Spring DAO 예외 `ObjectOptimisticLockingFailureException`을 직접 catch해 skip하고, 그 메서드의 `@Transactional`을 제거했다. 세 정책을 한꺼번에 어겼다.

1. **application이 Spring DAO 예외를 직접 catch.** 정책은 "인프라 예외는 adapter에서 도메인 예외로 변환, application/domain은 DAO 타입에 의존 안 함"이다. 같은 도메인에 선례가 있다 — `PaymentReservationRepositoryAdapter.saveUsed`는 `@Version` 충돌을 **adapter 안에서** `PaymentException(PAYMENT_RESERVATION_ALREADY_USED)`로 변환해 던진다([[예약-동시소비-가드-version-vs-cas]]). harness는 정반대로 갔다.
2. **`@Transactional`을 제거.** 보상 흐름은 "각 단계가 자기 `@Transactional`로 독립 commit"이 정책이다. 보상은 PG 응답과 정합을 단계별로 맞추는 작업이라 단일 트랜잭션으로 묶이면 뒤 단계 실패가 앞 단계까지 롤백해 부분 진행 보존이 깨진다. `@Transactional`을 떼면 그 독립 commit이 무너진다.
3. **흡수(skip)를 트랜잭션 안에서 했다 — 이게 진짜 버그다.** `@Version` 충돌은 flush 시점에 터지고 그 순간 트랜잭션이 rollback-only로 마킹된다. 같은 메서드 안에서 그 예외를 catch하고 정상 리턴하면 commit 시점에 `UnexpectedRollbackException`이 난다. 선례 `saveUsed`의 주석은 이미 "충돌 후 tx는 rollback-only"임을 알아 흡수하지 않고 **변환 throw만** 한다.

- **AI 코드 리뷰(Gemini)는 2번만 부분적으로 짚었다** — `@Transactional` 대신 `REQUIRES_NEW`를 쓰라는 제안. 그러나 `REQUIRES_NEW`를 붙여도 **같은 메서드 안에서 catch하면** 그 새 트랜잭션이 rollback-only가 되어 여전히 `UnexpectedRollbackException`이 난다. 뿌리는 전파 옵션이 아니라 "catch가 트랜잭션 경계 안에 있다"는 **위치**였다.
- **검증이 통과로 위장됐다.** 타이밍 의존 동시성 테스트라 흡수 경로(진 쪽)를 결정적으로 밟지 못해, 충돌이 안 난 실행이 "통과"로 잡혀 버그가 가려졌다.

## 결정 — 3계층 구조 (transition 전파 / adapter saveChecked 변환 / useCase skip)

핵심 프레이밍 한 줄: **"문제는 흡수한 게 아니라, 흡수를 트랜잭션 안에서 한 것이다."** 그래서 충돌 처리를 세 계층으로 가른다.

- **transition (별도 빈, public `@Transactional`):** `find → 도메인 전이 → saveChecked` 호출만 한다. 충돌도 가드 위반도 catch하지 않아, 도메인 예외가 그대로 전파되고 트랜잭션이 깨끗이 rollback된다.
- **adapter `saveChecked` (신규):** `saveAndFlush`로 flush를 adapter 프레임 안으로 당겨와, 진 쪽의 `ObjectOptimisticLockingFailureException`을 `PaymentException(PAYMENT_CONCURRENTLY_MODIFIED)`(충돌 전용 신규 코드, HTTP 409 `PAYMENT-409-6` "다른 처리가 먼저 결제 상태를 변경했습니다")로 변환해 던진다. 변환만 하고 catch에서 추가 DB 쓰기는 하지 않는다 — 충돌 후 tx는 이미 rollback-only이므로.
- **useCase(=orchestrator: 보상·실시간 승인·대사 흐름 조율, 트랜잭션 없음):** transition을 호출하고, skip이 필요하면 private 래퍼에서 그 도메인 예외를 catch해 skip한다. 이 catch가 트랜잭션 경계 **밖**이라 rollback-only 문제와 무관하다.

이렇게 세 위반이 동시에 풀린다 — DAO 예외 변환은 adapter가(1), 보상은 트랜잭션 없는 useCase에서 단계별 독립 commit 유지한 채(2), 흡수는 트랜잭션 경계 밖에서(3).

## 함정 — public 별도 빈 필수, useCase @Transactional 금지, saveAndFlush 오해

- transition은 **반드시** useCase와 별도 빈의 **public** 메서드여야 한다. private이면 `@Transactional`이 무효, 같은 빈에서 self-call하면 프록시 우회로 역시 무효.
- useCase에는 `@Transactional`을 **달지 않는다** — 달면 흡수가 다시 트랜잭션 안으로 들어가 원래 문제로 회귀.
- **`saveAndFlush`는 "실패할 save를 성공시키는 도구"가 아니라 "충돌을 잡을 수 있는 위치(adapter 프레임)로 flush를 당겨오는 도구"다.** 선례 `saveUsed`(예약 소비, [[예약-동시소비-가드-version-vs-cas]])·`saveApproved`(승인 저장, `uk_payment_approved_order_key` unique 위반을 이 프레임 안에서 확정)가 `saveAndFlush`를 쓰는 이유가 바로 이것이다.

## 예외 코드 granularity — 일반 코드 + 재조회, 의미 코드의 거짓 양성

- **충돌은 일반 코드로 던진다.** `PAYMENT_CONCURRENTLY_MODIFIED`("다른 처리가 먼저 상태를 바꿈")로 던지고, "그래서 지금 무엇이 됐는지"가 필요하면 재조회로 판정한다. 이 코드는 409로 매핑돼 있어 전파 정책이면 그대로 409 응답이 되고, adapter에서 변환 못 되고 새어 나간 낙관 락 예외를 받는 전역 안전망 핸들러도 같은 409로 응답하므로 어느 경로든 충돌은 409로 수렴한다.
- **unique 위반과 version 충돌은 절대 한 코드로 합치지 않는다.** unique 위반은 `PAYMENT_DUPLICATE`로 가는데, 후속 정책이 반대다 — 중복은 보상(이미 다른 결제 성공) 쪽, version 충돌은 재시도/skip 쪽. 의미가 반대라 합치면 안 된다.
- **의미 코드는 전제가 좁을 때만 정직하다.** `PAYMENT_RESERVATION_ALREADY_USED` 같은 의미 코드는 "version 충돌 = 이미 사용됨"이 1:1로 성립하는 전제 위에서만 참이다. 같은 행에 동시 쓰기 경로가 하나만 늘어도 거짓 양성이 되고 컴파일로 안 잡힌다. 그래서 **새로 짜는 전이는 일반 코드 + 재조회**로 간다. 재조회를 최종 진실로 삼는 결은 [[find-first-write-not-check-db-unique-멱등]]과 같다.

## 흡수 vs 전파 정책 — fail·markUnknown 흡수 / succeed 전파, APPROVE 종착 한정

- **fail·markUnknown 충돌은 흡수(skip)** — 단조 종착이라 충돌 = 이미 다른 주체가 종착시킴. skip 판단을 useCase로 올리니 그동안 "사전 상태 체크 후 조건부 흡수"하던 보상 전용 메서드가 존재 이유를 잃었다 — `failIfPending → 무조건 fail`로 병합, `markUnknownIfRequested → markUnknown`으로 개명(조건 분기를 도메인 가드로 흡수, REQUESTED 아니면 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`).
- **succeed 충돌은 전파** — succeed가 졌다면 상대가 FAILED/UNKNOWN일 때 "PG는 과금했는데 우리는 실패로 기록"한 모순이라 드러나야 한다.
- **이 "전파" 원칙은 사실 APPROVE 종착 한정이다(독립 리뷰 발견).** `saveChecked`가 raw DAO를 `PaymentException`으로 바꾸자, CANCEL 결제의 succeed/fail 충돌이 보상 흐름 `runPgCancel`의 기존 `catch(PaymentException)` best-effort에 **흡수**되게 됐다. CANCEL succeed에서 진 쪽은 이미 다른 주체가 같은 CANCEL 레코드를 종착시킨 **중복 보상**이므로 흡수가 멱등적으로 옳다. 미해소분은 REQUESTED로 남아 CANCEL 대사(미구현, Epic #208)에서 재확정. 코드 변경 없이 경계만 ADR에 명확화("전파는 APPROVE 종착 기준").

## 같은 코드가 맥락으로 갈림 — PAYMENT_STATUS_TRANSITION_NOT_ALLOWED, SKIPPABLE 집합

선행 설계가 남긴 미결("가드 위반 코드 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`가 현재 500인데 skip 경로에선 부적절")을 **새 코드 없이** 해소했다.

- 그 코드를 그대로 두고, useCase skip 래퍼의 흡수 집합 `SKIPPABLE`(충돌 `PAYMENT_CONCURRENTLY_MODIFIED` / 가드 위반 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED` / 미존재 `PAYMENT_RECORD_NOT_FOUND` 3종)에 넣어 **skip 경로에서는 클라이언트에 노출 안 되게** 흡수.
- 반대로 무조건 전이 경로(대사의 `fail`, 승인 확정 `succeedApproval` = payment 전이 + order 완료 한 트랜잭션, [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]])에서 이미 종착 상태가 가드 위반을 내면 그건 진짜 버그 신호라 500 전파가 맞다.
- 한 줄: **"skip 경로면 흡수 / 무조건 경로면 전파".** 코드 자체보다 "어느 경로에서 났나"가 처리를 정한다.

## escalation 조건부 UPDATE → 도메인 escalate() 환원

`@Version` 도입 전엔 version이 없어 escalation 멱등을 조건부 UPDATE 영향 행 수로 보장했으나([[결제-escalation-종착통지-escalatedAt-직교필드]]), version이 생기며 그 전제가 사라졌다.

- **`Payment.escalate(now)`**: escalation 가능 상태(UNKNOWN/REQUESTED)이고 `escalatedAt IS NULL`이면 기록하고 `true`, 아니면 no-op `false`.
- **통지 주체 판정**: escalate transition(`find → escalate() → saveChecked`)이 boolean과 `saveChecked` 성공을 함께 보고 판정. 진 쪽은 `saveChecked`가 `PAYMENT_CONCURRENTLY_MODIFIED`를 던져 skip되므로 **통지 정확히 1회**.
- repository `escalateIfPending`은 **완전히 제거**. 규칙을 SQL WHERE에서 도메인 메서드로 올리니 네 전이(`succeed`/`fail`/`markUnknown`/`escalate`)가 모두 엔티티 가드에 모여 일관되다.
- 추가 안전성: `@DynamicUpdate`가 없어 version bump 없는 CAS를 유지했다면 동시 `fail()` save가 `escalatedAt`을 stale로 덮어쓸 위험이 있었다. 도메인 환원이 결국 더 단순·안전.

## 결정적 충돌 테스트는 계층 분리 — 변환 지점 + 분기 지점

transition이 자기 tx 안에서 재조회(`find`)하는 구조라 단일 스레드로는 transition 내부 충돌을 결정적으로 만들 수 없다(재조회가 항상 최신 version을 다시 로드). 그래서 검증을 두 지점으로 갈랐다.

- **(1) adapter `saveChecked` 변환**: 같은 행을 두 번 조회한 detached 복사본 중 하나를 먼저 저장해 version bump 후, stale 복사본 저장 → `PAYMENT_CONCURRENTLY_MODIFIED` 변환 확인(통합 테스트).
- **(2) useCase skip 래퍼 흡수**: transition을 mock해 충돌을 던지게 하고 `SKIPPABLE`이면 흡수·아니면 재전파 확인(단위 테스트).
- 실제 race의 lost-update 무결성은 기존 확률적 동시성 테스트가 담당.
- 일반화: **"결정적 검증"의 자리는 메커니즘 변환 지점(adapter)과 정책 분기 지점(useCase 래퍼)이지 풀 플로우가 아니다** — [[동시성-테스트-작성-규칙과-단언-전략]].

## 배운 것 / 미해결

- **"흡수했다"와 "흡수를 트랜잭션 안에서 했다"는 전혀 다른 문제다.** 근본 원인은 catch의 존재가 아니라 catch의 위치(트랜잭션 경계 안/밖). 이 프레이밍이 잡히니 `REQUIRES_NEW` 같은 국소 수정이 왜 안 통하는지도 바로 설명된다.
- **인프라 예외 변환은 흡수 정책까지 바꾼다.** `saveChecked`가 DAO를 `PaymentException`으로 바꾸자 CANCEL 보상의 기존 `catch(PaymentException)`이 그걸 삼키게 됐다(부수효과, 독립 리뷰가 잡음). 변환 지점을 바꿀 땐 하류 catch가 무엇을 삼키는지까지 봐야 한다.
- **미해결** — 낙관 락 충돌 처리의 루트 정본화(메커니즘/처리 위치/충돌 정책 3종/코드 granularity/조건부 UPDATE 대안을 예외 처리 전략 문서에 정본 신설)는 별도 작업으로 잔류. CANCEL 전용 동시 충돌 재현 테스트는 CANCEL 후처리가 도입되는 Epic #208로 위임(APPROVE 결정적 충돌 테스트가 메커니즘 검증 대신 담당). 가드 위반 코드(`PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`)의 500 자체 재검토는 skip 편입으로 실무적으로는 해소.

## 근거

- [[raw/sessions/backend/2026-06-12-pr-245-optimistic-lock-conflict-handling]] — 3계층 구조 확정, 정책 3위반, 예외 granularity, 낙관 락 vs 조건부 UPDATE 트레이드오프.
- [[raw/sessions/backend/2026-06-12-pr-245-conflict-handling-implementation]] — 구현하며 확정·변경된 것(메서드 병합, SKIPPABLE, CANCEL 흡수, escalate() 환원, 계층별 테스트).
- [[raw/sessions/backend/2026-06-11-why-payment-optimistic-lock]] — 흡수 범위 초안(fail·markUnknown 흡수 / succeed 전파)과 escalation 도메인 환원 방향의 원천.
