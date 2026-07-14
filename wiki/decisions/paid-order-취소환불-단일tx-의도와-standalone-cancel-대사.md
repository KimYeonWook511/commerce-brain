---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, order, refund, cancel, reconciliation, append-only, idempotency, pessimistic-lock, cross-aggregate, saga]
created: 2026-06-18
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# PAID 주문 취소·환불 — 단일 tx 환불 의도 + standalone CANCEL 대사로 보장

## 배경 — INIT 취소만 있던 공백

그동안 결제 전(INIT) 주문만 취소할 수 있었다 — `Order.cancel()`이 INIT에서만 허용했고, 환불은 시스템이 잘못 승인한 결제를 되돌리는 보상(compensation) 경로에만 있었지 사용자 주도 환불 경로는 없었다. PAID 주문 사용자 취소 시 전액 환불 + 재고 전체 복구 흐름을 채운 세션이다(PR #258). 무게중심은 "환불을 어떻게 **보장**하느냐"였고, 그 과정에서 초안이 기댄 안전망이 코드에 실재하지 않음을 발견해 standalone CANCEL 대사를 신설한 것이 진짜 핵심이 됐다. (INIT 경합을 보상으로 정리한다는 방향은 [[payment-order-facade-결합끊기-tell-dont-ask]]에서 예고됐다.)

## 환불 보장 — 단일 tx 환불 의도 영속화 + 대사 안전망(Outbox 대안 기각)

- **핵심 문제:** PG 취소 호출은 외부 I/O라 주문 취소 트랜잭션에 넣을 수 없다. 그렇다고 "주문을 CANCELED로 커밋한 뒤 환불을 트리거"하는 순서면, 둘 사이에서 프로세스가 죽을 때 주문은 취소됐는데 환불 기록이 없는 — 돈을 안 돌려준 채 취소만 된 — 상태가 남는다.
- **택한 방식:** 조율 service가 한 RDB 트랜잭션 안에서 `CANCEL 결제 REQUESTED(=환불 의도) 영속화 + 주문 취소 + 재고 복구`를 원자적으로 커밋한다. 실제 PG 환불 호출은 이 트랜잭션 **밖**(커밋 이후)에서 best-effort로 실행한다. 어느 시점에 중단돼도 "이 결제는 환불해야 함"이라는 durable 기록이 DB에 남고, 그 기록을 끝까지 마무리하는 책임은 아래 CANCEL 대사가 진다.
- **왜 이 구조:** 단일 DB 조건을 활용하면 이벤트/Outbox 없이 트랜잭션 하나로 order·payment 두 aggregate에 걸친 정합(cross-aggregate consistency)을 확보할 수 있다. 단일 tx cross-aggregate 정합 원칙은 [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]].
- **검토한 대안 — Outbox 이벤트:** 주문 tx에 "환불 필요" 이벤트를 원자적으로 적고 별도 consumer가 집행하는 방식. aggregate 경계는 더 깨끗하지만 환불 전용 이벤트·consumer를 새로 만들어야 해 현재 스코프에 과하다고 봐 기각. 다만 결합이 커지면(부분취소·다채널) 승격 여지를 남겼다 — 영속된 CANCEL REQUESTED 행이 이벤트와 사실상 같은 역할이라 전환이 자연스럽다.
- **트레이드오프:** 조율 service의 단일 tx가 order·payment 두 aggregate 테이블을 함께 쓴다(경계 침범). Outbox 승격을 미래 탈출구로 두고 지금은 감수했다.

## append-only CANCEL 레코드(보상 경로 재사용 안 함)

- 환불을 "승인 행을 취소됨으로 mutate"가 아니라 "별도 CANCEL 레코드 append"로 표현한다. 대상 APPROVE 결제의 SUCCEEDED 상태는 그대로 보존한다. 결제 테이블은 사건을 쌓는 불변 원장이고([[payment-append-only-원장과-exists-완료판단]]), 승인 성공은 이미 일어난 사실, 환불도 별개 사실이라 승인 사실을 훼손하지 않아야 감사·분쟁·부분취소 확장에서 일관된다. "이 결제를 취소했는가"는 승인 행을 뒤집는 게 아니라 CANCEL 레코드의 존재·상태로 판단한다.
- **보상 경로를 재사용하지 않은 이유:** 기존 시스템 보상 경로(`CompensateApprovalUseCase.runPgCancel`)는 "애초에 승인되면 안 됐던" 결제를 되돌리느라 대상 APPROVE를 FAILED로 마킹한다. 사용자 주도 환불은 정당하게 성공한 승인을 되돌리는 것이라 의미가 정반대다. `runPgCancel`을 쪼개 공유하지 않고 별도 환불 실행 경로(`RefundExecutionUseCase`)를 둬, 공유 코드를 건드려 보상 경로에 회귀를 내는 위험을 피했다.
- **트레이드오프:** "이 주문 결제가 지금 유효한가"를 알려면 APPROVE·CANCEL 레코드를 집계해야 한다. 단일 row mutate보다 조회가 복잡하지만 append-only 원장이 치르는 일관된 비용으로 봤다.

## standalone CANCEL 대사 신설(죽은 배선 살림)

- **결정:** `type=CANCEL ∧ status∈{REQUESTED, UNKNOWN}`인 stale CANCEL 결제를 스캔해 PG 재조회·재실행으로 종착시키는 대사(reconciliation) 경로를 신설한다. 위에서 영속한 환불 의도가 PG 호출 전/중 중단으로 남으면 이 대사가 그 행을 집어 끝까지 마무리한다.
- **왜 굳이 새로:** 후처리 대상·흐름을 정하는 정책 뼈대는 이미 있었다 — 후처리 대상 판정에 CANCEL 분기가, 흐름 정책에도 `CANCEL_RECONCILE` 매트릭스가 있었다. 그런데 그걸 구동하는 배선만 죽어 있었다. 기존 결제 대사(`ReconcilePaymentUseCase`)는 APPROVE 타입만 스캔했고(`findStaleApprovePaymentsForReconciliation`), 후처리 대상 판정에 취소 인자를 늘 비운 채(`resolvePostProcessTarget(payment, null, …)`) 불러 `CANCEL_RECONCILE`을 `SKIPPED`로 흘렸다. **standalone CANCEL을 구동하는 경로가 코드상 아예 없었다.** 게다가 이번 CANCEL은 성공(SUCCEEDED)한 승인에 매달리는데 기존 CANCEL은 스캔되는 *APPROVE 실패*에 앵커링돼, 어떤 기존 스캔에도 걸릴 수 없었다. 안전망이 글로만 존재했다. 리뷰 에이전트가 BLOCKER로 짚었고 코드로 재확인해 확정했다. 새 정책·새 PG 로직 없이 CANCEL 타입 stale 후보 스캔 쿼리(`findStaleCancelPaymentsForReconciliation`)와 대사 루프의 CANCEL 처리 분기만 추가해 죽은 정책을 live로 살렸다(기존 취소 상태전이 service·PG 취소 어댑터·승인 이력 조회 재사용).
- **구동 규칙:** stale CANCEL마다 PG에 현재 상태를 물어 — 이미 취소됐으면 SUCCEEDED로 확정, 아직 승인 상태로 남아 있으면 취소 재시도, PENDING·조회불가면 대기 유지. 환불 의도 기록 시점 기준 약 6시간 안 풀리면 승인 대사와 동일하게 escalation 통지로 사람에게 올린다([[결제-escalation-종착통지-escalatedAt-직교필드]]). UNKNOWN 보존·query-before-retry 원칙은 [[payment-unknown-결과불명-처리와-예외분류]], AlreadyCanceled/취소 경로 확장은 [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]].
- **확정적 실패(FAILED)는 통지로만 — 두 근거:** (의미론) FAILED는 "취소 불가 기간·이미 정산·인증 오류"처럼 같은 요청을 재전송해도 안 풀리는 거절이라 반복이 무의미하다. (구조) 아래 4컬럼 unique가 CANCEL 슬롯을 하나만 허용해, FAILED CANCEL이 그 슬롯을 차지하면 같은 결제에 새 CANCEL 재생성이 막혀 전체취소 스코프에선 자동 재시도가 구조적으로 불가능하다. 그래서 조용히 종착시키지 않고 escalation으로 surface해 환불 미집행이 묻히지 않게 한다. 재시도 N회 + 백오프 자동 재처리 엔진은 이번 스코프에서 빼고 별도 이슈(#208)로 분리.
- **트레이드오프:** 대사 스캔이 CANCEL 종류만큼 하나 늘어 PG 조회 부하 증가. 승인 스캔과 동일한 cutoff·페이징 정책으로 통제.

## 환불 멱등 5겹과 4컬럼 unique 하드보장(오진 정정)·H2 parity

- **5겹:** (1) CANCEL 단일 생성, (2) REQUESTED일 때만 PG 호출하는 상태 가드, (3) 결과 불명은 성급히 실패로 보지 않고 UNKNOWN 보존, (4) 대사가 재전송 전 PG에 현재 상태를 먼저 물음(query-before-retry), (5) PG가 이미 취소된 건엔 alreadyCanceled 응답. 이미 성공한 환불 재요청엔 성공 결과를 그대로 반환하도록 실행 경로를 다듬어 재요청 안전성을 마무리했다.
- **정정 — "DB unique 없음"은 오진이었다:** 초안엔 "CANCEL 생성에 DB unique가 없어 주문 행 잠금으로만 직렬화(소프트 보장)"라 적었으나 실제 스키마 확인 결과 틀렸다. 기존 결제 테이블의 unique는 4컬럼 `uk_payment_merchant_pay_key_provider_pg_payment_id_type` — `merchant_pay_key`/`provider`/`pg_payment_id`/`type`. `type`이 키에 들어 한 (merchantPayKey, provider, pgPaymentId)에 APPROVE 행 하나·CANCEL 행 하나만 존재할 수 있다(둘은 type이 달라 안 부딪힘). 즉 전체취소에선 한 결제당 CANCEL이 하나로 **이미 DB가 하드 보장**하고 있었다. 사용자의 "승인 REQUESTED는 중복 생성되나?" 질문을 따라 실제 스키마를 열어보고 발견했다. → 멱등 근거는 추론하지 말고 실제 제약을 확인하라. 이중결제용 approved_order_key NULL 트릭과의 관계는 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]].
- **테스트 parity 문제:** 그 4컬럼 unique는 Flyway(V6, 운영/로컬 MySQL)엔 있으나 Payment 엔티티 `@Table`엔 없었다. 테스트 프로파일은 H2 `create-drop`으로 엔티티에서 스키마를 생성하므로, H2엔 이 제약이 아예 없어 멱등을 테스트로 검증할 수 없는 상태였다. `@Table`에 미러링해 H2에서도 검증되게 했다(운영은 `validate` 모드라 무해). "테스트 통과"가 운영 MySQL 거동을 보장하지 않는 silent zone이라 앞선 같은 결의 문제(#189)와 이어진다([[silent-schema-drift-패턴]]).

## 락 전략 — 단일 행 락 vs join fetch fan-out

- **결정:** 취소 흐름에서 주문을 잠글 때 부모+자식을 한 번에 끌어오는 `select distinct o … join fetch o.orderItems … FOR UPDATE`를 쓰지 않고, 주문 행 하나만 잠그는 단일 행 락(`findByIdAndMemberIdForUpdate`) + orderItems는 aggregate lazy 로드로 분리한다.
- **비관적 락인 이유:** 취소 service는 주문을 FOR UPDATE로 잠가 동시·중복 취소를 직렬화한다(먼저 든 취소가 이기고 둘째는 CANCELED를 보고 거부). 한 tx에서 여러 aggregate를 쓰므로 낙관적 락이면 커밋 시점에야 충돌을 감지해 헛일+롤백이 되지만, 비관적 락은 앞에서 직렬화해 둘째가 헛일을 안 한다.
- **리뷰 지적과 함정:** 리뷰 봇은 `distinct` 제거를 제안했는데, 그건 join fetch의 row fan-out(주문 1건이 자식 아이템 수만큼 복제)을 놓친 제안이라 그대로 따르면 단건 매핑에서 `NonUniqueResultException`을 부른다. 진짜 문제는 따로였다 — join fetch를 락 쿼리에 합치면 락이 자식(order_item) 행까지, 그것도 실행계획·인덱스 순서에 의존해 번져 락 범위가 넓어진다.
- **택한 트레이드오프:** fetch join은 RTT를 1회 아끼지만, 취소는 사용자 단발 동작(hot path 아님)이라 RTT 1회 추가는 미미하고 락 범위를 주문 행 하나로 좁히는 이득(동시성 안전 + 데드락 예방, 돈 정합성 직렬화 락이라 더욱)이 크다. "약간의 RTT < 좁은 락 범위"로 분리를 택했다. 부수 이득으로 distinct/단건 매핑·SQL 생성 거동 의존·자식 락 순서 의존 같은 모호함이 사라져 락이 검증 가능하게 확실해진다(H2·MySQL·Hibernate 버전 무관). 락 전략의 상위 근거(존재=unique, 계산=PK 단일 행)는 [[payment-동시성-unique-vs-lock-gap-lock회피]].

## 응답 계약 — '취소 접수' 기준

취소 API 응답은 조율 service 커밋(주문 CANCELED + 환불 의도 영속화) 시점에 보장되는 "취소 접수"를 기준으로 끊는다. 커밋 후 인라인 best-effort PG 환불이 happy path에서 성공하면 환불 결과까지 담고, UNKNOWN·실패면 "환불 처리중"으로 응답하고 나머지는 대사가 마무리한다. 응답에 환불 진행 상태 필드를 두어 환불 없음(INIT 취소)/완료/처리중 세 값을 구분한다. 취소·환불 보장은 영속된 의도 + 대사가 지므로 사용자가 PG 왕복을 끝까지 기다릴 필요가 없어 @Async·백그라운드 완전 비동기 인프라는 현재 불필요(즉시 응답이 꼭 필요해지면 후속에서 가산적 도입).

## 미해결 — 부분취소·자동 재시도 엔진·데드락 검증

- **부분취소는 범위 밖.** 4컬럼 unique는 전체취소에선 type=CANCEL·provider 고정이라 신원이 (merchantPayKey, pgPaymentId)로 좁혀지지만, 부분취소는 같은 키에 금액만 다른 CANCEL이 여럿이라 이 unique로 표현할 수 없다. amount를 unique에 끼우는 건 답이 아니다(같은 금액 두 번 취소 가능 → amount는 신원이 아님). 올바른 모델은 취소 요청마다 고유 키(승인이 결제 단위 merchantPayKey를 갖듯 취소 요청 단위로 하나) + unique를 그 키에 걸고, "다른 키로 같은 물건 두 번 환불"은 잠금 + "취소액 합 ≤ 승인액 한도 검증"으로 막는 것. 이 취소 요청 키 + 한도 검증 모델은 별도 설계 필요. 모델을 열어둔 결정은 [[payment-부분취소-모델만-열고-구현-보류]].
- **확정적 환불 실패(FAILED)의 자동 재시도 + 백오프 엔진 미도입** — escalation surface까지만, 별도 이슈(#208).
- **fetch join + FOR UPDATE 자식 락 순서·데드락 검증은 후속(#259).** 가설은 order_id 인덱스 순서대로 자식 행이 잠긴다는 것. 다른 인덱스로 order_item에 락을 거는 로직이 생기면 겹치는 행 + 다른 락 순서 + 엇갈린 타이밍 조합에서만 데드락이 성립할 수 있다.

## 근거

- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]
