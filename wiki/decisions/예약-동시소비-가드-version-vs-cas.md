---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [reservation, payment, optimistic-lock, cas, concurrency, exception-handling, transaction-boundary]
created: 2026-06-09
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-09-pr-235-reservation-concurrency-guard]]"
---

# 예약 동시 이중 소비 가드 — @Version vs CAS, 정확성 동등 시 도메인 표현력 우선

PR #235(이슈 #231)는 서로 얽힌 세 결정을 함께 넣었다. 이 노트는 그중 (1) 예약 동시 소비 가드다. 나머지 (2)(3)은 [[주문-이중결제-앞단-진입차단-예약조회-단일화]]로 분리했다.

## 컨텍스트·문제 — 다른 pgPaymentId 예약 경합, 기존 unique 제약으로 못 막음

승인 기록 경로(도메인 `use()`가 예약을 `RESERVED → USED`로 전이)가 메모리 상태 검사만 해서, 같은 예약(같은 merchantPayKey)에 **서로 다른 pgPaymentId 승인 2건**이 소비 커밋 전에 경합하면 둘 다 RESERVED로 읽고 통과 → REQUESTED 결제 2건 → PG 청구 2건이 나간다. 기존 unique 제약은 못 막는다 — 주문당 RESERVED 예약 하나만 강제하는 제약은 예약이 이미 하나뿐이라 안 걸리고, `(merchantPayKey, provider, pgPaymentId, type)` 멱등 unique는 pgPaymentId가 다르면 둘 다 통과시킨다.

## 검토한 대안 — 비관(배제) / CAS / @Version, 정확성 동등

- **비관 락(`SELECT ... FOR UPDATE`)** — 읽기 시점부터 행을 잠가 대기 구간이 길고, 단순 단일조건 상태 전이에 과하다. 일찍 뺐다(같은 결의 판정 [[payment-낙관적-락-도입-왜-비관-아님]]).
- 남은 둘 — **조건부 CAS**(`UPDATE ... WHERE status='RESERVED'`의 영향 행 수)와 **`@Version` 낙관 락**.

핵심 통찰: **둘의 정확성이 동등하다.** "낙관 락은 락을 안 쓴다"는 흔한 오해다 — 읽기는 lock-free지만 UPDATE 시점엔 InnoDB가 행 X-lock을 잡고 그 락은 트랜잭션 끝까지 유지된다. CAS의 `UPDATE ... WHERE status='RESERVED'`도 UPDATE 시 똑같이 X-lock을 잡는다. 그래서 동시 두 요청은 한쪽이 락 대기 → 상대 commit 후 풀린 쪽이 0 rows affected로 차단되며 둘 다 직렬화된다. 차이는 **감지 기준**(version 컬럼 vs `WHERE status`)과 **예외 처리**(JPA 자동 vs affected 행 수 직접 체크)뿐이다.

## 결정 — @Version (도메인 use() 캡슐화·단위 테스트 회귀 방어 보존)

정확성이 동등하므로 판정 기준은 "도메인 표현력·캡슐화를 보존하는 쪽"으로 옮겼고, **`@Version`을 택했다.**

- **핵심 이유:** 도메인 `use()` 메서드 — status와 reserved_key를 한 번에 set하는 캡슐화(결제 도메인 재설계 #205의 NULL 트릭 캡슐화, [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]) — 를 그대로 살릴 수 있다. CAS로 가면 그 두 필드 동시 set 책임이 SQL로 빠지고, "RESERVED일 때만 전이" 규칙이 도메인에서 사라져 단위 테스트로 잡던 회귀 방어가 약해진다.
- **결정 기준:** "정확성이 동등하면 도메인 표현력·캡슐화를 보존하는 쪽." #205의 캡슐화·테스트 방향과 정합적이다.

구현: `PaymentReservation`에 `version`(`@Version`)을 추가(스키마 마이그레이션 V7), 승인 기록 경로를 예약 소비 전용 저장 메서드 `saveUsed`로 통과. 예약당 다른 pgPaymentId 재시도라 cart처럼 retry로 흡수하지 않고([[cart-동시성-낙관락-processor-분리-retry]]와 대조) 진 쪽을 차단한다.

## 단방향 상태 머신 = status가 자연 version, bulk UPDATE와의 궁합

- **reservation은 `RESERVED → USED/EXPIRED` 단방향 상태 머신**이라 `status` 컬럼 자체가 사실상 자연 version 역할을 한다. 그래서 CAS도 충분히 타당했다 — `@Version`을 얹으면 사실상 중복 신호다.
- 반대 논거도 진지하게 봤다: 만료(EXPIRED) 후처리를 배치/스케줄러 1급으로 두면 CAS가 더 일관적이다. 만료는 "가공 없는 대량 조건부 전이"라 bulk UPDATE 한 방이 자연스러운데, `@Version` 테이블을 bulk로 치려면 version bump를 매번 챙겨야(안 챙기면 낙관 락이 그 경로에서 누수) 하는 숨은 규약이 생긴다.
  - > 왜 bulk UPDATE는 version 충돌을 확인 안 하나? bulk는 read-modify-write가 아니라 "내가 읽은 기준 version"이 없다. bulk가 version을 +1 올리는 건 자기 확인용이 아니라 **반대편(엔티티 단위 소비) 트랜잭션의 낙관 락을 동작시키려는 신호**다. reservation은 status가 그 신호 역할을 이미 하니 version이 중복 — CAS가 더 자연스러운 깊은 이유다.
- **다만 현재 만료는 예약 진입 시 lazy하게 단일 행만 처리할 뿐 배치가 없어서 그 부담이 아직 없다.** 그래서 이번엔 `@Version`.

## 트랜잭션 경계 — saveAndFlush 조기 flush로 PG 청구 전 차단(load-bearing)

이 가드가 "PG 청구 전 차단"을 보장하는 건 기존 트랜잭션 경계 설계 덕이다(외부 PG 호출은 트랜잭션 밖, 결제·주문 DB 쓰기는 한 트랜잭션 안 — #205에서 잡은 경계).

- 승인 조율 진입점 `approve()`는 **트랜잭션 없는 orchestration**이다. 그 안에서 승인 기록(`create()`)과 승인 완료(`succeedApproval()`)가 각각 짧은 트랜잭션이고, 그 사이 PG approve 호출은 트랜잭션 밖이다.
- **`saveUsed`의 `saveAndFlush` 조기 flush가 낙관 락 충돌을 `create()` 트랜잭션 안에서 확정한다(load-bearing).** `create()`가 PG 호출보다 앞이라 진 쪽은 별도 처리 없이 자동으로 PG 청구 전에 차단된다. 일반 `save`(지연 flush)였다면 충돌이 커밋 시점(프록시 경계, 메서드 리턴 후)에 터져 `create()` 안에서 못 잡고 변환 안 된 채 전파됐을 것이다. 이 `saveAndFlush` 조기 flush 패턴은 [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]에서 `saveUsed`를 선례로 인용하며 일반화됐다.
- 충돌 시 트랜잭션은 **rollback-only**가 된다 → 진 쪽은 예약 소비도 결제 INSERT도 전부 롤백되어 어느 행도 안 남긴다(원하는 정합성). 그래서 catch 후 같은 트랜잭션에서 추가 DB 쓰기를 하면 `UnexpectedRollbackException`이 나므로 **catch에서는 차단 예외만 던진다(추가 쓰기 금지).**

## catch 위치 = adapter (DB 제약/락 위반의 도메인 예외 번역은 adapter 책임)

`saveUsed`는 진 쪽의 `ObjectOptimisticLockingFailureException`을 infra adapter가 도메인 예외 `PAYMENT_RESERVATION_ALREADY_USED`("이미 다른 승인이 예약을 소비했습니다", 409)로 번역한다. 기존에 "주문당 성공결제 unique 위반을 `saveAndFlush`에서 잡아 도메인 예외로 번역"하는 패턴이 이미 있어(이중결제 탐지 작업), 낙관 락 충돌도 같은 자리에 두는 게 일관적이다 — "DB 제약/락 위반을 도메인 예외로 번역하는 건 adapter 책임". application 유스케이스에서 잡는 안도 있었지만 기존 컨벤션을 택했다.

## 트레이드오프·미해결 — 만료(EXPIRED) 배치 도입 시 CAS 통일 재검토

- 만료 후처리 배치가 도입되면 `@Version`과 bulk UPDATE의 궁합 문제(version bump를 챙겨야 낙관 락 누수 없음)를 다시 마주친다. 그때 reservation 상태 전이 전체를 CAS로 통일할지 재검토한다. 단방향 상태 머신이라 status가 자연 version이므로 CAS가 자연스러운 후보다.
- 다만 CAS 통일은 동시성 코드를 두 번 작성하는 비용이라, 만료 배치가 확정 로드맵이었다면 처음부터 CAS가 나았을 수도 있다.
- **배운 것:** 정확성이 동등한 두 동시성 기법 사이에선 도메인 표현력·캡슐화 보존이 기본값. "낙관 락은 락을 안 쓴다"는 오해를 깔고 비교하지 말 것 — 둘 다 UPDATE 시 X-lock으로 직렬화되고 진 쪽이 0 rows로 차단되는 건 동일하다.

## 근거

- [[raw/sessions/backend/2026-06-09-pr-235-reservation-concurrency-guard]] — 예약 동시 소비 가드(CAS vs @Version), 정확성 동등, saveAndFlush 트랜잭션 경계, adapter 변환(PR #235, 이슈 #231).
