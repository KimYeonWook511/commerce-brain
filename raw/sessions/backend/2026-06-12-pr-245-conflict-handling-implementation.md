---
platform: backend
author: KimYeonWook511
created: 2026-06-12
origin:
  - { type: pr, repo: commerce-backend, ref: 245 }
---

# Payment 낙관 락 충돌 처리 구현 — 설계를 코드로 옮기며 확정·변경된 것

Payment 엔티티에 `@Version` 낙관 락을 도입하면서(같은 결제 행을 동시에 read-modify-write 전이하는 succeed vs fail 등에서 lost update를 막기 위함), "충돌을 어떻게 흡수하느냐"의 구조를 선행 설계 메모에서 확정한 뒤 이 세션에서 실제 코드로 옮겼다. 확정된 구조는 세 계층이다 — 상태 전이 전용 빈(이하 transition: 자기 트랜잭션 안에서 `find → 도메인 전이 → 저장`을 수행하는 public `@Transactional` 메서드)은 충돌을 **트랜잭션 안에서 전파**해 깨끗이 rollback시키고, 이를 조율하는 orchestrator(이하 useCase: 보상·실시간 승인·대사 흐름을 엮는 트랜잭션 없는 상위 계층)는 **트랜잭션 밖에서 skip**하며, 영속화 adapter의 신규 저장 경로 `saveChecked`가 인프라 예외를 도메인 예외로 **변환**한다. 구현하면서 독립 리뷰·AI 코드 리뷰를 거쳐 여러 지점이 추가로 확정되거나 바뀌었다. 이슈 #243, PR #245.

`saveChecked`는 `saveAndFlush`로 flush를 adapter 프레임 안에서 강제로 일으켜, 진 쪽이 던지는 Spring DAO 예외 `ObjectOptimisticLockingFailureException`을 도메인 예외 `PaymentException(PAYMENT_CONCURRENTLY_MODIFIED)`(신설한 충돌 전용 코드, HTTP 409 `PAYMENT-409-6` "다른 처리가 먼저 결제 상태를 변경했습니다")로 변환해 던진다. flush를 adapter 프레임 안으로 당겨오는 이유는 "실패할 save를 성공시키기 위해서"가 아니라 "충돌을 잡을 수 있는 위치에서 확정시키기 위해서"다. 변환만 하고 catch 블록에서 추가 DB 쓰기는 하지 않는다 — 충돌 후 트랜잭션은 이미 rollback-only이기 때문이다.

## 결정한 것

### 1. skip을 useCase로 올리자 두 보상용 메서드가 무조건 전이와 하나로 합쳐졌다

skip 판단을 transition 안이 아니라 useCase(트랜잭션 경계 밖)로 올리니, 그동안 "사전 상태 체크 후 조건부로 흡수"하던 보상 전용 메서드들이 존재 이유를 잃고 무조건 전이와 같은 형태가 됐다.

- **`failIfPending`(미확정이면 FAILED로 만들고 아니면 흡수) → 무조건 `fail`로 병합.** 보상 흐름에서 상태를 확인해 흡수하던 로직이 useCase의 skip 래퍼로 빠지니, 남는 건 `find(orElseThrow) → 도메인 `fail()` → saveChecked`뿐이라 대사(reconcile)의 무조건 `fail`과 똑같아졌다.
- **`markUnknownIfRequested`(REQUESTED일 때만 UNKNOWN 마킹) → `markUnknown`으로 개명.** 조건 분기를 도메인 가드로 흡수했다. 도메인 메서드 `markUnknown`은 상태가 REQUESTED가 아니면 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`를 던진다.
- **책임 분담이 명확해졌다.** 도메인 가드가 상태가 안 맞으면 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`, 이력이 없으면 `find`의 `orElseThrow`가 `PAYMENT_RECORD_NOT_FOUND`를 던지고, skip은 useCase의 private 래퍼가 skip 대상 3종(충돌 `PAYMENT_CONCURRENTLY_MODIFIED` / 가드 위반 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED` / 미존재 `PAYMENT_RECORD_NOT_FOUND`)만 흡수한다. 이 3종은 보상 서비스에서 `SKIPPABLE` enum 집합으로 명시돼 있다.

### 2. 선행 설계가 남긴 가드 위반 코드 미결을 "새 코드 없이 흡수 집합에 편입"으로 해소

선행 설계 메모가 열어둔 미결이 하나 있었다 — "가드 위반 시 던지는 `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`가 현재 HTTP 500(`PAYMENT-500-1`)인데, 가드 위반이 skip 대상이 되는 경로에서는 이 500이 부적절할 수 있어 재검토 필요."

- **결론: 새 코드를 만들 필요가 없다.** 그 코드를 그대로 두고, useCase skip 래퍼의 흡수 집합(`SKIPPABLE`)에 넣어 **skip 경로에서는 클라이언트에 노출되지 않게** 흡수한다.
- **반대로 무조건 전이 경로(대사의 `fail`, 승인 확정 `succeedApproval` = payment 전이 + order 완료를 한 트랜잭션에 묶음)에서 이미 종착 상태가 가드 위반을 내면 그건 진짜 버그 신호**라 500 전파가 맞다.
- 한 줄로: **"skip 경로면 흡수 / 무조건 경로면 전파".** 같은 코드가 호출 맥락에 따라 다르게 다뤄진다.

### 3. CANCEL succeed/fail 충돌은 흡수한다 — "전파" 원칙은 APPROVE 종착 한정 (독립 리뷰 발견)

`saveChecked` 도입의 부수효과를 독립 리뷰에서 잡았다. 취소(CANCEL) 결제의 succeed/fail 충돌이 이제 `PaymentException(PAYMENT_CONCURRENTLY_MODIFIED)`이 되면서, 보상 흐름 `runPgCancel`(PG 취소 호출 + 그 결과를 CANCEL 레코드에 기록하는 묶음)의 기존 `catch(PaymentException)` best-effort 처리에 **흡수**된다. 이전엔 raw DAO 예외라 그대로 전파됐던 것이다.

- **"succeed·무조건 fail은 전파" 원칙은 사실 승인(APPROVE) 종착 기준이었다.** 그 원칙의 목적은 "PG에 과금됐는데 결제 이력엔 실패로 기록"되는 모순을 막는 것이라, APPROVE 종착에만 걸린다.
- **CANCEL succeed 충돌에서 진 쪽은 이미 다른 주체가 같은 CANCEL 레코드를 종착시킨 중복 보상**이므로 흡수가 멱등적으로 옳다. 미해소분은 REQUESTED로 남아 CANCEL 대사(아직 미구현, Epic #208)에서 재확정된다.
- 코드 변경은 없다. 이 경계를 이 task의 ADR에 한 줄로 명확화했다("전파는 APPROVE 종착 기준").

### 4. escalation 멱등을 조건부 UPDATE에서 도메인 메서드로 환원

`@Version` 도입 전에는 Payment에 version이 없어 메모리 가드로 race를 못 막았고, 그래서 escalation 멱등을 DB 레벨 원자성(조건부 UPDATE의 영향 행 수)으로 보장했었다. `@Version`이 생기면서 그 전제가 사라져, escalation을 도메인 메서드로 되돌렸다.

- **`Payment.escalate(now)`**: escalation 가능 상태(UNKNOWN/REQUESTED)이고 아직 escalation되지 않았으면(`escalatedAt IS NULL`) `escalatedAt`을 기록하고 `true`, 아니면 no-op으로 `false`를 반환한다.
- **통지 주체 판정**: escalate transition(`find → escalate() → saveChecked`)이 그 boolean과 `saveChecked` 성공을 함께 보고 통지 주체를 정한다. 동시 시도 중 진 쪽은 `saveChecked`가 `PAYMENT_CONCURRENTLY_MODIFIED`를 던져 skip되므로 **통지가 정확히 1회**만 나간다.
- **repository 조건부 UPDATE `escalateIfPending`은 완전히 제거**했다. 규칙(어떤 상태에서·한 번만)을 SQL WHERE에서 도메인 메서드로 올리니 네 전이(`succeed`/`fail`/`markUnknown`/`escalate`)가 모두 엔티티 가드에 모여 일관되고, 통지 정확히 1회는 `@Version`이 보장한다.
- **추가 안전성 근거**: `@DynamicUpdate`가 없어 version bump 없는 CAS를 유지했다면 동시 `fail()` save가 `escalatedAt`을 stale 값으로 덮어쓸 위험이 있었다. 도메인 메서드 환원이 결국 더 단순하고 안전했다.

### 5. 결정적 충돌 테스트는 계층을 갈라야 했다

선행 설계가 "충돌의 진 쪽을 결정적으로 강제해 검증하라"고 했는데, transition이 자기 트랜잭션 안에서 재조회(`find`)하는 구조라 단일 스레드로는 transition 내부 충돌을 결정적으로 만들 수 없었다 — 재조회가 항상 최신 version을 다시 로드하기 때문이다. 그래서 검증을 두 지점으로 갈랐다.

- **(1) adapter `saveChecked`의 변환**: 같은 행을 두 번 조회한 detached 복사본 중 하나를 먼저 저장해 version을 bump한 뒤, stale 복사본을 저장 → `PAYMENT_CONCURRENTLY_MODIFIED`로 변환됨을 확인(통합 테스트).
- **(2) useCase skip 래퍼의 흡수**: transition을 mock해 충돌을 던지게 하고, `SKIPPABLE`이면 흡수·아니면 재전파를 확인(단위 테스트).
- 실제 race에서 lost-update가 없다는 것은 기존 동시성 테스트가 담당한다.
- **교훈으로 일반화**: "결정적 검증"의 자리는 메커니즘 변환 지점(adapter)과 정책 분기 지점(useCase 래퍼)이지 풀 플로우가 아니다. 검증 가능한 지점을 억지로 한 곳에 몰지 않는다.

## 막힌 점·해결

### AI 코드 리뷰 인라인 4건은 전부 outdated·기각 재제안이라 reject

이 PR에 붙은 AI 코드 리뷰(Gemini) 인라인 지적 4건은 모두 이미 낡았거나 이전에 기각한 제안의 재탕이었다 — 핵심은 "흡수를 `@Transactional(REQUIRES_NEW)`로 감싸라"는 재제안. 그러나 REQUIRES_NEW를 메서드에 붙여도 같은 메서드 안에서 충돌을 catch하면 그 새 트랜잭션이 rollback-only가 되어 commit 시점에 `UnexpectedRollbackException`이 여전히 난다. 문제의 본질은 "흡수했다"가 아니라 "흡수를 트랜잭션 안에서 했다"이므로, 트랜잭션 밖(useCase)으로 흡수를 옮긴 이번 구조가 정답이다. 전부 reject 처리했다.

## 배운 것

- **skip 판단의 위치를 옮기니 메서드 개수가 줄었다.** 조건부 흡수를 도메인 가드(상태 위반 예외) + 상위 계층 흡수(skip 집합)로 분해하니, `failIfPending`/`markUnknownIfRequested` 같은 "상태 보고 분기하던" 보상 전용 메서드가 무조건 전이 하나로 수렴했다. 정책을 계층별로 제자리에 놓으면 특수 메서드가 사라진다.
- **같은 에러 코드가 호출 맥락에 따라 흡수/전파로 갈릴 수 있다.** `PAYMENT_STATUS_TRANSITION_NOT_ALLOWED`를 위해 새 코드를 만들 필요가 없었다 — skip 경로면 SKIPPABLE 집합이 흡수, 무조건 경로면 진짜 버그라 500 전파. 코드 자체보다 "어느 경로에서 났나"가 처리를 정한다.
- **인프라 예외 변환은 도메인 예외를 낳고, 그 변환은 흡수 정책까지 바꾼다.** `saveChecked`가 raw DAO 예외를 `PaymentException`으로 바꾸자, CANCEL 보상의 기존 `catch(PaymentException)`이 그걸 삼키게 됐다 — 이건 부수효과였고 독립 리뷰가 잡았다. 변환 지점을 바꿀 때는 하류 catch가 무엇을 삼키게 되는지까지 봐야 한다.
- **재조회하는 트랜잭션은 단일 스레드 결정적 충돌 테스트가 불가능하다.** 검증은 변환 지점(adapter)과 분기 지점(useCase)으로 계층을 갈라 결정적으로 하고, 실제 race의 무결성은 확률적 동시성 테스트에 맡긴다.

## 미해결·열린 질문

- **낙관 락 충돌 처리의 루트 정본화가 잔류.** 메커니즘(어디서 flush·변환) / 처리 위치(transition 전파 vs useCase skip) / 충돌 정책 3종(전파·skip·retry) / 코드 granularity / 조건부 UPDATE 대안을 한데 모은 "낙관 락 충돌 처리" 섹션을 예외 처리 전략 문서에 정본으로 신설하는 것은 별도 작업으로 미뤘다(선행 설계 메모의 다음 단계 그대로).
- **CANCEL 전용 동시 충돌 재현 테스트는 별도 Epic으로 위임.** 흡수/전파 처리 코드는 CANCEL 경로에도 일관 적용했지만, CANCEL 대사가 아직 미구현이라 CANCEL 종착의 동시 충돌 시나리오 자체가 없다. 전용 재현 테스트는 CANCEL 후처리가 실제 도입되는 Epic #208로 넘겼다(메커니즘 검증은 APPROVE 결정적 충돌 테스트가 대신 담당). CANCEL escalation도 같은 이유로 범위 밖.
