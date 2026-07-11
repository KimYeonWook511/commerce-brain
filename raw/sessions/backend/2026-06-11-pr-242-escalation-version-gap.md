---
platform: backend
author: KimYeonWook511
created: 2026-06-11
origin:
  - { type: pr, repo: commerce-backend, ref: 242 }
---

# 결제 escalation 종착·통지 — 새 상태 대신 escalatedAt 직교 필드 + 조건부 UPDATE 멱등

자동 대사가 6시간 안에 결론을 못 낸 미확정 APPROVE 결제(UNKNOWN/REQUESTED)가 그동안 통지 없이 `UNKNOWN`으로 묻혀, 운영자가 능동 조회해야만 인지할 수 있었다. 이 세션은 그런 건을 운영자에게 통지하고 "종착 표시"하는 escalation 경로(#238)를 구현한 것이다. 종착을 어떤 축으로 표현할지(새 상태 vs 직교 필드), 중복 통지를 어떻게 막을지(멱등)를 정하고, 그 과정에서 `Payment` 엔티티에만 낙관 락(`@Version`)이 빠져 있다는 더 넓은 동시성 빈틈을 발견했다.

## 결정한 것

### 1. escalation 종착을 새 결제 상태 대신 `escalatedAt` 직교 필드로 표현

종착 표현으로 세 가지를 검토했다.

- **(A) 새 결제 상태 도입** — `ESCALATED` 같은 상태를 만들어 "escalation됨"을 status에 박는다.
- **(B) `escalatedAt` 직교 필드** — status는 그대로 두고, nullable timestamp 컬럼에 escalation 시각만 기록한다.
- **(C) 별도 테이블** — escalation 이력을 독립 엔티티로 둔다.

**(B)를 택했다.** status는 "결제에 일어난 사실"(미확정=UNKNOWN, 요청중=REQUESTED 등)만 담는 축이고, "이 건이 운영자에게 위임됐나"는 그와 직교하는 별개 축이다. 새 상태(A)는 "결론이 났나 / 처리됐나"를 status 한 값에 뭉개서, 사실과 파생(운영 위임 여부)을 섞는다. 마침 같은 이유로 이미 철회한 선례가 있다 — 자동 처리를 포기하고 운영자 수동 확인으로 넘긴 건을 표현하려던 `MANUAL_REVIEW` 결제 상태를, "한 상태가 두 현실을 뭉갠다"는 바로 그 이유로 이전에 도입하지 않기로 되돌린 적이 있다. 별도 테이블(C)은 지금 필요가 "한 번 통지하고 종착 표시" 하나뿐이라 과하다(YAGNI) — escalation 이력·단계가 실제로 필요해지면 그때 테이블로 승격할 여지만 남긴다.

- **구현:** `Payment` 도메인에 `escalatedAt`(`LocalDateTime`, nullable) 필드를 추가하고, `tbl_payment.escalated_at`(`DATETIME(6) NULL`) 컬럼을 Flyway 마이그레이션으로 얹었다. nullable 추가라 기존 행 백필은 불필요하다(NULL = 미escalation). 결제 상태 enum은 `REQUESTED/SUCCEEDED/FAILED/UNKNOWN` 4개 그대로 유지된다.

### 2. 중복 통지 방지(멱등)를 조건부 UPDATE의 영향 행 수로 보장

여러 대사 사이클(또는 다중 인스턴스)이 같은 escalation 건을 동시에 집어도 통지는 정확히 한 번만 나가야 한다. 이를 **조건부 UPDATE의 DB 레벨 원자성**으로 보장한다.

- repository에 `escalateIfPending(id, now)`를 추가했다 — `SET escalated_at=:now WHERE id=:id AND escalated_at IS NULL AND status IN (UNKNOWN, REQUESTED)` 형태로, 영향 행 수(int)를 반환한다. **영향 행이 1인 호출만이 자기가 escalation 주체**라고 보고 커밋 이후 통지하며, 0이면 이미 다른 주체가 처리한 것이라 통지하지 않는다.
- **메모리 객체 가드는 쓰지 않는다.** "로드한 `Payment`의 `escalatedAt == null`을 검사한 뒤 save" 하는 방식은 두 트랜잭션이 동시에 `null`을 읽으면 둘 다 통지하므로 race를 못 막는다. 그 근본 이유는 `Payment`에 `@Version`(낙관 락)이 없어 행 단위 원자성을 얻을 수 없다는 것 — 그래서 멱등의 진실 원천을 DB의 조건부 UPDATE 하나로 둔다. (이 방식은 "한 주문에 성공 결제 하나"를 강제하는 `uk_payment_approved_order_key` unique가 이중 SUCCEEDED를 막는 것과 같은 결의 DB 레벨 멱등이다.)

### 3. escalation 스캔을 대사 스캔과 별도 쿼리로 두고 대사 루프 말미에 통합

- 기존 자동 대사 스캔은 짧은 시간 윈도우(UNKNOWN은 1분, REQUESTED는 15분 하한 ~ 6시간 상한) 안의 미확정 건만 잡는다. escalation은 그 상한(6시간)을 넘겨 자동 대사에서 빠진 건이 대상이라, 기존 스캔 쿼리(`findStaleApprovePaymentsForReconciliation`)를 건드리지 않고 별도 `findEscalationCandidates` 쿼리를 새로 뒀다. 대사 서비스(`PaymentReconciliationService`)의 `reconcile()`에서 기존 stale 처리 루프가 끝난 뒤 `processEscalations()`로 이어 처리하며, **별도 스케줄러는 만들지 않았다**(같은 대사 진입점 안에서 처리).
- **범위는 승인(APPROVE)만.** 취소(CANCEL) 대사 자체가 아직 미구현이라 CANCEL escalation도 이번 범위에서 제외했다(별도 이슈).

### 4. 주문 없음(order==null) 정합성 오류는 자동 환불 없이 실패 종착 + 통지

대사 중 결제에 매달린 주문이 조회되지 않는(`order == null`) 정합성 붕괴 상태를 만나면, **자동 환불(PG 취소)을 하지 않는다.** 원인 불명의 정합성 붕괴 상태에서 자동으로 PG 취소를 거는 것은 위험하다는 판단이다. 대신 `FAILED`로 종착시킨 뒤 운영자에게 통지해 사람이 판단하게 한다.

### 5. 6시간을 재는 시간 기준을 status별로 분리

- 미확정(`UNKNOWN`)은 PG 응답을 받은 시각(`respondedAt`) 기준으로 6시간을 잰다.
- 요청중(`REQUESTED`)은 생성 시각(`createdAt`) 기준으로 잰다.
- 차이는 "PG 응답을 받았는지"다 — UNKNOWN은 응답을 받았으나 결과가 불명한 상태라 응답 시각이, REQUESTED는 아직 응답 자체가 없어 생성 시각이 기준점이다. escalation 스캔 쿼리와 조건부 UPDATE가 이 기준을 그대로 반영한다.

### 통지 추상화

통지는 `NotificationPort`(알림 추상화) + 현재는 로그 어댑터 구현만 두고, 실제 채널(디스코드 웹훅 등)은 후속으로 미뤘다 — hook 지점만 대사 flow에 미리 박아 두고 채널 교체를 adapter 교체로 끝내려는 것. escalation 통지·주문없음 통지 모두 이 port의 `notifyManualReviewRequired(orderId, merchantPayKey, reason)`를 호출하며, 커밋 이후 best-effort(try/catch로 감싸 전송 실패가 트랜잭션·루프를 막지 않고 `log.warn`만 남김)다.

## 막힌 점·발견

### 멱등이 처음엔 메모리 가드로 설계됐다가 조건부 UPDATE로 교정됨

초기 설계에서는 멱등을 "2겹 방어: 스캔 필터(1차) + 도메인 `escalate()` 가드(2차, race 방어)"로 잡았다. 그런데 그 도메인 가드는 로드한 객체의 `escalatedAt == null`을 검사하는 **메모리 가드**라 실제 동시 트랜잭션 race를 못 막는다(`Payment`에 `@Version`이 없어서). "race 방어"라는 표현과 메커니즘의 실체가 어긋나 있었고, 이 불일치가 드러나 escalation 종착을 조건부 UPDATE(CAS) + 영향 행 수 판단으로 교정했다. 메모리 상태 가드는 단일 트랜잭션 재진입만 막는다는 게 핵심 교훈이다.

### `Payment`에만 `@Version`(낙관 락)이 빠져 있다

`Order`·`PaymentReservation`·`CartItem`·`Stock`에는 전부 `@Version`이 붙어 있는데 **`Payment`만 빠져 있다**. 그래서 종착 전이(실패·미확정 마킹 등 `fail()`/`markUnknown()` 같은 read-modify-write)에서 같은 행을 동시 수정하면 lost update가 가능하다. 이를 별도 이슈(#243)로 분리했다.

- **머지는 막지 않았다.** 발현 조건이 비현실적이다 — succeed/fail이 같은 행에서 동시에 갈리려면 서로 다른 PG 응답을 봐야 하고, 단일 인스턴스 운영 전제라 대사가 순차로 돈다. 다만 다중 인스턴스 운영이면 통지 중복이 실제로 나므로(아래 codex 지적) 표면 패치 대신 근본 이슈(#243)로 일반화해 뒀다.
- **`Payment`는 그동안 `@Version` 없이 경로별 개별 메커니즘으로 동시성을 막아 왔다:** 결제 생성 = 예약의 `@Version`, 이중 성공 = 주문당 성공 1개 unique(`uk_payment_approved_order_key`), 동시 승인 = 주문 행 비관 락, escalation = 이번 조건부 UPDATE. 통합된 행 단위 보호(`@Version`)만 빠진 것이고, 이번 PR 안에서 escalation엔 멱등을 넣으면서 주문없음/종착 전이엔 안 넣은 비일관도 처음부터 점검했어야 했다.

### PR 리뷰: Gemini는 reject, codex는 #243으로 위임

리뷰가 축이 다른 두 지적을 냈고, 서로 다른 문제로 갈라서 처리했다.

- **Gemini** — 조건부 UPDATE 메서드(`escalateIfPending`)에 `REQUIRES_NEW` 전파를 붙이자는 제안. **reject.** 호출처(`processEscalations`)가 트랜잭션을 열지 않으므로 기본 `@Transactional`(REQUIRED)만으로도 이미 독립 커밋되고, 따라서 "커밋 후 통지"가 보장된다. `REQUIRES_NEW`는 상위 트랜잭션이 있다는 (현재 없는) 가정을 깔고 repository에 전파 정책을 박아 레이어를 오염시킨다.
- **codex** — 주문 없음 통지가 다중 인스턴스에서 중복될 수 있다는 지적. 같은 행 동시 `fail()`이 둘 다 성공하면 통지가 두 번 나갈 수 있다는 것으로, **lost update의 증상**이다. 표면 패치 대신 근본 이슈(#243, `Payment @Version`)로 위임했다.

둘은 다른 축이다 — Gemini는 트랜잭션 커밋 타이밍, codex는 동시 수정 lost update. 무관한 Gemini 지적을 근본 이슈에 잘못 엮지 않는 게 중요했다.

## 배운 것

- **동시성을 "고려했나"가 아니라 "어떤 형태로 고려했나"를 봐야 한다.** 중복·이중·진입은 다 막았어도, 행 단위 lost update 방어가 한 엔티티에만 빠질 수 있다. 여기선 핵심 엔티티 중 `Payment`만 `@Version`이 없었다 — 낙관 락 적용의 일관성을 별도로 점검해야 한다.
- **멱등은 메모리 가드가 아니라 DB 레벨(조건부 UPDATE 영향 행 수, 또는 낙관 락)로 보장해야 동시 race에 안전하다.** 메모리 가드는 단일 트랜잭션 재진입만 막는다. "race 방어"를 문서에 쓰려면 그게 정말 동시 race를 막는 메커니즘인지 작성 시점에 self-check해야 한다.

## 미해결·열린 질문

- **`Payment @Version` 도입 (#243):** 종착 전이의 lost update 방어와 다른 엔티티와의 일관성 복구. 이번엔 escalation 멱등을 조건부 UPDATE로 막아 뒀지만, `@Version`이 들어오면 그 전제가 바뀌므로 멱등 메커니즘도 재검토 대상이 된다.
- **CANCEL escalation:** CANCEL 대사 자체가 미구현이라, 대사 구현과 묶어 별도 이슈로 남긴다.
- **실제 알림 채널 adapter:** 지금은 `NotificationPort` + 로그 어댑터뿐이고, 디스코드 등 실채널 adapter는 후속으로 분리돼 있다.
