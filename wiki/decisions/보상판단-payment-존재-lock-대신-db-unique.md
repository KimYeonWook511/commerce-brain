---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, compensation, concurrency, db-unique, race-condition, transaction-boundary, ddd]
created: 2026-05-29
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
---

# 보상 필요 판단 = Payment 엔티티 존재 — 락 대신 DB unique 통일 원칙

> [!note] 05-29 스냅샷 결정
> [[payment-도메인-구조-개요]] 스냅샷 시점. 이후 완료 판단은 [[payment-완료여부-사실조회-hascompletedpayment-srp]]로 SRP 관점에서 재정리되고, 상태 모델은 [[payment-append-only-원장과-exists-완료판단]]로 진화한다. "Payment 존재 = 완료"라는 이 결정의 핵심은 이어지되 메서드 책임 경계가 보정된다.

## 컨텍스트·문제(attempt.status race-unsafe, Issue #114)

보상 진행 여부를 판단할 때 기존 구조는 `PaymentAttempt.status`로 cancel 진행 여부를 판단했는데, attempt에 row lock이 없어 **race-unsafe**였다.

```
Thread A: completeApprovedPayment
  → succeedApproveAttempt (메모리상 SUCCEEDED)
  → order.completePayment() race throw
  → catch → 보상 진입
       → fail 처리 → 새 status 가드 throw (REQUESTED 아님)
       → PG cancel 흐름 중단
       → PG는 결제 승인됨 + 우리 시스템 미반영 (외부 정합성 깨짐)  ← Issue #114
```

새 상태 전이 가드([[paymentattempt-상태전이-도메인-검증-defensive]])가 보상 catch에서 throw를 유발한 것도 이 문제의 일부다.

## 도메인 통일 원칙(lock 대신 DB unique)

**이 결정은 우연이 아니라 "락을 안 쓰고 DB unique로 동시성을 보장한다"는 이 도메인 통일 원칙의 일관된 적용이다.** 보상 진행 여부를 race-unsafe한 `PaymentAttempt.status` 대신 **완료된 Payment 엔티티 존재**로 판단한다 — `PaymentApprovalService.isCompensationRequired(merchantPayKey)`가 Payment가 이미 존재하면 false를 돌려 cancel을 skip하고 log를 남긴다.

원래는 Payment 테이블 없이 PaymentAttempt만으로 최종 상태까지 표현하려 했다(시도 row를 update해 최종 상태로 전이). 문제 인식: 코드 분기 폭증(시도/완료/취소가 한 row에 섞임) + 동시성/멱등을 위해 lock 필요. 결제는 빈도가 높아 lock 경쟁이 클 것으로 판단(보안↑ vs 안정성↓ 트레이드오프 인지). 해결 방향은 **"lock을 안 쓰는 방법"** — unique 전략 + 최종 상태 별도 테이블(Payment). DB unique 제약이 lock 역할을 자연스럽게 하고, 멱등성·동시성을 lock이 아니라 데이터 모델로 보장한다. [[payment-amount-mismatch-이중검증-409-vs-400-분리]]의 멱등 키 unique도 같은 패턴이다.

## 후보 비교(@Version / FOR UPDATE / Payment 존재)

- **`PaymentAttempt @Version`(낙관적 락)** ✗ — DB 스키마 변경(운영 마이그레이션) + attempt 수준 락 범위가 Order lock과 중첩.
- **`PaymentAttempt FOR UPDATE`(비관적 락)** ✗ — Order FOR UPDATE와의 락 획득 순서 조율 필요, 데드락 가능성.
- **Payment 존재 체크 ✓** — Payment의 `order_id`/`merchantPayKey`/`pgPaymentId`가 모두 unique + `completeApprovedPayment`가 Order FOR UPDATE 안에서 Payment 저장 → **Payment 존재는 DB 레벨 race-safe. 스키마 변경 없음.**

## DDD 관점(두 Aggregate 독립 불변식 협력)

두 별도 Aggregate(`Payment`·`PaymentAttempt`)의 독립 불변식을 cross-Aggregate 협력으로 활용한다. `PaymentApprovalService`가 Payment Aggregate owner이므로 보상 필요 판단을 그쪽에 노출한다 — NaverPay adapter가 `paymentRepository`에 직접 접근하지 않는다(이 캡슐화는 [[보상-catch-2차예외-평탄화-원칙]]의 "예외 안 던지는 의도 캡슐화 메서드"와도 맞물린다). 미래에 Payment 도메인을 분리하면 이 판단이 외부 API로 자연 승격되고 `PaymentApprovalService`가 anti-corruption layer가 된다.

## 자기비판(REQUIRES_NEW 무실효 #160)

`isCompensationRequired`는 `@Transactional(readOnly = true, propagation = REQUIRES_NEW)`로 격리돼 있다(외부 트랜잭션 1차 캐시에 오염되지 않으려는 의도). DDD 학습 후 자기 비판:

- Application 레이어가 JPA 1차 캐시 동작에 의존 — 인프라 디테일을 Application이 알게 됨. DDD 원칙 위반.
- `REQUIRES_NEW`는 새 DB 커넥션을 요구 — 외부 API 호출처럼 경계가 확실히 분리된 경우에만 적합. 단순 1차 캐시 우회용으로 쓰면 커넥션 풀 고갈·데드락 위험.
- **더 깊은 분석:** 호출 경로를 추적하면 `completeVerifiedApproval`(무-트랜잭션) → 실패 시 TX1 롤백으로 1차 캐시 소멸 → `runPgCancel`(무-트랜잭션) → `isCompensationRequired`(외부 TX 없음). 즉 **격리할 외부 1차 캐시가 실제로 존재하지 않아 `REQUIRES_NEW`가 실효조차 없다.**
- **Issue #160** — `REQUIRES_NEW` 제거 + 주석을 비즈니스 의도로 교체. → [[requires-new-격리-제거-보상판단-트랜잭션정책]].

## 미해결·후속

- **Issue #174** — Order↔결제 경계 재설계. [[payment-order-도메인분리와-pg격리]].
- **`isCompensationRequired` cancel skip 빈도 모니터링** — 정상 race 결과이나 빈도가 높으면 결제 흐름 이상 신호. 별도 메트릭 권장.
- **통일 원칙의 전역화** — "lock 대신 DB unique"는 주문 멱등성([[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]])·재고 차감 동시성 판단([[재고차감-동시성-비관락과-productid-정렬]])·[[find-first-write-not-check-db-unique-멱등]]와 강하게 이어지는 전역 테마다. 보상 흐름 설계([[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]])는 이 판단의 인과 사슬 다음 단계.

## 근거

- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
