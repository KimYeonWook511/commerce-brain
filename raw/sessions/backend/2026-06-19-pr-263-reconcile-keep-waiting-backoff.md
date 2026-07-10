---
platform: backend
author: KimYeonWook511
created: 2026-06-19
origin:
  - { type: pr, repo: commerce-backend, ref: 263 }
---

## 한 일
- 결제 대사 스캔이 KEEP_WAITING(PG가 아직 PENDING/NOT_FOUND라 결론 못 냄)으로 끝난 건을
  매 주기 다시 긁어 PG에 재조회하던 문제를 backoff로 막았다. PR #263, 이슈 #239의 남은 부분.
- 윈도우 상한(6시간)·REQUESTED 하한(15분)은 선행 PR에서 이미 닫혀 있었고, 이번엔 "대기 건
  누적이 새 후보를 굶기고(starvation) 같은 건을 매 1분 PG에 두드리는" 부분만 남아 있었다.

## 결정한 것
- backoff를 담을 자리로 새 직교 컬럼 next_reconcile_at을 추가했다. status 상태머신을 안 건드리는
  타임스탬프 필드로, escalation 시각(escalated_at)이 이미 검증해 둔 패턴을 그대로 빌렸다.
  스캔 쿼리는 "next_reconcile_at이 NULL이거나 now 이하"인 행만 후보로 본다. NULL=한 번도 안
  미뤄진 즉시 대상이라 기존 행·신규 행 동작이 그대로 보존됐다(백필 불필요).
- respondedAt 재사용은 안 했다. escalation·stale 윈도우 계산이 그 필드에 의존해서, backoff
  용도로 덮으면 그 계산이 오염된다.
- backoff 간격은 단일 고정 값(5분)으로 했다. 이슈 코멘트엔 내가 "재조회 간격을 점증(지수)"이라
  적었지만, 그건 예시였고 acceptance(starvation 해소 + PG 조회 빈도 감소)는 고정 간격만으로
  충족된다. 스캔 상한(6시간)이 무한 재시도를 이미 막아 한 건당 PG 조회가 bound되므로,
  next_reconcile_at과 별개로 시도 횟수를 세는 새 컬럼까지 더하는 지수 backoff는 과설계로 보고
  뺐다. 필요해지면 나중에 카운터 추가.
- backoff 기록은 "아직 대기"로 끝나는 분기(APPROVE 대기, CANCEL 대기·취소 재시도 처리중)에서만
  했다. succeed/fail/markUnknown처럼 status를 확정하는 분기는 이미 행을 써서 자기 재진입 cadence가
  있어서, 거기에 backoff까지 더하면 두 시점 필드가 경합한다.
- 정본 결정 기록: docs/tasks/reconciliation-backoff/adr.md

## 검토에서 잡힌 것 (실행 전 독립 검토 에이전트)
- 스캔 쿼리에 now 파라미터를 더하면 시그니처가 바뀌어 그 메서드를 호출/mock하던 테스트 3개가
  전부 컴파일 깨진다. step 문서에 "시그니처 바꾸면 호출부·mock도 같이 고친다"를 안 적으면
  developer가 컴파일 단계에서 막힌다. → 사전 보강.
- backoff를 기록하는 service를 처음엔 기존 CANCEL 전이 service들처럼 CANCEL 전용 finder 하나로
  적으려 했는데, 리포지토리 포트는 APPROVE용·CANCEL용 finder가 type별로 분리돼 있다. 단일 CANCEL
  finder로는 APPROVE 건을 못 찾아 "행 없음"으로 조용히 흡수돼서, APPROVE backoff가 절반만 동작하는
  (소리 없이 안 걸리는) 공백이 생긴다. service가 type으로 finder를 고르도록 고쳤다. 이건 usecase를
  mock으로 보면 안 잡혀서 service 단위 테스트로 finder 분기를 못 박았다.

## 배운 것
- "양쪽(APPROVE·CANCEL)에 일관 적용"이라는 요구는 적용 경로가 실제로 둘 다 거치는지 코드로
  확인해야 한다. 로드 경로가 한쪽만 가리키면 나머지 절반은 에러도 없이 조용히 안 한다.
- 이슈 본문/코멘트의 "예: ~" 예시를 요구로 굳히지 말 것. acceptance가 진짜 뭘 요구하는지로
  최소 설계를 정한다. ("점증"이라 적어놓고 고정으로 간 케이스)
- Testcontainers(docker 태그) 통합 테스트는 일반 test가 아니라 integrationTest task로 돌려야
  실제 실행된다. test로 적으면 "통과"가 사실은 "실행 안 됨"이다.

## 다음 단계
- 지수 backoff(시도 카운터)는 의도적으로 뺐다. PG 조회 빈도를 더 줄여야 할 운영 신호가 보이면 도입.
- #260(FAILED CANCEL 자동 재시도)에서 이 next_reconcile_at backoff 메커니즘을 재사용할 수 있다.
