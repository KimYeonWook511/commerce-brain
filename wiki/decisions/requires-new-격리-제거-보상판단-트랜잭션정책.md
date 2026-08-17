---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [transaction-boundary, spring, compensation, payment, requires-new, jpa]
created: 2026-06-01
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-01-pr-183-requires-new-cleanup]]"
---

# 결제 보상 판단의 REQUIRES_NEW 격리 제거 — 근거 사라진 방어벨트와 잘못 박힌 정책 문구 정정

## 컨텍스트·문제 — REQUIRES_NEW 격리 명분과 외부 TX 부재

결제 승인 실패 후 PG 보상 취소를 진행할지 판단하는 `PaymentApprovalService.isCompensationRequired`에 `@Transactional(readOnly = true, propagation = REQUIRES_NEW)`가 걸려 있었다(이슈 #160, PR #183). 이 메서드는 "완료된 Payment row가 이미 존재하는가"를 조회(`findPaymentByMerchantPayKey(...).isEmpty()`)해 보상 필요 여부를 판단한다 — false면 이미 완료된 결제가 있으니 취소를 skip. 이 판단 로직 자체는 [[보상판단-payment-존재-lock-대신-db-unique]]에 정리돼 있다.

`REQUIRES_NEW` 격리는 "외부 트랜잭션의 1차 캐시에 오염되지 않고 커밋된 DB 상태만 읽어 race-safe하게 판단한다"는 명분으로 붙어 있었다. 그런데 실제 호출 경로에 격리할 외부 트랜잭션이 존재하지 않았다.

## 결정1 — 격리 제거 (안전 근거·남은 비용)

어노테이션을 `@Transactional(readOnly = true)`로 낮췄다.

- **안전 근거:** 실제 호출 경로는 `NaverPayApprovalService.completeVerifiedApproval`의 catch → `PaymentApprovalCompensationService.runPgCancel` → `isCompensationRequired`인데, 이 경로 어느 클래스·메서드에도 `@Transactional`이 붙어 있지 않다(두 서비스 클래스 모두 0건 확인). 외부 트랜잭션이 없으니 오염될 1차 캐시 자체가 없고, `REQUIRES_NEW`가 지킬 격리 대상이 부재했다. `readOnly = true`만으로 자기 트랜잭션에서 커밋된 상태를 읽어 race-safe 판단은 그대로 성립한다.
- **남은 비용:** 격리 이득이 공집합인데 `REQUIRES_NEW`는 진행 중 TX를 suspend하고 별도 커넥션을 여는 성격이라, 미래에 상위 TX가 끼는 순간 커넥션을 이중 점유해 풀 포화로 이어질 수 있는 **잠복 위험**만 남긴다. 이득 없는 격리를 남겨 둘 이유가 없다.

## 결정2 — 정책 문서 근거를 "단계별 독립 commit"으로 교체

코드보다 더 컸던 문제는, 사라진 방어벨트의 명분이 여러 정책 문서에 "핵심 격리"로 못 박혀 **잘못된 전제를 재생산**하고 있었다는 점이다. 보상 서비스에 클래스 레벨 `@Transactional`을 두지 않는 이유를 문서들이 "`isCompensationRequired`의 `REQUIRES_NEW` 격리를 보존하기 위해"라고 적어 두고 있었다.

- **진짜 근거로 교체:** 클래스 레벨 `@Transactional`을 두지 않는 진짜 이유는 격리가 아니라 **단계별 독립 commit(부분 진행 보존)**이다. 보상 흐름의 각 단계(`failIfRequested`·`isCompensationRequired`·`getOrCreate`·`succeed`/`fail`)가 각자 자기 트랜잭션을 열어 독립 커밋해야, 한 단계가 실패해도 앞서 진행한 부분(상태 전이된 approve attempt, 생성된 cancel attempt)이 보존된다. 단일 트랜잭션으로 묶으면 한 단계 실패가 이전 진행까지 롤백시켜 부분 진행 보존이 불가능하다. race-safe 판단은 이제 `readOnly = true`가 별도로 성립하고 트랜잭션 정책 근거에서 분리했다.
- **정정 대상:** 보상 정책·트랜잭션 정책 정본 결정 문서, 보상 도메인 이관(payment-compensation-to-domain)의 ADR·PRD·아키텍처, payment-attempt-service-split 아키텍처 문서, 그리고 코드 주석까지. 다만 회고·step 스냅샷 문서는 **작업 당시의 불변 기록**이라 사후 수정하지 않고 그대로 뒀다.

이 사건은 "잘못된 근거가 정책 문서에 박혀 재생산되는" stale 패턴의 구체 사례로, 원칙은 [[문서-코드-정합성-개념정본-심볼최소화]]에 정리돼 있다. 보상 흐름 전체 설계는 [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]] 참조.

## 결정3 — 판단 메서드 위치 이동 보류(#182, 동시성 mock 충돌·정책 조립자 패턴)

`isCompensationRequired`를 호출처인 보상 서비스로 옮기는 리팩터를 검토했으나 후속 이슈 #182로 보류했다.

- **택한 이동안:** 보상 서비스의 `private` 메서드로 옮기되, 내부 결제 조회는 **다른 application service를 경유**(`paymentApprovalService.findPaymentByMerchantPayKey(...)`)한다. 결정적 근거는 보상 서비스가 이미 다섯 단계를 전부 다른 application service를 거쳐 수행하는 **"정책 조립자" 패턴**을 갖고 있어, 새 메서드도 그 패턴을 따라야 일관되기 때문. Repository 직접 의존안은 응집도/SRP만 보면 깔끔하지만 다섯 단계 중 하나만 다른 의존 방식이 돼 패턴을 깨므로 기각.
- **왜 이번엔 보류 — 테스트가 막았다:** 동시성 통합 테스트 `NaverPayServiceConcurrencyTest`가 단일 mock으로 `isCompensationRequired`가 **특정 시점에만** false를 반환하도록 제어해 race window를 재현하는데, 이동 후 mock 대상이 `findPaymentByMerchantPayKey`가 되면 이 메서드는 `approve()` 시작 시점에도 보상 시점에도 똑같이 호출돼 두 호출을 시점으로 구분할 수 없다. chained `willReturn`이나 `AtomicReference`+`doAnswer`로 시점 구분하는 대안은 모두 fragile. 테스트 재설계 비용까지 포함해 #182로 넘겼다. 이 동일 테스트의 단언 전략·fragility는 [[동시성-테스트-작성-규칙과-단언-전략]] 참조.

## 판단이 두 번 흔들린 과정·안정화 계기

위치 이동 판단은 세 번 뒤집혔다: (1) 처음 service 경유안을 "변경 영향이 적어서"라는 약한 근거로 택함 → (2) 응집도/SRP만 보고 repository 직접안으로 뒤집음(패턴 일관성 놓침) → (3) 사용자 지적("다른 단계는 다 service 경유인데 이것만 repository 직접이냐?")으로 다시 service 경유안으로 정정. 결정 근거가 "변경 영향" 같은 약한 것에서 "구조 패턴 일관성"이라는 재사용 가능한 기준으로 바뀌었다. 사용자가 "휘둘리지 말고 네 판단으로만 답하라"고 지시한 뒤에야 결론이 안정됐다 — 근거의 강도로 판단을 고정하는 게 흔들림을 막았다.

## 배운 것

- **방어적 안전벨트가 후속 리팩터로 의미를 잃어도, 정책 문서까지 동기화되지 않으면 잘못된 전제가 살아남는다.** 코드만 보지 말고 정책 표현의 근거가 여전히 유효한지 확인해야 한다.
- **Spring Data JPA `SimpleJpaRepository`는 클래스 레벨 `@Transactional(readOnly = true)`를 기본 적용**한다 — Repository 호출 자체가 자기 트랜잭션을 시작한다. self-invocation 한계 우회 시 활용 가능.
- **멀티스레드 mock chaining(호출 시점 의존 반환값 변경)은 fragile하다.** 동시성 통합 테스트는 진짜 race window로 모델링하거나 단위 테스트로 분해해야 한다.
- **"트랜잭션" 용어 모호성:** "보상 트랜잭션"처럼 비즈니스 수준 보상 처리가 `@Transactional` DB 트랜잭션과 같은 단어를 쓰면 혼동을 부른다 — "PG 보상 취소 처리"처럼 명시적으로 쓰기로.

## 근거

- [[raw/sessions/backend/2026-06-01-pr-183-requires-new-cleanup]]
