---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [compensation, exception-handling, saga, convention, external-system]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# 보상 catch의 2차 예외 평탄화 — 예외 안 던지는 의도 캡슐화 메서드

## 한 줄 정의

catch 블록의 2차 작업(보상 트랜잭션·알림 발송)이 또 예외를 던지면 1차 예외가 가려지거나 보상 흐름이 중단되는 문제를, payment 한 곳 임시 처방이 아니라 **일반 원칙**으로 명문화한 것. 외부 시스템 호출 + 보상이 있는 모든 도메인에 적용된다.

## 정책 3단계(1차 ERROR / 캡슐화 / addSuppressed)

1. **catch 진입 즉시 1차 예외를 `log.error`(ERROR)** — 원래 실패 사유를 먼저 보존한다.
2. **2차 작업은 "예외 안 던지는 의도 캡슐화 메서드"로** — 호출처가 try-catch 없이 평탄하게 호출.
3. **그래도 던지면** — 덜 중요하면 `log.warn` + 1차 전파, 치명적이면 `addSuppressed`로 Composite(1차를 마스킹하지 않고 함께 보존).

## 적용 예(failIfRequested, isCompensationRequired)

- **`PaymentApprovalAttemptService.failIfRequested`** — "REQUESTED면 fail 처리, 아니면 skip"이라는 의도를 캡슐화. 호출처(`runPgCancel`)는 try-catch 없이 호출한다. race window에서 attempt가 이미 SUCCEEDED여도 PG cancel은 멈추지 않고 mark만 skip한다.
- **`PaymentApprovalService.isCompensationRequired`** — 보상 진행 여부 판단을 Payment Aggregate owner에 캡슐화. NaverPay adapter가 `paymentRepository`에 직접 접근하지 않는다([[보상판단-payment-존재-lock-대신-db-unique]]).

`runPgCancel` 안에서 `pgCanceller.cancel` 중 `PaymentException`이 나면 log 후 swallow — 원래 승인 실패 예외(1차)가 보상 실패 예외(2차)에 가려지지 않게([[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]).

## 왜 일반 원칙으로 명문화했나(세 동기)

1. **다른 도메인에서도 같은 패턴이 나온다** — 보상 catch 2차 예외는 payment 특화가 아니라 외부 시스템 호출 + 보상 흐름이 있는 모든 도메인(이메일·SMS·외부 API)에 적용된다. 임시 처방으로 두면 매번 다시 발견하고 다시 풀어야 한다.
2. **의도를 문서에 남기는 것 자체가 가치** — 나중에 문제 생겼을 때 "왜 이렇게 짰지?"의 답이 문서에 남는다. 코드만 보면 의도가 안 보이고 주석은 부분적이다.
3. **payment의 복잡성이 기록을 강제했다** — race window·보상 흐름·PG 응답 다양성·멱등·도메인 무결성 검증 등 너무 많은 상황을 고려해야 해 머리로만 기억하기엔 한계가 있다는 인식.

## 진화 경로(임시 처방 → 원칙 → 격하)

- **임시 처방:** 상태 전이 도메인 검증([[paymentattempt-상태전이-도메인-검증-defensive]]) 직후, `failApproveAndCancelApprovedPayment` 내부 `failApprove`를 넓은 try-catch로 감싸는 임시 처방이 들어갔다. catch 범위가 너무 넓어 `PAYMENT_ATTEMPT_NOT_FOUND` 같은 의도치 않은 예외까지 삼키는 문제가 식별됐다.
- **원칙 명문화:** 위 3단계(1차 ERROR / 캡슐화 / addSuppressed)를 일반 원칙으로 명문화.
- **격하:** Payment 존재 체크([[보상판단-payment-존재-lock-대신-db-unique]])로 race window 자체가 축소되며, 임시 try-catch는 보조 방어선으로 격하됐다.

## 관련 링크

- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]·[[보상판단-payment-존재-lock-대신-db-unique]]·[[paymentattempt-상태전이-도메인-검증-defensive]] — 인과 사슬로 직결.
- [[payment-도메인-구조-개요]] — payment 도메인 진입점.
- [[persistence-exception-노출-경계-추상수준]] — addSuppressed/예외 마스킹 회피는 예외 노출 경계와 연결.

## 열린 질문

- 향후 다른 외부 연동 도메인(이메일/SMS)에서 이 원칙을 재사용할 때, `addSuppressed` Composite 전파 vs `log.warn` 격하의 경계 기준을 케이스별로 정리 필요.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
