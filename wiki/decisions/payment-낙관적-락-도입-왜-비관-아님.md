---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, optimistic-lock, pessimistic-lock, concurrency, lost-update, order, version]
created: 2026-06-11
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-11-why-payment-optimistic-lock]]"
  - "[[raw/sessions/backend/2026-06-11-pr-242-escalation-version-gap]]"
---

# Payment에 @Version(낙관 락) 도입 — 왜 비관 락이 아닌가

이 노트는 "무엇을, 왜" 낙관 락을 골랐나에 관한 것이다. 충돌을 실제로 "어떻게" 흡수하는 구조는 후속 PR #245의 [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]가 이어받는다.

## 컨텍스트·문제 — Payment만 @Version 부재

핵심 도메인 엔티티 `Order`·`PaymentReservation`·`CartItem`·`Stock`은 모두 `@Version`(행 버전 컬럼으로 동시 수정 충돌을 감지하는 낙관 락)을 갖는데, `Payment`만 빠져 있었다. 이 빈틈은 [[결제-escalation-종착통지-escalatedAt-직교필드]] 작업(PR #242) 도중, escalation 멱등을 "메모리 가드로 race를 막는다"고 쓰려다 "Payment엔 `@Version`이 없어 행 단위 원자성을 못 얻는다"는 자각으로 드러났고, 넓은 이슈(#243)로 분리됐다.

빈틈의 실체: `Payment`의 종착 전이(`fail()`·`markUnknown()` 같은 read-modify-write)는 "status가 REQUESTED/UNKNOWN이 아니면 예외"라는 **메모리 상태 가드**뿐이라, 같은 행을 두 트랜잭션이 동시에 전이하면 나중 커밋이 앞을 덮는 lost update가 난다. 특히 **succeed(승인 성공)와 fail(실패 처리)이 같은 행에서 겹치면 "PG는 과금했는데 우리는 FAILED로 기록"**되는 돈 사고가 가능하다. 이 세션은 그 방어책을 정하는 구현 전 설계(이슈 #243)이며, 방향만 확정하고 코드는 아직 없다.

## 검토한 대안 — 낙관 락 vs 비관 락 (판단 기준: tx 길이가 아니라 충돌 빈도)

- **낙관 락(`@Version`)** — 채택. 근거는 워크로드 특성이다.
  - payment+order 상태를 바꾸는 트랜잭션이 매우 짧다(상태를 두 번 바꾸는 게 전부) → 충돌 확률이 낮다. 낮은 충돌·짧은 tx는 낙관 락의 전형적 적합 조건이다.
  - 이 코드베이스엔 이미 두 원칙이 확립돼 있다 — 외부 PG 호출은 트랜잭션 밖에 두고, 부수효과(메일·알림·포인트·Redis 캐시 갱신)는 결제 성공 커밋 이후에만 실행한다. 그래서 기능이 늘어도 상태 변경 tx는 구조적으로 계속 짧게 유지된다 → 낙관 락이 앞으로도 적합하게 남는다.
  - 낙관 락도 동시 write 시 먼저 update한 쪽이 InnoDB 행 X-lock을 잡고 두 번째는 flush 시점에 대기한다. 앞 tx가 짧아 금방 커밋하면 진 쪽은 version 불일치로 즉시 `OptimisticLockException`을 받고, 재시도 루프가 없으니 커넥션을 빨리 반환한다.
- **비관 락(`SELECT ... FOR UPDATE`)** — 여기선 부적합. **비관이 유리한 건 tx가 길 때가 아니라 충돌 빈도가 높을 때다**(예: 배치가 같은 행을 동시 수정). 긴 tx는 오히려 비관 락에 독이다(락 보유가 길어짐). 지금 결제 상태 전이엔 그런 고빈도 충돌 워크로드가 없다.

판단 기준을 명시적으로 세운 게 이 결정의 핵심이다 — **"tx 길이"가 아니라 "충돌 빈도"가 낙관/비관을 가른다.** 같은 결의 판정이 [[예약-동시소비-가드-version-vs-cas]]에도 적용된다(그쪽은 정확성 동등 시 도메인 표현력으로 갈랐다).

## 결정 — @Version 낙관 락 + 재시도 루프 없음

- `Payment`에 `@Version`을 추가해 종착 전이의 동시 write 충돌을 DB 버전 불일치(`OptimisticLockException`)로 감지한다.
- **자동 재시도 루프는 두지 않는다.** 충돌은 흡수(fail·markUnknown) 또는 전파(succeed)로만 처리한다.
  - **fail·markUnknown은 단조 종착**이라 충돌 = 이미 다른 주체가 종착시킴 → 재시도가 아니라 멱등 흡수(skip).
  - **succeed는 전파한다.** succeed가 졌다면 상대가 SUCCEEDED일 땐 재호출 시 find 흡수로 자연 처리되지만, 상대가 FAILED/UNKNOWN이면 "PG는 과금했는데 우리는 실패로 기록"한 모순이라 조용히 흡수하면 돈 문제가 묻힌다 → 드러나야 한다. 단조 종착 흡수는 "종착 의도"(fail)에만 맞고, succeed는 "성공 의도"라 흡수하면 의미가 왜곡된다.
- `@Version`은 UPDATE 전이만 보호한다. 생성(INSERT, 새 row)의 중복 차단은 별개 메커니즘 — 예약의 `@Version`, 주문당 성공 1건 unique(`uk_payment_approved_order_key`), 멱등 키 `(merchant_pay_key, provider, pg_payment_id, type)` unique — 가 담당한다. 이 DB 레벨 멱등의 결은 [[find-first-write-not-check-db-unique-멱등]]과 같다.

## order 비관 락은 유지 — 낙관·비관의 상호보완

승인 반영 경로 `succeedApproval`(payment 전이 + order 완료를 한 트랜잭션에 묶음, [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]])은 order를 PK 비관 락(`findByIdForUpdate`)으로 잠가 같은 주문의 동시 승인 반영을 직렬화한다. 잠그는 건 order 행뿐이고 payment 행 락은 없다. 이번 `@Version` 도입은 이 order 비관 락을 건드리지 않는다.

- **왜 order는 비관인가:** 결제 도메인 재설계에서 세운 원칙 — 존재 보장(없으면 INSERT, 있으면 막기)은 unique로, 여러 행을 합산해 내리는 계산 기반 판단(예: 부분취소 과다취소 검증)은 Order PK 단일 행 FOR UPDATE로. 또 승인 반영은 PG 과금 이후라 충돌 시 재시도가 위험해, 비관의 "대기 후 멱등 흡수(재진입 시 이미 SUCCEEDED면 흡수)"가 낙관의 "예외→재시도"보다 깔끔하다.
- **둘은 상호보완이다.** order 락은 order 락을 잡는 경로끼리만 직렬화한다. `fail`은 order 락을 안 잡으므로, order 락으로 못 막는 **succeed-vs-fail lost update를 payment `@Version`이 막는다.** succeed-vs-succeed는 order 비관 락이 이미 직렬화한다.

## 트레이드오프·리스크

- **부분취소(합산 검증) 도입 시 비관 재정당화.** "낙관이 정답"은 지금의 단순 상태 전이에 한정된 결론이다. 여러 취소 레코드를 합산해 과다취소를 판정하는 부분취소가 들어오면 충돌 성격이 바뀌어 비관 락이 다시 정당해질 수 있다.
- **order 낙관 전환은 별도 작업.** 타당한 고민이나 #243에 끼우지 않는다 — 그 재설계 결정을 supersede해야 하고 승인 반영의 흡수 로직도 재작성해야 하기 때문이다.
- **escalation 멱등 메커니즘 변화.** `@Version`이 생기면 escalation을 조건부 UPDATE(CAS)로 멱등 보장하던 전제가 사라진다 → escalation을 도메인 `escalate()` 메서드로 환원한다([[결제-escalation-종착통지-escalatedAt-직교필드]]의 조건부 UPDATE 멱등이 이 지점에서 갈아엎힘). 이 환원의 확정은 [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]].

## 미해결·후속

- **succeed의 `OptimisticLockException` 전파를 호출자가 어떻게 다루나** — 실시간 승인(`NaverPayApprovalService`)·대사 루프(`PaymentReconciliationService`) 각각의 처리는 구현 상세에서 확정(→ PR #245).
- **escalation의 트랜잭션 경계·통지 순서·멱등 skip 판정의 코드 표현** — 구현 상세에서 확정(→ PR #245).
- **order 비관 락 → 낙관 전환** — 부분취소가 들어오면 재판단.

이 설계 이후 실제 충돌 처리 구조는 [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]에서 확정된다.

## 근거

- [[raw/sessions/backend/2026-06-11-why-payment-optimistic-lock]] — 낙관 락 도입 설계 본체, 판단 기준(충돌 빈도), order 상호보완.
- [[raw/sessions/backend/2026-06-11-pr-242-escalation-version-gap]] — `Payment`만 `@Version` 부재라는 빈틈의 최초 발견 맥락(#243 분리).
