---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, reconciliation, backoff, starvation, next-reconcile-at, escalation, scheduler]
created: 2026-06-19
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-19-pr-263-reconcile-keep-waiting-backoff]]"
---

# 대사 스캔 KEEP_WAITING backoff — status-직교 next_reconcile_at 필드로 재조회를 미룬다

## 컨텍스트 — KEEP_WAITING이 행을 안 써 재스캔·PG 낭비·starvation

결제 대사 스케줄러는 매 1분마다 stale한 APPROVE/CANCEL 결제를 `id ASC` 첫 페이지(batch size)만 스캔해 PG에 실제 상태를 물어본다. PG가 아직 결론을 못 내면(PENDING/NOT_FOUND) 그 건은 `KEEP_WAITING`으로 판정되는데, 이때 **결제 행을 전혀 쓰지 않는다.** 그래서 같은 행이 다음 주기에도 같은 첫 페이지를 그대로 차지해, ① 뒤에 쌓인 새 UNKNOWN 후보가 스캔에서 굶고(starvation) ② 같은 건을 매 1분마다 PG에 반복 조회한다(PG API 낭비·Rate Limit 위험).

이슈 #239가 제기한 문제 중 스캔 윈도우 상한(6시간)과 미성숙 REQUESTED 하한(15분)은 선행 PR([[미확정차단-대사스캔-정합성-starvation-escalation]])에서 이미 닫혔다. 이번엔 그 starvation의 **남은 본체** — "쓰지 않아 매 주기 재스캔되는 대기 건"을 backoff로 미루는 부분만 다뤘다(PR #263).

## 결정 1: next_reconcile_at 직교 필드 + 스캔 게이트 (NULL=즉시, 백필 불필요)

`KEEP_WAITING` 건의 재조회를 미룰 자리로, 결제 행에 `status`와 무관한 직교 타임스탬프 컬럼 `next_reconcile_at`을 추가했다(마이그레이션 V10, `DATETIME(6) NULL`). 스캔 쿼리에는 `(next_reconcile_at IS NULL OR next_reconcile_at <= :now)` 게이트를 더해, 미래 시각이 박힌 행은 그 시각이 지날 때까지 스캔에서 빠진다. backoff가 "방금 본 건"을 스캔에서 빼주므로, 정렬은 `id ASC`를 그대로 두고도 뒤의 새 후보가 첫 페이지로 올라와 starvation이 풀린다(정렬 정책 교체 불필요).

- **왜 status 상태머신을 안 건드리는 직교 필드인가:** escalation 종착 시각을 담는 `escalatedAt` 컬럼이 이미 "새 status를 만들지 않고 직교 타임스탬프로 부가 시점을 표현한다"는 패턴을 검증해 뒀다([[결제-escalation-종착통지-escalatedAt-직교필드]]). backoff도 status와 무관한 "다음 재조회 하한"일 뿐이라 그 패턴을 그대로 빌렸다.
- **NULL = 즉시 대상:** `next_reconcile_at`이 NULL이면 "한 번도 미뤄진 적 없는 즉시 대사 대상"이다. 기존 행과 신규 행 모두 NULL로 시작하므로 게이트가 이전과 동일하게 동작한다 — nullable 추가라 기존 행 백필도 불필요하다.
- **도메인 메서드는 시각만 세팅:** `Payment.delayReconcile(now, backoff)`는 `next_reconcile_at = now + backoff`만 세팅하고 `status`·`respondedAt` 등 다른 필드는 건드리지 않는다. "wait 판정 후 다음 재조회를 미룬다"는 의도만 순수하게 표현한다.
- **검토한 대안:** 스캔 정렬을 `next_reconcile_at`/`respondedAt` 기준으로 바꿔 오래된 것부터 보는 방법 — 정렬만으로는 같은 건의 PG 반복 조회를 줄이지 못하고(윈도우 안에 남아 있으면 계속 후보), 게이트가 더 단순해 기각.

## 결정 2: respondedAt 재사용 안 함 (오염 회피)

backoff 시각을 담을 자리로 기존 `respondedAt` 재사용을 처음 떠올렸으나 접었다. escalation 판정과 stale 스캔 윈도우 계산(특히 CANCEL 스캔의 UNKNOWN 하한)이 `respondedAt`에 의존하고 있어, backoff 용도로 그 값을 덮으면 그 계산들이 오염된다. 별도 필드가 안전하다.

## 결정 3: 고정 5분 backoff, 지수 backoff 제외 (윈도우 상한이 bound)

재조회 backoff는 시도 횟수에 따라 늘리지 않고 단일 고정 간격(초기값 5분)으로 했다. 이 상수(`RECONCILE_BACKOFF = Duration.ofMinutes(5)`)는 스캔 윈도우 상한(6시간, `ESCALATION_DELAY`)과 같은 정책 클래스에 단일 출처로 뒀다.

- **이슈 코멘트의 "점증(지수)"은 예시였지 요구가 아니었다.** #239 코멘트에 "재조회 간격을 점증"이라 적었지만 예시였고, acceptance가 진짜 요구하는 건 starvation 해소 + 같은 건 PG 조회 빈도 감소 두 가지뿐이다. 둘 다 고정 간격으로 충족된다.
- **무한 재시도는 이미 막혀 있다.** 스캔 윈도우 상한(6시간)이 영구 정체 건을 결국 윈도우 밖으로 밀어내므로, 한 건당 PG 조회 횟수는 `6시간 / 간격`으로 이미 bound된다. 시도 횟수 컬럼·상태·테스트까지 더하는 지수 backoff는 지금 스코프에 과설계로 보고 뺐다.
- **트레이드오프:** 영구 정체 transient 건은 고정 간격이라 6시간 동안 `6시간/간격`회까지 PG를 두드린다(지수보다는 많다). 그러나 금전을 바꾸는 호출이 아니라 PG 읽기 조회일 뿐이고 escalation 상한 안에서 bound되므로 허용 범위로 판단했다. 필요하면 나중에 시도 카운터를 가산적으로 얹으면 된다.

## 결정 4: wait 분기에만 backoff, status 확정 분기 제외

`next_reconcile_at` 기록은 대사 결과가 "아직 대기"로 끝나는 세 분기에만 배선했다: APPROVE `KEEP_WAITING`, CANCEL `KEEP_WAITING`, CANCEL 취소 재시도가 아직 처리 중인 `PROCESSING`. succeed/fail/markUnknown처럼 `status`를 확정하는 분기에는 넣지 않았다.

- **왜 확정 분기 제외:** status를 확정하는 분기는 이미 행을 쓴다(예: markUnknown이 `responded_at = now` 갱신). 그 write가 자기 재진입 cadence를 만들어 다음 스캔을 늦춘다. 거기에 backoff까지 더하면 의미가 중복되고 두 시점 필드(`responded_at`·`next_reconcile_at`)가 경합한다. backoff 목적은 "쓰지 않아 재스캔되는 wait 건"을 늦추는 것이라 그 분기에 한정했다.
- **status에 의존하지 않게 둔 이득:** wait 분기에는 UNKNOWN/REQUESTED뿐 아니라 일부 FAILED CANCEL도 도달할 수 있다(후처리 정책이 특정 FAILED CANCEL을 재조회 대상으로 돌리고, PG가 PENDING이면 그 건도 `KEEP_WAITING`). `delayReconcile`이 `status`를 읽지도 바꾸지도 않으므로 어떤 status가 와도 가드 없이 안전하다.
- **동시성은 기존 skip 패턴 그대로:** backoff 기록도 `@Version` 낙관 락을 거쳐 저장한다([[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]). 동시 전이가 먼저 행을 바꿔 `PAYMENT_CONCURRENTLY_MODIFIED`가 나면 대사 루프를 멈추지 않고 흡수(skip)한다 — backoff는 best-effort cadence 힌트라 충돌 시 건너뛰어도 다음 주기에 자연히 재시도된다.

## 실행 전 독립 검토가 잡은 설계 공백 (시그니처 파급, APPROVE finder silent no-op)

구현 착수 전 독립 검토 에이전트가 코드를 대조해 두 결함을 짚었고, step 문서를 보강한 뒤 실행해 둘 다 코드에 들어가기 전에 차단했다. 이 "실행 전 정본·코드 대조"는 [[설계단계-검증-정본·선례-확인실패와-전제의-적대적검증]]과 같은 결의 예방 사례다.

- **시그니처 변경의 파급 — 테스트 3개 컴파일 깨짐.** 스캔 쿼리에 `now` 파라미터를 더하면 4-인자 → 5-인자가 되어, 그 메서드를 호출/mock하던 테스트 세 파일(통합 1 + usecase mock 2)이 전부 컴파일이 깨진다. step 문서에 "시그니처를 바꾸면 호출부·mock stub도 같은 step에서 함께 고친다"를 명시하지 않으면 developer가 "관련 파일"만 보고 `compileTestJava`에서 막힌다. → 사전 보강.
- **APPROVE backoff가 소리 없이 절반만 동작할 뻔(silent no-op).** backoff 기록 service(`DelayPaymentReconcileService`)의 로드 키를 처음엔 CANCEL 전용 finder(`findCancelPayment`) 하나로 적으려 했다. 그런데 리포지토리 포트는 APPROVE용(`findApprovePayment`)·CANCEL용(`findCancelPayment`) finder가 type별로 분리돼 있다. 단일 CANCEL finder로는 APPROVE 건을 못 찾아 `PAYMENT_RECORD_NOT_FOUND`가 나는데, 대사 루프의 skippable 처리가 이 예외를 흡수해 **APPROVE backoff가 에러도 없이 조용히 안 걸린다** — starvation 해소가 CANCEL 쪽만 반쪽으로 동작하는 공백이었다. service가 `type`으로 finder를 고르도록 고쳤다. usecase mock으로는 안 잡혀서 service 단위 테스트로 finder 분기를 직접 못 박았다.

## 배운 것 (양쪽 적용 코드 확인, 예시≠요구, NULL=중립 패턴, integrationTest 태그)

- **"양쪽(APPROVE·CANCEL)에 일관 적용" 요구는 적용 경로가 실제로 둘 다 거치는지 코드로 확인해야 한다.** 로드 경로가 한쪽 type만 가리키면 나머지 절반은 에러도 없이 조용히 안 한다.
- **이슈 본문/코멘트의 "예: ~" 예시를 요구로 굳히지 말 것.** acceptance가 진짜 요구하는 것으로 최소 설계를 정한다. 여기선 "점증"이라 적어놓고도 고정 간격으로 갔다.
- **"어디에 적용하지 않을지"가 "어디에 적용할지"만큼 설계의 일부다.** 이미 자기 cadence를 가진 확정 분기를 backoff에서 뺀 판단이 두 시점 필드의 경합을 막았다.
- **부가 시점이 필요하면 상태머신에 끼워 넣지 말고 직교 필드 + "NULL=중립" 패턴을 먼저 본다.** NULL을 즉시 대상으로 두니 백필도 기존 동작 보존도 공짜로 따라왔다.
- **Testcontainers(docker 태그) 통합 테스트는 일반 `test`가 아니라 격리된 `integrationTest` task로 돌려야 실제 실행된다.** 처음 acceptance를 `test`로 적었는데 그 태그가 `test`에서 제외돼 "통과"가 사실은 "실행 안 됨"이었다. → `@Tag` 격리 테스트의 acceptance는 해당 task를 명시한다. 이 2축 태그 모델은 [[테스트-태그-2축-모델-ci-잡-분리]], 실 경로 검증 철학은 [[도달불가분기-방어금지-불변식테스트-돈정합성-통합테스트]] 참고.

## 미해결·후속 (#260 FAILED CANCEL 재시도 재사용)

- 지수 backoff(시도 카운터)는 의도적으로 뺐다. PG 조회 빈도를 더 줄여야 할 운영 신호가 보이면 시도 횟수 컬럼을 가산적으로 얹어 도입한다.
- FAILED CANCEL 자동 재시도(#260)에서 이 `next_reconcile_at` backoff 메커니즘을 재조회 cadence로 재사용할 수 있다.

## 근거

- [[raw/sessions/backend/2026-06-19-pr-263-reconcile-keep-waiting-backoff]]
