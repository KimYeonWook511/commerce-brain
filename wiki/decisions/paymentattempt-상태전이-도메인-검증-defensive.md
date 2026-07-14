---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, payment-attempt, domain-model, defensive-programming, state-transition, idempotency]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# PaymentAttempt 상태 전이 도메인 검증 — 멱등 자기 전이까지 거부

> [!note] 05-29 스냅샷 결정
> [[payment-도메인-구조-개요]] 스냅샷 시점. 이후 06-04 재설계에서 결제 상태 표현이 append-only 원장 + `exists` 완료 판단([[payment-append-only-원장과-exists-완료판단]])으로 진화한다. 이 검증(REQUESTED 단방향 전이 가드)의 정신은 이어지되, 최종 상태를 별도 원장 존재로 판단하는 쪽으로 상태 모델 자체가 재정의되므로 그 관계를 함께 본다.

## 컨텍스트·문제(상위 멱등의 불완전성)

`PaymentAttempt.succeed(respondedAt)` / `fail(failCode, detail, respondedAt)`은 호출 시점에 `status == REQUESTED`를 검증한다. 위반 시 `PAYMENT_ATTEMPT_STATUS_TRANSITION_NOT_ALLOWED`(500, `PAYMENT-500-1`). **멱등 자기 전이(SUCCEEDED → SUCCEEDED)도 거부**한다.

상위 레이어 멱등 처리(`getOrCreate` + `processApproveAttempt` switch)가 *완벽하면* 같은 attempt에 succeed가 두 번 호출될 일이 없다. 하지만 상위는 깨질 수 있다: PG 콜백 중복 발송(네이버페이 retry, redirect URL 두 번 클릭), 상위 switch case 누락, 향후 새 호출 경로(admin manual reconcile 등), 테스트 오류.

## 결정과 선택 이유(defensive, 자기 무결성)

**도메인 모델을 최후의 방어선으로 세운다.** 도메인 메서드가 상위의 완벽함을 가정하지 않고 자기 무결성을 지킨다(defensive programming). 이는 `Order.cancel`(`status != INIT` → throw) 등 주문 도메인의 명시적 선조건 검증([[order-도메인-구조-개요]])과 같은 결의 패턴 — enum-level 가드만 두고 도메인 메서드는 무방어로 두는 대안 대신 도메인 보호를 택했다.

## 거부하지 않으면 위험(실패 사유 소실)

두 번째 `succeed()` 호출 = `failCode = null` 초기화 + `respondedAt` 덮어쓰기 → **흔적 없이 실패 사유가 사라진다.** 운영 시점 추적 불가. 거부하면 즉시 500 → 운영 대시보드 알람 → 상위 멱등 처리 버그를 **즉시 감지**한다.

## type 가드 진화(도입 → service 분리 → 제거)

여기에 type 가드의 진화가 누적돼 있다 — 상위 구조 변화가 도메인 책임 경계를 다시 그린 자연스러운 진화다.

1. **도입:** 초기에는 mark 메서드 4개(`markApproveSucceeded`·`markApproveFailed`·`markCancelSucceeded`·`markCancelFailed`)가 *상태 가드 + type 가드*를 함께 검증했다. 메서드 이름에 처리할 type이 명시돼 있으니, 의도와 다른 type이 들어오면 호출자 실수 → 도메인이 즉시 거부해 메서드 이름에 표현된 의도를 방어했다.
2. **service 분리:** 이후 service를 흐름별로 나누자(`PaymentApprovalAttemptService`는 APPROVE 전용, `PaymentCancellationAttemptService`는 CANCEL 전용) *호출자가 type을 잘못 보낼 경로 자체가 사라졌다*. type 가드의 방어 가치가 소멸.
3. **제거:** mark 4개 → `succeed`/`fail` 2개로 통합(type 가드 제거, 상태 가드 유지).

처음부터 완성된 설계가 아니라 코드 진화의 결과다. 이 흐름별 분리 순서는 [[payment-도메인-구조-개요]]의 application 서비스 구성과 맞물린다.

## 에러코드 500(내부 결함 vs 4xx 외부원인)

에러 코드는 500이다. amount mismatch 같은 외부 원인은 4xx([[payment-amount-mismatch-이중검증-409-vs-400-분리]]), *도메인 무결성 위반(내부 결함)*은 5xx로 운영 대시보드에서 "호출자 4xx" vs "내부 5xx"를 분리한다.

## 택하지 않은 대안(멱등 자기전이 허용 #117)

- **멱등 자기 전이 허용(SUCCEEDED → SUCCEEDED)** — 위 "거부하지 않으면 위험" 때문에 기각. 이 허용 요청은 Issue #117로 열렸다가 이 결정으로 close.

## 다시 본다면·미해결

- **이 검증이 다른 결정을 유발했다.** 새 검증이 들어오며 보상 흐름 catch에서 `fail()` 호출 시 race window에서 throw할 가능성이 생겼다 → [[보상-catch-2차예외-평탄화-원칙]]이 필요해졌고 임시 try-catch 처방이 들어갔다. 이후 [[보상판단-payment-존재-lock-대신-db-unique]]로 race window가 축소되며 임시 처방은 보조 방어선으로 격하됐다.
- **type 가드를 처음부터 안 넣을 것인가** — 어차피 흐름별 service 분리로 없어질 방어라면 처음부터 상태 가드만 둘 수도 있었다. 다만 분리 전에는 실제 방어 가치가 있었다는 점에서 판단이 갈린다(미결 재검토).
- **상태 전이 표 문서화** — 현재 mark 메서드 내부 코드로만 표현. enum/별도 문서에 허용/거부 표를 명시하고 주문 도메인 상태 전이 규칙과 함께 정리.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
