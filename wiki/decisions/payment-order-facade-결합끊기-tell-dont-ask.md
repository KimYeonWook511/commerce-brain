---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, order, coupling, facade, tell-dont-ask, saga, compensation, error-code, escalation]
created: 2026-06-10
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-10-pr-237-payment-order-decoupling]]"
  - "[[raw/sessions/backend/2026-06-11-payment-order-facade-decoupling]]"
  - "[[raw/sessions/backend/2026-06-18-pr-262-payment-order-decouple-facade]]"
---

# 결제-주문 결합 끊기 — Tell-Don't-Ask와 facade 조율

방향 제시(#237) → 구체화(06-11) → 구현(#262, #240)까지 세 세션에 걸쳐, 결제 로직이 주문 상태 머신을 대신 돌리던 결합을 끊은 결정이다. 세 시점의 사고 흐름을 한 노트에 모았다.

## 문제 — 결제가 주문 상태머신을 대신 돌림(조합 폭발·판단 중복)

UNKNOWN 결제 대사와 보상 코드를 짜다 결제 로직이 폭발하는 뿌리가 드러났다.

- **조합 폭발의 진짜 원인은 모놀리식이 아니라 결합이다.** 결제 로직 안에서 주문 상태를 직접 조회(`order.getStatus()`)해 분기하면 경우의 수가 "주문 상태 가짓수 × 결제 상태 가짓수"로 곱해진다. 한 도메인 안에 두 상태 머신이 섞여 조합이 터진다.
- **판단 중복.** 대사의 `handleOrderNotCompletable`이 주문을 재조회해 취소됨/PAID/주문 없음/기타 4갈래로 환불 필요를 판단하는데, `order.completePayment()`는 이미 "주문이 INIT일 때만 허용, 아니면 거부"로 같은 판단을 하고 있다. 주문 안에서 한 번 판단한 것을 거부받은 결제 쪽이 되물어 다시 분석한다.

## 방향(#237) — Tell-Don't-Ask·단방향·facade·Saga 보상·이벤트 불필요

이번 PR에선 동작하는 4분기로 두고 근본 개선 원칙만 정해 후속(#240)으로 분리했다.

- **Tell-Don't-Ask.** 결제가 주문 상태를 묻지 말고 `order.completePayment()`로 "반영하라" 시키고, 반영 가능 여부 판단은 주문 도메인 안에 가둔다. 결제는 주문 상태머신을 알 필요가 없어진다.
- **단방향 결합.** 결제→주문 통보 한 방향만 남긴다. 양방향(결제가 주문을 되묻는 쪽)을 끊으면 조합이 절반으로 준다.
- **조율은 facade가.** 여러 도메인을 엮는 흐름(승인 확정 + 거부 시 보상)은 결제 것도 주문 것도 아니라 그 위 조율자(facade)에 둔다. 결합이 facade 한 점에만 격리된다.
- **보상은 판단과 실행을 분리(Saga).** 경합("취소된 주문에 뒤늦은 결제 성공")에서 역할을 셋으로 가른다 — (1) 결제는 "승인 성공" 사실만 확정, (2) 환불 필요 판단은 주문/facade, (3) 환불 실행(PG 취소)은 다시 결제. 사실 확정·정책 판단·실행을 서로 다른 책임으로.
- **이벤트는 아니다(이번엔).** 결합을 끊는 핵심은 조율을 facade로 빼는 것이지 이벤트 자체가 아니다. 모놀리식에선 facade가 각 도메인을 직접 호출하는 것으로 충분하다. 반응자가 여럿(주문+알림+적립…)이거나 MSA로 갈 때가 이벤트 도입 시점. 이벤트 불필요·단일 tx 경계는 [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]] 참조.
- **단서: 일시적 실패는 환불하면 안 된다.** DB 오류 같은 transient 실패를 확정적 거부와 뭉뚱그리면 재시도로 풀릴 상황에 환불을 때려 정상 결제를 날린다. "환불로 종착"과 "재시도"는 예외 종류로 반드시 갈라야 한다.

## 구체화(06-11) — 수용/거부 이진·조율자 위치·환불 판단/실행 분리·order 없음 교정

- **결제 분기 축을 "주문 상태 N개"에서 "수용/거부 이진"으로 못박음.** 결제는 `order.completePayment()` 하나를 시도해 수용/거부만 받는다. 주문 상태가 늘어도 결제 분기는 2개로 고정된다(전날 "단방향이면 절반"을 "이진"으로 확정).
- **조율자를 payment 응용 계층 facade 한 점에.** 흐름의 트리거(PG 승인 응답, 대사 스캔)가 둘 다 결제 쪽에서 시작하니 조율자도 결제 계층에 둔다. payment는 주문을, order는 결제를 모르고 둘 다 아는 건 오직 facade다. 조율 전용 별도 패키지는 엮이는 흐름이 하나뿐이라 YAGNI로 기각. 실시간 승인·대사 두 진입점을 한 코어(시도→수용/거부→확정 or 환불)로 통합하는 게 진짜 이득.
- **환불 "판단"만 facade로, "실행"은 결제가.** order가 결제를 모른 채 스스로 취소/만료하고 payment가 뒷수습하는 비대칭은 단방향 결합의 필연이다(order가 payment를 보게 하면 양방향이라 더 나쁨). 틀렸던 건 비대칭이 아니라 payment가 "환불하나"를 스스로 판단하던 것. 환불 실행은 외부 결제 ID로 PG를 취소하는 일이라 결제만 할 수 있다.
- **order 없음(주문 미존재)은 자동 환불 금지 + escalation 교정.** 전날은 "반영 불가 확정 예외(종결/없음/중복)면 환불 종착"으로 order 없음을 환불에 뭉쳐 넣었는데 교정했다. 주문은 hard delete가 없어 정상 흐름엔 도달 불가급 정합성 오류이고, 원인 불명 상태에서 자동 PG 취소는 또 다른 오류를 부른다. 자동 환불하지 않고 운영자 통지(#238, [[결제-escalation-종착통지-escalatedAt-직교필드]])로 위임(구현에선 FAILED 종착 + 통지).

## 구현(#262) — errorCode 단방향·dead 분기 제거+금전 안전망·Outcome enum+record

`order.getStatus()` 직접 분기를 제거하고 승인 확정 조율을 provider 중립 facade(`ConfirmApprovalUseCase` — tx를 열지 않고 흐름만 조립하는 payment.application UseCase)로 모았다. 실시간 승인(`ApproveNaverPayUseCase`)·대사(`ReconcilePaymentUseCase`)가 이 facade를 공유한다. API·DB 스키마 변경 없는 순수 내부 리팩터.

- **거부 사유를 errorCode로 단방향 전달.** 기존엔 상태 불문 단일 예외(`ORDER_PAID_NOT_ALLOWED`)를 던지고 대사가 재조회해 4분기했다. 이걸 상태별 errorCode로 세분화 — 이미 결제 완료면 `ORDER_ALREADY_PAID`, 취소된 주문이면 `ORDER_CANCELED_FOR_PAYMENT`, 그 외 비-INIT은 `ORDER_INVALID_STATE_FOR_PAYMENT`. facade는 주문을 재조회하지 않고 errorCode만으로 분기한다. "이 결제가 중복인가"는 주문이 아니라 결제에게 묻는다(`existsApprovedByOrderId`, [[payment-완료여부-사실조회-hascompletedpayment-srp]]). 주문 자체 없음은 락 조회에서 `ORDER_NOT_FOUND`로 먼저 나와 facade가 환불 없이 통지+FAILED. 검토한 대안 — enum 반환 + 주문 테이블에 결제 식별자 컬럼 추가 — 는 DB 스키마 변경이 따라와 기각. 트레이드오프는 OrderErrorCode 증가지만, 주문 상태가 늘어도 facade 분기는 errorCode 단위로만 늘고 재조회 결합 자체는 사라진다. errorCode의 예외 경계·번역은 예외 경계/공통예외 클러스터와 이어진다.
- **dead 분기 제거 + 금전 안전망.** 대사가 주문 완료 거부를 받고 주문이 이미 PAID면 "이 결제가 성공 주체면 결제 기록만 SUCCEEDED로 맞춘다"는 경로(`succeedApprovalRecordOnly`)가 있었다. 그 진입 조건 "주문 PAID인데 성공 승인 없음"은 도달 불가임을 코드 흐름 + DB 제약으로 증명했다 — 주문이 PAID 되는 유일 경로는 승인 성공과 한 트랜잭션이고, `uk_payment_approved_order_key`가 두 번째 결제를 `completePayment` 도달 전에 `PAYMENT_DUPLICATE`로 막으므로 그 조건은 모순이다. 그래도 돈이 걸린 경로라 "증명됐으니 삭제"로 끝내지 않고, 만에 하나 도달해도 성공 주체를 잘못 환불하는 사고를 막게 환불 대신 통지+FAILED 안전망을 남겼다. dead 증명·금전 안전망 원칙은 [[도달불가분기-방어금지-불변식테스트-돈정합성-통합테스트]] 참조.

> [!warning] Supersede — 이전 대사 결정 뒤집힘
> 위 dead 분기 제거는, "대사가 비-INIT 주문을 만나면 건너뛰지 말고 종착 전이하되 주문이 PAID인데 성공 승인이 없으면 이 건을 성공 주체로 보고 SUCCEEDED로 맞추라"던 이전 후처리 결정([[결제-후처리-대상식별-status중심-재설계]])을 도달불가로 증명해 폐기한 것이다. 뒤집힌 쪽 노트가 별도로 존재하면 그쪽에 `superseded_by`로 이 노트를 걸어야 한다.

- **Outcome은 sealed interface 대신 enum + nullable record.** 승인 확정 결과를 `Decision` enum(`SUCCEEDED|REJECTED|PROPAGATE`) + nullable 필드(errorCode, cause) record로 표현했다. 코드베이스에 sealed interface 선례가 0건이고 "enum+nullable로 충분하면 그쪽(과한 추상화 지양)"이라는 컨벤션을 따랐다. 정적 팩토리(`succeeded()`/`rejected(code)`/`propagate(cause)`)로 세 결과를 만든다.

보상 취소 트리거 자체는 unique 위반 catch → 보상이라, 그 배선(load-bearing save/flush)은 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]에, CANCEL 보상 실행 경로(`CompensateApprovalUseCase.runPgCancel`)와 사용자 주도 환불의 차이는 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]에 있다.

## provider 중립 facade는 절반만(resolver 후속)

provider별 gateway resolver, 공통 승인 진입 UseCase, PG 결과 정규화 레이어는 이번에 만들지 않기로 명시했다. 결제 provider가 네이버페이 하나뿐이고, 네이버는 `ready→approve(redirect)`·`ALREADY_COMPLETE` 같은 특유 상태머신을 가져 카카오/토스와 다르다. 정규화 경계는 두 번째 provider의 실제 모양을 봐야 제대로 그어진다(가상 provider를 상상해 추상화하면 네이버 한 곳에만 맞는 틀린 경계). 대신 provider 특화 진입점은 그대로 두고 provider 공통 "confirm" 로직만 facade로 추출해, 두 번째 provider가 오면 재사용할 토대(절반)만 미리 깔았다. PG 격리 원칙은 [[payment-order-도메인분리와-pg격리]].

## 미해결 — 통합 경로 미검증·INIT 경합·gateway 추상화

- **gateway resolver·공통 승인 진입 UseCase·PG 결과 정규화는 의도적 범위 밖** — 두 번째 provider 도입 시 후속.
- **통합 경로 미검증** — Docker 통합/배치 테스트는 내부 리팩터라 이번에 미실행. 단위 테스트로 커버했으나 통합 경로가 실제로 도는지는 미검증.
- **INIT 취소와 결제 승인 경합** — 주문은 INIT에서만 취소 가능한데 INIT이 "결제 시작 전"과 "결제 진행 중"을 뭉뚱그려, 취소와 승인이 같은 주문을 두고 경합한다. 이 경합이 "취소된 주문에 뒤늦은 결제 성공 → 환불 보상"의 근원. 막으려 상태를 쪼개지 말고 보상으로 정리하는 방향(외부 비동기 경합의 정공법). PAID 취소는 이후 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]에서 구현.
- UNKNOWN 대사 처리 원칙은 [[payment-unknown-결과불명-처리와-예외분류]], escalation 상태모델은 [[결제-escalation-종착통지-escalatedAt-직교필드]].

## 근거

- [[raw/sessions/backend/2026-06-10-pr-237-payment-order-decoupling]]
- [[raw/sessions/backend/2026-06-11-payment-order-facade-decoupling]]
- [[raw/sessions/backend/2026-06-18-pr-262-payment-order-decouple-facade]]
