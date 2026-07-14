---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, concurrency, learning-code, naming-convention, dead-code]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# OrderConcurrencyService — production과 격리한 동시성 학습 코드

## 컨텍스트·문제(동시성 전략 비교 학습)

주문 생성의 동시성을 어떤 락 전략으로 잡을지 학습·비교하기 위해 8가지 전략을 구현했다: `createOrderWithoutLock`, `createOrderWithSynchronized`, `createOrderWithSynchronizedAndTransaction`, `createOrderWithReentrantLockAndTransaction`, `createOrderWithOptimisticLock`, `createOrderWithPessimisticLock`, `createOrderWithPessimisticLockOrdered`, `createOrderWithPessimisticLockBatch`. 이 실험 코드를 production 생성 흐름에 어떻게 둘지가 문제다.

## 결정과 선택 이유(production 격리)

8가지 전략을 별도 `OrderConcurrencyService`로 분리하고 production `createOrder` 흐름과 섞지 않았다. production은 `OrderCreateService` → `OrderCreateProcessor`(`createOrderWithPessimisticLockOrdered`와 동등, 비관락 + productId 정렬 = [[재고차감-동시성-비관락과-productid-정렬]])만 사용한다.

- 실험 코드를 production service에 두면 public API가 비대해지고 호출자가 혼란스럽다.
- 각 락 전략의 동작·문제·트레이드오프를 학습·비교하기 위한 명시적 보존이다. 부하 테스트/벤치마크에서 strategy별 호출도 가능하다.
- production이 채택한 두 전략의 근거는 [[재고차감-동시성-비관락과-productid-정렬]](비관락 정렬)과 [[order-version-낙관락-비관락-공존]](낙관락 공존)에 있고, 여기 보존된 `createOrderWithPessimisticLockOrdered`·`createOrderWithOptimisticLock`이 그 production 버전의 비교 대상이다.

## 현재 사용처(production 0, 테스트 5개)

grep 확정 결과 production 사용처는 0, 테스트만 5개 클래스다:

- `OrderApplicationServiceTest`, `OrderConcurrencyServiceDebugTest`, `OrderConcurrencyServiceTest`, `OrderConcurrencyServiceDeadlockTest`, `OrderConcurrencyServiceDeadlockMysqlTest`.

## 네이밍 원칙의 의도적 예외

`decreaseWithPessimisticLock`처럼 구현 전략을 public 메서드 이름에 노출하지 말자는 stock 도메인 회고 원칙([[ddd-이관-컨벤션-adapter-command-query-네이밍]]의 책임 중심 네이밍)과, `createOrderWithPessimisticLock`/`...Ordered`/`...Batch` 같은 strategy 노출 네이밍이 충돌한다. 하지만 이 service는 실험·비교가 **목적**이라 strategy를 이름으로 드러내는 게 오히려 명확하다. 일관성 원칙의 의도적 예외로, 컨벤션 노트가 금지하는 패턴의 정당한 반례다.

## 미해결·후속(운영 진입 시 문서화 후 제거)

운영 진입 전 단계라 학습 코드를 production에 의도적으로 남긴 임시 보존 상태다. 운영 진입 시 학습 내용을 문서로 옮기고 코드는 제거한다 — 미사용 추상화는 지우고 그 존재·제거 이유는 git history에 맡기는 패턴으로, 사용처 없는 인터페이스 메서드를 지운 [[refreshtokenstore-delete-제거-로그아웃-미구현]]과 동일 원칙이다.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
