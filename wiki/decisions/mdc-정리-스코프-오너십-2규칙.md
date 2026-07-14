---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [logging, mdc, servlet-filter, observability, trace-id, spring]
created: 2026-07-10
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-07-10-pr-275-mdc-cleanup-scope-ownership]]"
---

# MDC 정리 책임을 스코프-오너십 2규칙으로 — memberId 필터 간 릴레이 제거

## 컨텍스트·문제 — 겹친 필터 3개와 memberId 릴레이 결합

요청 하나는 바깥→안쪽으로 겹친 서블릿 필터 3개를 통과한다: `TraceIdFilter`(최외곽, `traceId` 발급, order `HIGHEST_PRECEDENCE+10`) → `AccessLogFilter`(접근 로그, 요청 종료줄에 status·latency, `+20`) → `JwtAuthenticationFilter`(인증, 최안쪽, `+30`). order 값이 작을수록 바깥이다.

어긋남의 뿌리: **memberId(인증된 사용자 식별자)는 가장 안쪽 인증 필터에서만 알 수 있는데, 그 값을 실어야 할 "요청 종료" 접근 로그는 그 바깥 필터에서, 인증 필터의 `finally`가 이미 memberId를 지운 뒤에 찍힌다.** 값을 아는 지점(안쪽)과 로그를 찍는 지점(바깥)이 갈려, 종료 로그에 memberId를 실으려면 릴레이가 필요했다 — 인증 필터가 `request.setAttribute(...)`로 값을 넘기고, 접근 로그 필터가 종료 로그 직전 MDC에 재삽입했다가 직후 재제거. 결과적으로 요청당 put/remove가 각 2회, 필터 간 request attribute 결합이 생겼다.

관측 동작(종료줄에 memberId 실림 + 요청 끝나면 스레드 MDC 잔류 0)은 이미 맞게 나오고 있었다. 이번 작업은 관측 결과는 그대로 두고 **구조만** 단순화한다.

## 검토한 대안

- **A. 접근 로그 필터가 memberId를 지운다**(원래 이슈안): 인증 필터는 넣기만, 접근 로그 필터가 종료 로그 후 "마지막 소비자"로서 지운다. → 접근 로그 필터가 인증 관심사(memberId 정리)를 떠안아 결합이 여전히 남는다. 기각.
- **B. 필터 순서를 뒤집어 인증 필터를 최외곽으로**: 넣은 쪽이 지우는 대칭. → 인증 실패(401)에서 인증 필터가 조기 반환하면 그 안쪽이 된 접근 로그 필터를 건너뛰어 **401 요청의 접근 로그가 사라진다**. memberId 소비 시점도 어긋난다. 탈락.
- **C. 채택 — 스코프 경계 기준 2규칙.**

## 결정 + 이유 — 최외곽 clear·중첩 자기 키만

- **(a) 최외곽 요청 필터가 요청 끝(`finally`)에서 MDC 전체를 `clear()`로 비운다.**
- **(b) 그보다 안쪽 nested 스코프는 자기가 넣은 키만 remove 한다.**
- memberId는 (a)에 얹는다 — 안쪽 인증 필터가 populate만 하고, 최외곽 `clear()`가 정리한다. 접근 로그 필터는 memberId를 안 건드리는 순수 로거로 남는다.

**왜 C인가.** 최외곽 필터의 `finally`는 안쪽 스코프가 전부 풀린 요청 종료 지점이라, 그때의 `clear()`는 바깥에 살아남은 스코프가 없어 남의 키를 조기 삭제하지 않고 오히려 스레드 풀 반납 전 잔류를 막는 최종 보루다. 반대로 안쪽/중첩 스코프에서 `clear()`를 부르면 아직 살아있는 **바깥** 스코프의 키(예: 유스케이스가 진입 시 넣은 `orderId`·`pgPaymentId`)를 날린다 — 단일 동기 요청 스레드에서 MDC 스코프는 호출 스택을 따라 엄격히 중첩되기 때문. 그래서 정리 책임이 "최외곽 = 통째 clear, 중첩 = 자기 키만"으로 갈린다. 일반화하면, **값을 아는 지점(안쪽)과 정리하는 지점(스코프 소유자)이 갈릴 때 억지 대칭(넣은 쪽이 뺀다)을 고집하기보다 populate/teardown을 분리하는 게 깔끔하다** — 대안 B가 대칭 고집의 부작용(401 로그 유실)을 보여준다.

## 방향이 뒤집힌 지점 — clear는 최외곽에서만 안전

처음엔 "`clear()`는 뭉툭하니 무조건 피하고 키별로 지우자"는 쪽이었고, 최외곽 필터에도 그런 방어 주석(중첩 스코프에서 push될 다른 키를 같이 날리는 위험 차단)이 붙어 있었다. 그런데 그 위험이 **중첩 스코프에서만** 성립하고 최외곽에서는 성립하지 않는다는 걸 짚고 나니, 최외곽의 `clear()`가 오히려 더 단순하고 잔류 방지에 더 견고하다고 방향이 뒤집혔다. 방어 주석도 "최외곽 스레드 경계라 clear()로 스코프 전체를 정리한다"는 취지로 교체했다.

## 규약 정합화 — 상충처럼 읽히던 두 규칙 재서술

로깅 컨벤션 문서가 표면적으로 모순돼 보였다 — 핵심 원칙 요약은 "자신이 push한 키만 remove, `MDC.clear()` 주의"인데 MDC 절은 "요청 종료 시 반드시 `MDC.clear()` 호출"이었다. 이 둘은 **적용 스코프가 다른 두 규칙**인데 한 문서 안에서 상충처럼 읽혔다. 위 (a)/(b) 두 규칙으로 재서술해 정합화했다(운영 코드의 nested 스코프 `clear()`는 금지, 테스트 격리용 `clear()`만 허용 명시). 이 "규약을 코드 현실에 맞게 재서술"은 [[문서-코드-정합성-개념정본-심볼최소화]] 원칙의 적용이다.

## 코드 구조로 강제

공용 MDC 유틸 `LogContext`에서 `removeTraceId()`는 남기되 **`removeMemberId()`는 삭제**하고 `MDC.clear()`를 감싼 `clear()`를 추가했다. memberId teardown을 키별 remove로 열어두지 않고 최외곽 `clear()`만 소유하도록 API 자체를 비운 것 — "populate은 안쪽, teardown은 스코프 소유자"가 서술이 아니라 코드 구조로 강제된다. 릴레이용 request attribute 상수와 그 사용처(인증 필터 setAttribute, 접근 로그 필터 재삽입·재제거)도 모두 제거했다. `removeTraceId()`를 남긴 이유는 비동기 경계가 규칙 (b)로 계속 쓰기 때문이다.

## 미해결

- 이 모델은 **최외곽 필터가 MDC를 만지는 가장 바깥으로 유지됨**을 전제한다. 최외곽 order가 `HIGHEST_PRECEDENCE+10`이라 그보다 작은 값(`+0~+9`)에 MDC 키를 넣는 필터를 추가하면, 최외곽 `clear()`가 그 키를 바깥 필터의 `finally` 전에 지운다. 그런 필터 도입 시 이 정리 모델을 먼저 재검토해야 한다(전제는 ADR·규약에 남김).
- 비동기 경계(Kafka/Outbox)의 traceId 정리는 규칙 (b)(자기 키만 remove)와 정합한다 — 남의 스레드에 얹혀 돌기 때문. traceId를 명시적으로 실어 보내고 받는 쪽에서 복원하기로 한 기존 결정(PR #157)과 충돌 없음. 이 필터 체인의 예외 디스패치 경계는 [[서블릿-필터-예외-처리와-에러-디스패치]]와도 인접하다.

> 정본 ADR·로깅 규약은 코드 repo(commerce-backend)에 있다. 이 노트는 왜 2규칙·어디서 방향이 뒤집혔는지의 판단 기록이다.

## 근거

- [[raw/sessions/backend/2026-07-10-pr-275-mdc-cleanup-scope-ownership]]
