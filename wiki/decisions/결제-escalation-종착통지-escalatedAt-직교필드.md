---
type: decision
status: superseded
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, escalation, reconciliation, idempotency, concurrency, notification]
created: 2026-06-11
updated: 2026-08-17
superseded_by: "[[결과회수-상한-폐지와-백오프-표-통지-반복]]"
sources:
  - "[[raw/sessions/backend/2026-06-11-pr-242-escalation-version-gap]]"
---

# 결제 escalation 종착·통지 — 새 상태 대신 escalatedAt 직교 필드

> [!warning] 뒤집힘 (2026-08) — 세 가지가 갈렸다
> 결제·환불 모델 재설계에서 이 결정의 세 축이 각각 바뀌었다.
> - **자동 회수를 멈추는 상한 자체가 폐지됐다.** "6시간 넘으면 통지하고 자동 대사에서 뺀다"가 "멈추지 않는다"로 갔다 — **통지가 로그 한 줄이고 받아서 처리할 경로가 없어 멈춤이 실질적으로 방치였다**([[결과회수-상한-폐지와-백오프-표-통지-반복]]).
> - **통지가 한 번이 아니라 상태가 이어지는 동안 반복된다.** 그리고 시각을 먼저 찍고 보내는 순서가 뒤집혀, 아래 "정확히 한 번" 멱등 자체가 목표에서 빠졌다.
> - **"자동 처리가 더 못 간다"를 직교 필드가 아니라 상태로 저장한다**([[승인은-다시-물어-확정-환불에는-실패-종착이-없다]]). 아래 (A) 기각 근거("사실과 파생을 섞는다")는 유효하지만, "자동으로 더 못 간다"는 파생 분류가 아니라 사실이라는 재판정을 받았다.
>
> **살아남은 것:** 통지를 포트로 추상화하고 실채널을 후속으로 미룬 배치, 그리고 "새 상태를 만들기 전에 직교 필드를 먼저 본다"는 순서 자체.

## 컨텍스트·문제 — 6시간 초과 미확정 결제가 통지 없이 묻힘

자동 대사가 6시간 안에 결론을 못 낸 미확정 APPROVE 결제(UNKNOWN/REQUESTED)가 그동안 통지 없이 `UNKNOWN`으로 묻혀, 운영자가 능동 조회해야만 인지할 수 있었다(PR #242, 이슈 #238). 이 세션은 그런 건을 운영자에게 통지하고 "종착 표시"하는 escalation 경로를 구현하면서, ① 종착을 어떤 축으로 표현할지, ② 중복 통지를 어떻게 막을지를 정했다. 그 과정에서 `Payment`에만 `@Version`이 빠졌다는 넓은 빈틈을 발견해 #243으로 분리했다([[payment-낙관적-락-도입-왜-비관-아님]]).

## 종착 표현 3안 검토 → escalatedAt 직교 필드

- **(A) 새 결제 상태(`ESCALATED`)** — status에 "escalation됨"을 박는다.
- **(B) `escalatedAt` 직교 필드** — 채택. status는 그대로 두고 nullable timestamp 컬럼에 escalation 시각만 기록.
- **(C) 별도 테이블** — escalation 이력을 독립 엔티티로.

**(B)를 택했다.** status는 "결제에 일어난 사실"(미확정=UNKNOWN, 요청중=REQUESTED 등)만 담는 축이고, "이 건이 운영자에게 위임됐나"는 그와 **직교하는 별개 축**이다. 새 상태(A)는 "결론이 났나/처리됐나"를 status 한 값에 뭉개 사실과 파생을 섞는다. 마침 같은 이유로 이미 철회한 선례가 있다 — 자동 처리를 포기하고 수동 확인으로 넘긴 건을 표현하려던 `MANUAL_REVIEW` 상태를 "한 상태가 두 현실을 뭉갠다"는 바로 그 이유로 도입하지 않기로 되돌린 적이 있다([[payment-status-사실만-분류는-정책계산-manual-review-철회]]). 별도 테이블(C)은 지금 필요가 "한 번 통지하고 종착 표시" 하나뿐이라 과하다(YAGNI) — 이력·단계가 실제로 필요해지면 그때 승격.

구현: `Payment`에 `escalatedAt`(`LocalDateTime`, nullable)을, `tbl_payment.escalated_at`(`DATETIME(6) NULL`)을 Flyway로 추가. nullable이라 백필 불필요(NULL = 미escalation). 상태 enum은 `REQUESTED/SUCCEEDED/FAILED/UNKNOWN` 4개 그대로.

## 멱등 메커니즘 — 조건부 UPDATE 영향 행 수 (@Version 도입 후 도메인 메서드로 대체됨)

여러 대사 사이클(또는 다중 인스턴스)이 같은 건을 동시에 집어도 통지는 정확히 한 번만 나가야 한다.

- 초기 설계는 멱등을 "스캔 필터(1차) + 도메인 `escalate()` **메모리 가드**(2차, race 방어)"로 잡았다. 그런데 그 가드는 로드한 객체의 `escalatedAt == null`을 검사한 뒤 save하는 방식이라 두 트랜잭션이 동시에 `null`을 읽으면 둘 다 통지한다 → 실제 race를 못 막는다. 근본 이유는 `Payment`에 `@Version`이 없어 행 단위 원자성을 못 얻는다는 것. "race 방어"라는 표현과 메커니즘의 실체가 어긋나 있었다.
- 그래서 멱등의 진실 원천을 **조건부 UPDATE의 DB 레벨 원자성**으로 교정했다. repository `escalateIfPending(id, now)` — `SET escalated_at=:now WHERE id=:id AND escalated_at IS NULL AND status IN (UNKNOWN, REQUESTED)` — 가 영향 행 수(int)를 반환하고, **영향 행이 1인 호출만이 escalation 주체**로서 커밋 이후 통지, 0이면 skip. 이는 `uk_payment_approved_order_key`가 이중 SUCCEEDED를 막는 것과 같은 결의 DB 레벨 멱등이다([[find-first-write-not-check-db-unique-멱등]]).

> [!warning] 진화(supersede) — 이 멱등 메커니즘은 후속에서 갈아엎혔다
> `@Version`이 `Payment`에 도입되면서(#243) "메모리 가드로는 race를 못 막는다"는 전제 자체가 사라졌다. escalation은 조건부 UPDATE `escalateIfPending`을 **완전히 제거**하고 도메인 메서드 `escalate(now)`로 환원됐다 — 네 전이(`succeed`/`fail`/`markUnknown`/`escalate`)를 모두 엔티티 가드에 모으고, 통지 정확히 1회는 `@Version`이 보장한다. 확정은 [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]. **단, 여기서 정한 `escalatedAt` 직교 필드·status 불변·통지 정신은 그대로 유지된다** — 바뀐 건 "멱등을 무엇으로 보장하나"뿐이다.

## 대사 루프 말미 통합 + 별도 스캔 쿼리, 별도 스케줄러 없음

기존 자동 대사 스캔은 짧은 윈도우(UNKNOWN 1분, REQUESTED 15분 하한 ~ 6시간 상한)의 미확정 건만 잡는다. escalation은 그 상한(6시간)을 넘겨 자동 대사에서 빠진 건이 대상이라, 기존 `findStaleApprovePaymentsForReconciliation`을 건드리지 않고 별도 `findEscalationCandidates`를 새로 뒀다. `PaymentReconciliationService.reconcile()`의 기존 stale 루프가 끝난 뒤 `processEscalations()`로 이어 처리하며, **별도 스케줄러는 만들지 않았다**(같은 대사 진입점 안). 범위는 승인(APPROVE)만 — CANCEL 대사 자체가 미구현이라 CANCEL escalation도 제외(별도 이슈).

## 주문없음(order==null) — 자동환불 금지, 실패 종착 + 통지

대사 중 결제에 매달린 주문이 조회되지 않는 정합성 붕괴(`order == null`)를 만나면 **자동 환불(PG 취소)을 하지 않는다.** 원인 불명의 붕괴 상태에서 자동으로 PG 취소를 거는 것은 위험하다는 판단. 대신 `FAILED`로 종착시킨 뒤 운영자에게 통지해 사람이 판단하게 한다.

## 6시간 시간 기준 status별 분리

- 미확정(`UNKNOWN`)은 PG 응답 수신 시각(`respondedAt`) 기준.
- 요청중(`REQUESTED`)은 생성 시각(`createdAt`) 기준.
- 차이는 "PG 응답을 받았는지"다 — UNKNOWN은 응답을 받았으나 결과 불명이라 응답 시각이, REQUESTED는 응답 자체가 없어 생성 시각이 기준점이다.

## 통지 추상화 — NotificationPort + 로그 어댑터, best-effort

통지는 `NotificationPort` + 현재는 로그 어댑터 구현만 두고, 실채널(디스코드 웹훅 등)은 후속으로 미뤘다 — hook 지점만 대사 flow에 미리 박고 채널 교체를 adapter 교체로 끝내려는 것. escalation·주문없음 통지 모두 `notifyManualReviewRequired(orderId, merchantPayKey, reason)`를 호출하며, 커밋 이후 best-effort(try/catch로 감싸 전송 실패가 트랜잭션·루프를 막지 않고 `log.warn`만)다.

## 리뷰 처리 — 축이 다른 두 지적을 갈라서 처리

- **Gemini** — `escalateIfPending`에 `REQUIRES_NEW` 전파를 붙이자는 제안. **reject.** 호출처(`processEscalations`)가 트랜잭션을 열지 않으므로 기본 `@Transactional`(REQUIRED)만으로 이미 독립 커밋되어 "커밋 후 통지"가 보장된다. `REQUIRES_NEW`는 상위 트랜잭션이 있다는 (현재 없는) 가정을 깔고 repository에 전파 정책을 박아 레이어를 오염시킨다.
- **codex** — 주문없음 통지가 다중 인스턴스에서 중복될 수 있다는 지적. 같은 행 동시 `fail()`이 둘 다 성공하면 통지가 두 번 나가는 **lost update의 증상**이다. 표면 패치 대신 근본 이슈(#243, `Payment @Version`)로 위임했다.

둘은 다른 축이다 — Gemini는 커밋 타이밍, codex는 동시 수정 lost update. 무관한 Gemini 지적을 근본 이슈에 잘못 엮지 않는 게 중요했다.

## 배운 것 — 멱등은 DB 레벨, 메모리 가드는 단일 tx 재진입만 막음

- **동시성을 "고려했나"가 아니라 "어떤 형태로 고려했나"를 봐야 한다.** 중복·이중·진입을 다 막아도 행 단위 lost update 방어가 한 엔티티에만 빠질 수 있다(여기선 `Payment`만 `@Version` 부재). 낙관 락 적용의 일관성을 별도로 점검해야 한다.
- **멱등은 메모리 가드가 아니라 DB 레벨(조건부 UPDATE 영향 행 수 또는 낙관 락)로 보장해야 동시 race에 안전하다.** "race 방어"를 문서에 쓰려면 그게 정말 동시 race를 막는 메커니즘인지 작성 시점에 self-check해야 한다 — 이 교훈은 [[동시성-테스트-작성-규칙과-단언-전략]]과도 이어진다.

## 미해결 — CANCEL escalation, 실채널 adapter, @Version 도입 후 재검토

- **CANCEL escalation** — CANCEL 대사 자체가 미구현이라 대사 구현과 묶어 별도 이슈.
- **실제 알림 채널 adapter** — 지금은 `NotificationPort` + 로그 어댑터뿐.
- **`@Version` 도입(#243) 후 멱등 메커니즘 재검토** — 위 진화 콜아웃대로 조건부 UPDATE는 도메인 `escalate()`로 대체됨([[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]).

## 근거

- [[raw/sessions/backend/2026-06-11-pr-242-escalation-version-gap]] — escalation 종착 표현·멱등 메커니즘·리뷰 처리·`@Version` 빈틈 발견의 원본(PR #242, 이슈 #238).
