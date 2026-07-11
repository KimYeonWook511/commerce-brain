---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 235 }
---

# 이중 PG 청구를 청구 도달 전에 줄이는 앞단 가드 — 예약 동시 소비 낙관적 락·주문 기준 진입 차단·예약 조회 단일화

이슈 #231. "이중 PG 청구가 실제 PG 청구까지 **도달하기 전에** 막는 진입·예약 단계 가드"를 보강한 세션이다. 정합성의 최종 보루 — 같은 주문의 두 번째 성공 결제를 DB 유니크(`uk_payment_approved_order_key`) 위반으로 막고 보상으로 정리하는 앞선 작업(#230)의 메커니즘 — 은 그대로 두고, 그 **앞단**에서 청구 도달 빈도를 낮추는 심층 방어다. 서로 얽힌 세 결정을 함께 넣었다: (1) 같은 예약을 두 승인이 동시에 소비하는 경합을 `@Version` 낙관적 락으로 막고, (2) 이미 성공 결제가 있는 주문의 새 승인을 PG 호출 전에 차단하고, (3) 예약 역조회를 회원 기준으로 단일화하면서 예약 미발견을 전용 코드로 분리했다. 여기 적는 건 각 선택의 "왜·검토한 대안·트레이드오프", 그리고 리뷰 후속으로 드러난 자동화 검증 함정이다.

## 결정한 것

### 1. 예약 동시 이중 소비 가드 — CAS와 @Version 사이 (가장 길게 논의)

문제: 승인 기록 경로(도메인 `use()`가 예약을 `RESERVED → USED`로 전이)가 메모리 상태 검사만 해서, 같은 예약(같은 merchantPayKey)에 서로 다른 pgPaymentId 승인 2건이 소비 커밋 전에 경합하면 둘 다 RESERVED로 읽고 통과 → REQUESTED 결제 2건 → PG 청구 2건이 나간다. 이때 다른 유니크 제약들은 이 경합을 못 막는다 — 한 주문에 RESERVED 예약 하나만 강제하는 제약은 예약이 이미 하나뿐이라 걸리지 않고, `(merchantPayKey, provider, pgPaymentId, type)` 유니크는 pgPaymentId가 다르면 둘 다 통과시킨다.

후보는 셋이었다.

- **비관적 락(`SELECT ... FOR UPDATE`)** — 읽기 시점부터 행을 잠가 대기 구간이 길고, 단순한 단일조건 상태 전이에 과하다. 일찍 뺐다.
- 남은 둘, **조건부 CAS**(`UPDATE ... WHERE status='RESERVED'`의 영향 행 수 체크)와 **`@Version` 낙관적 락**으로 좁혔다.

핵심은 **둘의 정확성이 동등**하다는 것이다. "낙관적 락은 락을 안 쓴다"는 흔한 오해다 — 읽기는 lock-free지만 UPDATE 시점엔 InnoDB가 행 X-lock을 잡고 그 락은 트랜잭션 끝까지 유지된다. CAS의 `UPDATE ... WHERE status='RESERVED'`도 UPDATE 시 똑같이 X-lock을 잡는다. 그래서 동시 두 요청은 한쪽이 락 대기 → 상대 commit 후 풀린 쪽이 0 rows affected로 차단되는 것으로 둘 다 직렬화된다. 차이는 **감지 기준**(version 컬럼 vs `WHERE status` 조건)과 **예외 처리**(JPA 자동 vs affected 행 수 직접 체크)뿐이다.

결정적 통찰: **reservation은 `RESERVED → USED/EXPIRED` 단방향 상태 머신**이라 `status` 컬럼 자체가 사실상 자연 version 역할을 한다. 그래서 CAS도 충분히 타당했다. 그럼에도 `@Version`을 택했다.

- **핵심 이유:** 도메인 `use()` 메서드 — status와 reserved_key를 한 번에 set하는 캡슐화(직전 결제 도메인 재설계(#205)에서 도입한 NULL 트릭 캡슐화) — 를 그대로 살릴 수 있다. CAS로 가면 그 두 필드 동시 set 책임이 SQL로 빠지고, "RESERVED일 때만 전이" 규칙이 도메인에서 사라져 단위 테스트로 잡던 회귀 방어가 약해진다.
- **결정 기준:** "정확성이 동등하면 도메인 표현력·캡슐화를 보존하는 쪽." 이 방향은 결제 도메인 재설계(#205)의 캡슐화·테스트와도 정합적이다.

반대 논거도 진지하게 봤다: 만료(EXPIRED) 후처리를 배치/스케줄러로 1급 처리로 두면 CAS가 더 일관적이다. 만료는 "가공 없는 대량 조건부 전이"라 bulk UPDATE 한 방이 자연스러운데, `@Version` 테이블을 bulk로 치려면 version bump를 매번 챙겨야(안 챙기면 낙관적 락이 그 경로에서 누수) 하는 숨은 규약이 생긴다. 소비도 CAS·만료도 CAS면 같은 패턴으로 통일된다. **다만 현재 만료는 예약 진입 시 lazy하게 단일 행만 처리할 뿐 배치가 없어서 그 부담이 아직 없다.** 그래서 이번엔 `@Version`. (배치가 생기면 그때 CAS 통일을 재검토 — 단 그건 동시성 코드를 두 번 작성하는 비용이라, 만료 배치가 확정 로드맵이었다면 처음부터 CAS가 나았을 수도 있다.)

> 왜 bulk UPDATE는 version 충돌을 확인하지 않나? bulk는 read-modify-write가 아니라 "내가 읽은 기준 version"이 없다. bulk가 version을 +1 올리는 건 자기 확인용이 아니라 **반대편(엔티티 단위 소비) 트랜잭션의 낙관적 락을 동작시키려는 신호**다. 그런데 reservation은 status가 그 신호 역할을 이미 하니 version이 중복이다 — 이게 CAS가 더 자연스러운 깊은 이유다.

구현: `PaymentReservation`에 `version`(`@Version`)을 추가하고(스키마 마이그레이션 V7로 컬럼 추가), 승인 기록 경로를 예약 소비 전용 저장 메서드 `saveUsed`로 통과시킨다. `saveUsed`는 `saveAndFlush`로 즉시 flush하고, 진 쪽의 `ObjectOptimisticLockingFailureException`을 infra adapter가 도메인 예외 `PAYMENT_RESERVATION_ALREADY_USED`("이미 다른 승인이 예약을 소비했습니다", 409)로 번역한다. 예약당 다른 pgPaymentId 재시도라 cart 쪽처럼 retry로 흡수하지 않고 진 쪽을 차단한다.

### 2. 트랜잭션 경계 — 왜 진 쪽이 PG 호출 전에 차단되나

이 가드가 "PG 청구 전 차단"을 보장하는 건 기존 트랜잭션 경계 설계 덕이다(외부 PG 호출은 트랜잭션 밖, 결제·주문 DB 쓰기는 한 트랜잭션 안 — 결제 도메인 재설계(#205)에서 잡은 경계).

- 승인 조율 진입점 `approve()`는 **트랜잭션 없는 orchestration**이다(코드상 `@Transactional`이 붙어 있지 않다). 그 안에서 승인 기록(`create()`)과 승인 완료(`succeedApproval()`)가 각각 **짧은 트랜잭션**이고, 그 사이의 **PG approve 호출은 트랜잭션 밖**이다.
- **`saveUsed`의 `saveAndFlush` 조기 flush가 낙관적 락 충돌을 `create()` 트랜잭션 안에서 확정한다(load-bearing).** `create()`가 PG 호출보다 앞이라, 진 쪽은 별도 처리 없이 자동으로 PG 청구 전에 차단된다. 일반 `save`(지연 flush)였다면 충돌이 커밋 시점(프록시 경계, 메서드 리턴 후)에 터져 `create()` 안에서 못 잡고 변환 안 된 채 전파됐을 것이다.
- 충돌 시 트랜잭션은 **rollback-only**가 된다 → 진 쪽은 예약 소비도 결제 INSERT도 전부 롤백되어 어느 행도 안 남긴다(원하는 정합성). 그래서 catch 후 같은 트랜잭션에서 추가 DB 쓰기를 하면 `UnexpectedRollbackException`이 나므로, catch에서는 차단 예외만 던져야 한다(추가 쓰기 금지).
- **catch 위치는 adapter로 뒀다.** 기존에 "주문당 성공결제 유니크 위반을 `saveAndFlush`에서 잡아 도메인 예외로 번역"하는 패턴이 이미 있어(직전 이중결제 탐지 작업), 낙관적 락 충돌도 같은 자리에 두는 게 일관적이다 — "DB 제약/락 위반을 도메인 예외로 번역하는 건 adapter 책임". application 유스케이스에서 잡는 안도 있었지만 기존 컨벤션 쪽을 택했다.

### 3. 주문 기준 진입 사전 차단 — `PAYMENT_DUPLICATE` vs `ALREADY_USED`

진입에서 그동안 UNKNOWN 결제가 걸린 주문만 차단하고 "이미 성공한 승인 결제가 있는 주문"은 차단하지 않아, 한 주문을 다른 예약(다른 merchantPayKey)으로 재결제하는 새 승인이 PG 호출까지 가서 최종 보루(주문당 성공결제 유니크 위반)에서 보상으로 처리됐다. 이를 앞당겨, "그 주문에 이미 성공한(APPROVE·SUCCEEDED) 결제가 있는가"를 EXISTS로 묻는 `existsApprovedByOrderId`를 추가하고, 승인 진입에서 새 승인(RESERVED 신규 승인)을 PG 호출 전에 `PAYMENT_DUPLICATE`("이미 다른 결제가 완료된 주문입니다", 409)로 차단한다. 기존 UNKNOWN 차단(`existsUnknownByOrderId`)과 동형이다.

- 진입 차단 위치는 예약이 이미 USED인 분기(같은 키 redirect 멱등 응답 경로) 이후·`create()` 전에 둔다. USED 예약의 같은 키 멱등 응답을 가로채지 않도록, 진입 차단은 RESERVED 신규 승인에만 적용한다.
- **두 예약 전용 코드의 네이밍 스타일이 달라 보이지만 막는 단위가 다르다.** `ALREADY_USED`는 **예약 단위**(한 예약을 다른 pgPaymentId가 경합 소비), `PAYMENT_DUPLICATE`는 **주문 단위**(주문에 이미 성공 결제 존재)다. 진입 차단은 주문 기준(`existsApprovedByOrderId`)이라 DUPLICATE가 맞다.
- 게다가 최종 보루(주문당 성공결제 유니크 위반 → 같은 `PAYMENT_DUPLICATE`로 번역)와 **같은 "주문 이중 결제" 사실을 PG 호출 전/후 다른 시점에 감지**하는 앞단·뒷단 관계라, 같은 코드를 공유하는 게 일관적이다.

부수적으로 짚어둔 것 — **"RESERVED 예약 + SUCCEEDED 결제 공존"은 어디서 생기나?** 같은 reservation에서는 **발생 불가능**하다. 승인 기록(`create()`)이 예약 소비와 결제 INSERT를 한 트랜잭션에서 원자적으로 하므로, 결제가 존재하면 그 예약은 반드시 USED다. 진입 차단이 실제로 잡는 건 **다른 reservation 간**이다: 주문 O를 예약 R1(USED)로 결제 성공한 뒤, 같은 주문에 새 예약 R2(RESERVED, 다른 merchantPayKey)를 발급해 재결제하는 경우 — orderId를 공유하는 SUCCEEDED 결제가 있어 진입 차단이 동작한다.

### 4. 예약 조회 단일화 + 미발견 전용 코드 분리 (PR 리뷰 후속)

기존 예약 역조회는 merchantPayKey 단독이라 회원 검증이 별도 분기로 남았고, 남의 키가 존재하면 403(`PAYMENT_MEMBER_MISMATCH`)이 반환되어 키 존재 여부가 노출됐다. 또 예약 미발견이 결제(Payment) 미발견과 같은 `PAYMENT_NOT_FOUND`("결제를 찾을 수 없습니다")로 응답되어, 결제창 준비물(Reservation)과 PG 사건(Payment)의 미발견 의미가 섞였다.

- **조회 단일화:** 승인 진입의 역조회를 `findByMemberIdAndMerchantPayKey(memberId, merchantPayKey)`로 단일화해 회원 검증을 조회 조건에 흡수했다. 기존 "키 조회 후 memberId 불일치 시 403" 분기를 제거하고(그 에러코드 `PAYMENT_MEMBER_MISMATCH` 자체를 삭제), 남의 키·없는 키를 모두 예약 미발견으로 흡수한다 — 키 존재를 비노출(보안). 응답 의미 변화(403 → 404)는 의도한 것이다.
- **미발견 코드 분리:** 예약 미발견 전용 `PAYMENT_RESERVATION_NOT_FOUND`("결제 예약을 찾을 수 없습니다", 404)를 신설해 승인 진입 역조회 실패만 교체했다. 이번에 만든 예약 전용 `PAYMENT_RESERVATION_ALREADY_USED`와의 비대칭도 이걸로 해소된다. PG·history 경로의 `PAYMENT_NOT_FOUND`는 그대로 둔다.
- **일반화한 판단:** 도메인 엔티티(Reservation vs Payment)가 다르면 미발견 코드도 분리하는 게 옳다.
- **한 가지 더 일관화한 것(리뷰 지적):** USED 예약에 다른 pgPaymentId로 온 요청이 그동안 `PAYMENT_NOT_FOUND`로 응답됐는데, 이는 실은 "이미 소비된 예약을 다른 pgPaymentId로 재사용하려는 중복 요청"이다. 이 sequential 경로도 `PAYMENT_RESERVATION_ALREADY_USED`로 바꿔, 같은 race(다른 pgPaymentId 재사용)가 동시(`saveUsed` 낙관적 락) 경로와 순차(USED 분기) 경로에서 같은 코드로 일관되게 차단되게 했다.

## 막힌 점·해결

**진입 차단이 동시성 테스트를 깨뜨렸는데 자동 실행이 못 잡았다.** 진입 차단(`PAYMENT_DUPLICATE`)이 "같은 주문 동시 요청"의 race 결과에 새 에러코드를 추가했는데, 그 동시성 테스트가 이를 반영하지 못했다. PR 리뷰 처리 중 전체 동시성 테스트를 돌려서야 드러났다.

- **같은 pgPaymentId 동시 멱등/AlreadyComplete 시나리오:** winner가 성공 commit한 뒤 RESERVED로 읽은 패자가 진입 차단에 걸리는 **현실적 flaky race**. race 허용 목록에 `PAYMENT_DUPLICATE`를 추가해 해결했다.
- **"이미 SUCCEEDED 결제 있으면 동시 요청도 멱등" 시나리오:** setup이 예약 소비(`use()`)를 빼고 결제만 SUCCEEDED로 만든 **비현실적 상태**(같은 예약에선 불가능 — `create()`가 소비+결제를 원자 처리하므로 SUCCEEDED 결제가 있으면 예약은 항상 USED)였다. 진입 차단이 이를 막아 결정적 실패가 났다. → setup을 예약 USED로 현실화해 멱등 경로를 복원했고, 진 쪽은 동시·순차 모두 `PAYMENT_RESERVATION_ALREADY_USED`로 일관 차단됨을 함께 못 박았다.

원인: **진입 차단을 추가한 작업 단계의 수용 기준(AC)에 동시성 테스트가 없었다.** 동시성 테스트는 분리 태스크라 기본 테스트 실행에서 빠지는데, 그걸 그 단계 AC에 안 넣어서 자동 검증이 회귀를 못 잡고 통과해버렸다. (이 자동화 검증 함정의 운영 교훈은 AI 운영 메타로 따로 더 정리했다.)

## 배운 것

- **정확성이 동등한 두 동시성 기법(CAS vs `@Version`) 사이에선 도메인 표현력·캡슐화를 보존하는 쪽이 기본값이 된다.** "낙관적 락은 락을 안 쓴다"는 오해를 깔고 비교하지 말 것 — 둘 다 UPDATE 시 행 X-lock으로 직렬화되고 진 쪽이 0 rows로 차단되는 건 동일하다.
- **단방향 상태 머신에서는 status 컬럼이 자연 version 역할을 한다** — CAS가 자연스러운 후보가 되고, `@Version`을 얹으면 사실상 중복 신호다. bulk UPDATE와의 궁합(version bump 누수)까지 보면 만료 배치가 생길 때 이 중복이 비용으로 돌아온다.
- **새 진입/상태 가드를 추가하는 단계는 수용 기준(AC)에 동시성 테스트를 반드시 포함해야 한다.** 동시성 테스트가 분리 태스크라 기본 테스트에서 빠지므로, 영향 범위를 AC와 대조하지 않으면 회귀가 머지 단계까지 숨는다.
- **도메인 엔티티가 다르면 미발견/차단 에러코드도 분리한다** — Reservation 미발견을 Payment 미발견과 한 코드로 뭉개면 의미가 섞이고 대칭이 깨진다.

## 미해결·열린 질문

- **만료(EXPIRED) 후처리 배치 도입 시:** `@Version`과 bulk UPDATE의 궁합 문제(version bump를 챙겨야 낙관적 락 누수가 안 남)를 다시 마주친다. 그때 reservation 상태 전이 전체를 CAS로 통일할지 재검토한다. 단방향 상태 머신이라 status가 자연 version이므로 CAS가 자연스러운 후보다.
- **진입 차단은 정합성의 대체가 아니라 빈도 저감이다.** 진짜 동시 race는 100% 못 막으므로, 주문당 성공결제 유니크 + 보상이라는 최종 보루(#230)가 계속 필수다.
