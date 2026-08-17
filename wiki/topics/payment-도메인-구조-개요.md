---
type: topic
status: draft
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, payment-attempt, aggregate, domain-model, naverpay, pg-gateway]
created: 2026-05-29
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# payment 도메인 구조 개요 — 두 Aggregate와 PG 연동 경계

> [!warning] 스냅샷 — 이후 두 차례 재설계로 크게 바뀜
> 이 문서는 **2026-05-29 시점의 payment 도메인 스냅샷**이다. 이 시점엔 결제 식별자 `merchantPayKey`가 아직 `Order`에 unique로 박혀 있고, 결제 도메인 경계 재설계(Issue #174) 이전이다. 그래서 `status: draft`.
>
> **1차 재설계(2026-06)** — append-only 원장·예약 테이블 분리·order↔payment 경계 재정의: [[payment-append-only-원장과-exists-완료판단]]·[[payment-완료여부-사실조회-hascompletedpayment-srp]]·[[payment-order-도메인분리와-pg격리]].
>
> **2차 재설계(2026-08, 부분환불)** — 경계를 다시 그었다. 이제 이 문서의 구조 서술 대부분이 유효하지 않다.
> - 예약 테이블이 폐지되고 이중결제 차단이 **결제 행의 활성 슬롯 하나**로 모였다: [[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]·[[배타점유-슬롯-미리잡기-vs-성공시-감지·되돌리기]].
> - **환불이 독립 aggregate**가 되고 한도 판정을 결제가 맡는다: [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]. 한도의 기준은 결제사가 실제로 승인한 금액이다([[한도-기준은-결제사가-실제로-승인한-금액]]).
> - 아래 "application 서비스 5개"는 열둘까지 늘었다가 넷으로 재정렬됐다: [[응용계층-서비스-분할-기준-다른-도메인까지-바꿀-때만]].
> - PG 콜백 인터페이스(`PgCanceller`)가 **결제사마다 어댑터 하나**로 대체됐다: [[결제사-연동타입-인프라-격리와-나가는-호출-읽기제한시간]].
> - 결과 불명 처리·회수 정책이 통째로 바뀌었다: [[외부-돈-호출-결과어휘-넷과-전송계층-판정-우선]]·[[결과불명-재호출은-같은-멱등키로-새키-결론-뒤집힘]]·[[결과회수-상한-폐지와-백오프-표-통지-반복]]·[[승인은-다시-물어-확정-환불에는-실패-종착이-없다]].
> - 결제 어휘와 세 층의 역할은 [[결제사-간편결제-구분과-세-층-역할-결과불명-재시도-모델]]에 따로 정리했다.

## 한 줄 정의

payment 도메인은 **결제 최종 상태(Payment)**와 **승인/취소 시도 이력(PaymentAttempt)** 두 Aggregate + PG provider 서브패키지(`naverpay/`)로 구성되며, 이 도메인을 관통하는 통일 원칙은 **"멱등·동시성을 락이 아니라 DB unique / 데이터 모델로 보장하고, 보상 정책은 결제 도메인 책임으로 둔다"**이다.

## 두 Aggregate(Payment / PaymentAttempt)와 unique 키

- **`Payment`** — 결제 최종 상태. `order_id`·`merchantPayKey`·`pgPaymentId`가 모두 unique. 한 주문당 최대 1건. **row 존재 자체 = "결제 완료"** — 이 사실이 보상 판단의 기준이 된다([[보상판단-payment-존재-lock-대신-db-unique]]).
- **`PaymentAttempt`** — 승인/취소 시도 이력. unique key `(merchant_pay_key, provider, payment_id, type)`. 같은 결제 건에 `type=APPROVE` row와 `type=CANCEL` row가 별도 존재. status는 `REQUESTED → SUCCEEDED | FAILED` 단방향이며, 이 전이는 도메인 메서드에서 엄격 검증한다([[paymentattempt-상태전이-도메인-검증-defensive]]).

## application 서비스 5개 + naverpay

- `PaymentReadyService` — 결제 준비(PG 호출 전 사전 데이터 생성, `merchantPayKey` 발급).
- `PaymentApprovalService` — 결제 완료 반영(`completeApprovedPayment`) + 보상 필요 판단(`isCompensationRequired`). **Payment Aggregate의 owner.**
- `PaymentApprovalAttemptService` — APPROVE attempt 생성·전이(`getOrCreate`·`succeed`·`fail`·`failIfRequested`).
- `PaymentCancellationAttemptService` — CANCEL attempt 생성·전이.
- `PaymentApprovalCompensationService` — 보상 dispatcher 4개 + 공통 골격([[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]).
- `NaverPayApprovalService`(`naverpay.application`) — PG 응답 switch로 메인 흐름 조율. PG cancel은 `this::pgCancel` 메서드 참조로 `PgCanceller` 콜백을 넘긴다.

APPROVE/CANCEL을 전용 service로 나눈 흐름별 분리가 이후 도메인 메서드의 type 가드를 자연 강제해 제거로 이어졌다 — [[paymentattempt-상태전이-도메인-검증-defensive]]의 진화 참조.

## PG 연동 경계(NaverPayGateway 포트·PgCanceller 콜백)

- **`NaverPayGateway`**(`naverpay.application.port`) — PG 호출과 응답 코드 매핑을 담당하는 포트. 구현체 `NaverPayGatewayImpl`(`naverpay.infrastructure`)이 `NaverPayClient` 호출 + `NaverPayException` 처리 + PG 응답 코드 → 도메인 코드 매핑을 수행한다. **Gateway는 application layer에 절대 의존하지 않는다**(역방향 의존 금지) — attempt 상태 반영은 `NaverPayApprovalService`(application)에 남긴다.
- **`PgCanceller`**(`payment.application.port`) — `@FunctionalInterface cancel(PaymentAttempt, String) → CancelOutcome`. payment.application이 PG-specific 타입을 import하지 않게 하는 좁은 콜백 경계. 정책은 PG-agnostic 결제 도메인 책임, PG-specific은 cancel 호출·응답 해석뿐.

## 핵심 승인·보상 흐름

```
NaverPayController
  → NaverPayApprovalService.processApproveAttempt
       (멱등 switch: REQUESTED → PG 호출 / SUCCEEDED → no-op / FAILED → 재시도)
  → NaverPayGateway.approve  (PG 호출 + 응답 코드 매핑)
  → completeVerifiedApproval
       (attempt.verifyApprovedResponse → succeed → completeApprovedPayment)
  → catch 분기 → PaymentApprovalCompensationService.compensate{...}(this::pgCancel)
       → runPgCancel: failIfRequested → isCompensationRequired → getOrCreate(CANCEL) → pgCancel → succeed/fail
```

승인은 멱등 switch로 진입을 흡수하고, 실패 시 catch에서 보상으로 넘어간다. 보상 catch의 2차 예외는 [[보상-catch-2차예외-평탄화-원칙]]으로 평탄화한다.

## 도메인 통일 원칙(lock 대신 unique)

이 도메인의 결정들은 우연히 모인 게 아니라 하나의 통일 원칙에서 나온다: **동시성·멱등을 락이 아니라 DB unique 제약·데이터 모델로 보장한다.** 결제는 빈도가 높아 락 경쟁이 크므로(보안↑ vs 안정성↓ 트레이드오프), unique 전략 + 최종 상태 별도 테이블(Payment)로 갔다. 이 원칙이 [[보상판단-payment-존재-lock-대신-db-unique]]와 [[payment-amount-mismatch-이중검증-409-vs-400-분리]]에 일관되게 흐르며, 코드베이스 전반의 [[find-first-write-not-check-db-unique-멱등]] 패턴, 주문 도메인의 [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]와 같은 결이다.

## 관련 결정 링크

이 topic이 허브로 가리키는 05-29 payment 결정들:

- [[payment-amount-mismatch-이중검증-409-vs-400-분리]] — 재요청 mismatch(409) vs PG 검증(400) 두 검증 분리.
- [[paymentattempt-상태전이-도메인-검증-defensive]] — 상태 전이 도메인 검증, 멱등 자기 전이까지 거부.
- [[paymentattempt-호출정책-문서-javadoc-archunit-보류]] — 호출 정책을 ArchUnit 대신 문서·JavaDoc으로.
- [[보상판단-payment-존재-lock-대신-db-unique]] — 보상 판단 = Payment 존재.
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]] — 보상 정책 이관 + 의미별 dispatcher.
- 코드베이스 knowledge: [[find-first-write-not-check-db-unique-멱등]]·[[보상-catch-2차예외-평탄화-원칙]].
- 인접: [[order-도메인-구조-개요]](merchantPayKey 생명주기·`assignMerchantPayKey` 멱등 setter가 Order 쪽에).

## 열린 질문(Issue #174 경계 재설계)

- **Issue #174** — payment 도메인 Order↔결제 경계 재설계. Order에 `merchantPayKey`(unique)가 박혀 결제 식별자가 주문 도메인에 누수됨 + "한 Order = 한 PG" 제약. 검토 옵션 A(유지)/B(별도 `payment_reference` 테이블)/C(Payment 의미 변경)/D(PaymentAttempt 재사용, 비권장). 이후 [[payment-order-도메인분리와-pg격리]]로 이어진다.
- **Issue #160** — `isCompensationRequired`의 `REQUIRES_NEW`가 격리할 1차 캐시가 실제로 없어 무실효 → 제거. [[requires-new-격리-제거-보상판단-트랜잭션정책]]으로 이어진다.
- **PaymentReference VO화** — `merchantPayKey`가 두 Aggregate 간 협력 키로 String 원시 타입으로 흐름. VO화 시 협력 경계가 타입으로 드러남. Payment 도메인 분리 시 함께 검토.
- **ArchUnit 도입** — [[paymentattempt-호출정책-문서-javadoc-archunit-보류]] 참조. 여러 회고가 동일 제안.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
