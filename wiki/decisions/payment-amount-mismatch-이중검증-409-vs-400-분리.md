---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, payment-attempt, idempotency, error-code, validation, merchant-pay-key]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# 결제 amount mismatch — 재요청 mismatch(409)와 PG 검증(400) 두 검증 분리

> [!note] 05-29 스냅샷 결정
> 이 결정은 [[payment-도메인-구조-개요]] 스냅샷 시점의 것이다. `merchantPayKey`가 Order에 박힌 구조를 전제로 하므로, Issue #174 경계 재설계([[payment-order-도메인분리와-pg격리]])와 충돌하는 부분이 생기면 그쪽에서 보정된다.

## 컨텍스트·문제(두 mismatch의 혼동)

payment 도메인에는 이름이 거의 같은 두 amount mismatch 검증이 있다. 이름이 비슷해서 하나로 뭉뚱그리기 쉬운데, **사실은 비교 대상·비교 시점·운영 대응이 완전히 다른 검증**이라는 게 이 결정의 근거다. 그래서 ErrorCode와 HTTP status를 의도적으로 분리했다.

## 두 검증의 차이(비교대상·시점·운영대응)

| ErrorCode | HTTP | 비교 대상 | 의미 | 운영 대응 |
|---|---|---|---|---|
| `PAYMENT_ATTEMPT_AMOUNT_MISMATCH` | 409 (`PAYMENT-409-3`) | DB에 기록된 amount ↔ 같은 멱등 키로 온 재요청 amount | 호출자 측 **재요청 mismatch** — 같은 키로 다른 값 | 호출자(클라이언트) 산정 오류 디버깅 |
| `PAYMENT_AMOUNT_MISMATCH` | 400 (`PAYMENT-400-8`) | PG에서 완료된 amount ↔ 사용자 요청 amount | **외부 응답 검증** — 세 원인: PG 문제 / 악의적 사용자 / 우리 서버 산정 오류 | PG 알람 + 보안 모니터링 + 내부 디버깅 |

409는 "같은 멱등 키로 다른 amount를 보냈다"는 호출자 재요청의 문제이고, 400은 "PG가 완료해 돌려준 amount가 우리 요청과 다르다"는 외부 응답 검증이다. 카테고리가 다르므로 코드·status·운영 대응을 갈랐다.

## 검증 위치를 catch로 한정한 이유

멱등 키 `(merchantPayKey, provider, paymentId, type)` 재요청 amount 검증은 `save()` 전에 pre-check(SELECT)를 두지 않고 **catch 한 곳으로 한정**했다. attempt save가 트랜잭션 밖(`NOT_SUPPORTED`)에서 돌아 **save commit 직후 unique 위반이 그 자리에서 잡히므로**, 그 catch에서 amount 비교 + 재조회를 하는 것으로 충분하다. pre-check를 두면 정상 경로마다 SELECT가 한 번씩 더 붙는다. 이는 [[find-first-write-not-check-db-unique-멱등]]의 "정상 멱등은 find로 흡수, unique 위반은 catch에서"라는 패턴의 직접 적용이다.

## 상태 무관 일관 거부

기존 attempt 상태(REQUESTED/FAILED/SUCCEEDED)와 **무관하게** 일관 거부한다. FAILED일 때만 amount 변경을 허용하면 "FAILED면 amount 수정 가능"이라는 *암묵적 규칙*이 생겨 멱등성 계약이 흐려진다. 그래서 상태를 안 따지고 일관되게 4xx로 거부한다. "조용히 진행 대신 명시적 거부"는 [[redis-장애-strict-정책-soft-fail-기각]]의 silent 실패 회피와 같은 정신이다.

**트레이드오프:** 호출자가 잘못된 amount로 재시도하면 즉시 4xx로 실패한다(이전에는 침묵 처리 후 뒤늦게 발견됐다). 명시적 실패가 늦은 발견보다 낫다는 판단.

## merchantPayKey 생명주기와 표현 정정

| 단계 | 동작 | 위치 |
|---|---|---|
| Order 생성 | merchantPayKey 없이 Order 생성 | `OrderCreateProcessor.execute()` → `Order.create` |
| 결제 준비 | `null`이면 `PAY-{ULID}` 발급 후 `order.assignMerchantPayKey()` | `PaymentReadyService.readyPayment()` |
| 결제 승인 | merchantPayKey로 Order 조회 → Payment·PaymentAttempt에 복사 저장 | `NaverPayApprovalService.approve()` |

`Order.assignMerchantPayKey()`는 `merchantPayKey == null`일 때만 set하는 **멱등 setter**다 — **한 Order = 한 merchantPayKey 영구**([[order-도메인-구조-개요]]).

**표현 정정:** 최초 기록은 "amount 변경이 필요하면 새 `merchantPayKey`로 새 요청 발급"이라 적었는데, 코드상 같은 Order에서 merchantPayKey 재발급은 불가(멱등 setter)하다. 정확한 의미는 **amount 변경 = 새 Order 생성 = 새 merchantPayKey**이고, "새 Order로 새 요청 발급"이 옳은 표현이다.

## 미해결

- **Issue #174** — Order에 `merchantPayKey`(unique)가 저장된 구조 = 결제 도메인 식별자가 주문 도메인에 박힘 → 도메인 책임 누수 + "한 Order = 한 PG" 제약. 경계 재설계로 등록. → [[payment-도메인-구조-개요]]·[[order-도메인-구조-개요]]·[[payment-order-도메인분리와-pg격리]]에 걸침.
- **400 세 원인 메트릭 부재** — `PAYMENT_AMOUNT_MISMATCH`(400)에 PG 문제 / 악의적 사용자 / 우리 서버 산정 오류 세 원인이 한 코드에 합쳐져 있다. 운영 시점 구분 위해 별도 ErrorCode 분리나 reason 메타데이터 추가 검토.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
