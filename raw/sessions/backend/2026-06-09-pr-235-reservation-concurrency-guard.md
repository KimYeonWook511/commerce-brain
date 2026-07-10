---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 235 }
---

# 이중 PG 청구 진입·예약 동시성 가드 (#231)

> 같은 PR(#235)의 AI 운영 교훈은 `[[raw/sessions/backend/2026-06-09-pr-235-harness-concurrency-ac-lessons-ai]]`에 따로 적었다.
> 정본: `docs/tasks/approval-concurrency-guard/adr.md`와 루트 `docs/adr.md`의 아래 세 결정(@Version 예약 가드 / 주문 기준 진입 차단 / 예약 조회 단일화). 아래는 "내가 어떻게 이해했나·다시 본다면".

## 한 일

이슈 #231 해결. "이중 PG 청구가 PG 청구까지 도달하기 **전에** 막는 진입·예약 단계 가드"를 보강했다. 정합성의 최종 보루(같은 주문 두 번째 성공 결제를 DB 유니크로 막고 보상하는 #230 메커니즘)는 그대로 두고, 그 **앞단**에서 청구 도달 빈도를 낮추는 심층 방어다.

- **예약 동시 이중 use 가드**: `PaymentReservation`에 `@Version`(낙관적 락) 추가. 승인 기록 경로를 예약 소비 전용 저장 메서드(`saveUsed` — `saveAndFlush`로 즉시 flush)로 통과시켜, 진 쪽의 낙관적 락 충돌을 adapter가 `PAYMENT_RESERVATION_ALREADY_USED`로 번역.
- **진입 사전 차단**: `existsApprovedByOrderId`(그 주문에 이미 성공한 승인 결제가 있는지 EXISTS)로 새 승인을 PG 호출 전에 `PAYMENT_DUPLICATE`로 차단.
- **조회 단일화**: 예약 역조회를 `(memberId, merchantPayKey)`로 단일화. 남의/없는 키를 모두 `PAYMENT_RESERVATION_NOT_FOUND`로 흡수(키 존재 비노출). 기존 회원 불일치 403 분기 제거.
- PR 리뷰 후속 2건(아래 "막힌 점·후속"에).

## 결정한 것

### CAS vs @Version — 가장 길게 논의한 결정

후보는 셋이었지만 비관적 락(`SELECT ... FOR UPDATE`)은 읽기 시점부터 행을 잠가 대기 구간이 길고 단순 단일조건 전이에 과해 일찍 빼고, CAS와 @Version으로 좁혔다. 핵심은 **둘의 정확성이 동등**하다는 것. 낙관적 락이 "락을 안 쓴다"는 건 오해다 — 읽기는 lock-free지만 UPDATE 시점엔 InnoDB 행 X-lock을 잡고, 그 락은 트랜잭션 끝까지 유지된다. CAS(`UPDATE ... WHERE status='RESERVED'`)도 똑같이 UPDATE 시 X-lock. 그래서 동시 두 요청은 둘 다 한쪽이 락 대기 → commit 후 풀린 쪽이 0 rows affected로 차단된다. 차이는 **감지 기준**(version 컬럼 vs `WHERE status` 조건)과 예외 처리(JPA 자동 vs affected 직접 체크)뿐.

결정적 통찰: **reservation이 `RESERVED → USED/EXPIRED` 단방향 상태 머신**이라 `status` 컬럼 자체가 사실상 자연 version 역할을 한다. 그래서 CAS도 충분히 타당했다. 그럼에도 `@Version`을 택한 이유:
- 도메인 `use()` 메서드(status·reserved_key를 한 번에 set하는 NULL-trick 캡슐화)를 그대로 살릴 수 있다. CAS로 가면 그 두 필드 동시 set 책임이 SQL로 빠지고, "RESERVED일 때만 전이" 규칙이 도메인에서 사라져 단위 테스트 회귀 방어가 약해진다.
- "정확성이 동등하면 도메인 표현력을 보존하는 쪽"이 결정 기준이었다.

반대 논거도 진지하게 봤다: 만료(EXPIRED) 후처리를 배치/스케줄러로 1급으로 두면 CAS가 더 일관적이다. 만료는 "가공 없는 대량 조건부 전이"라 bulk UPDATE 한 방이 자연스러운데, `@Version` 테이블을 bulk로 치려면 version bump를 매번 챙겨야(안 챙기면 낙관적 락이 그 경로에서 누수) 하는 숨은 규약이 생긴다. use도 CAS, 만료도 CAS면 같은 패턴으로 통일된다. **다만 현재 만료는 reserve 진입 시 lazy 단일 행 처리뿐, 배치가 없어서 그 부담이 아직 없다.** 그래서 @Version. (배치가 생기면 그때 CAS로 통일 재검토 — 단 그건 동시성 코드를 두 번 작성하는 비용이라, 만료 배치가 확정 로드맵이면 처음부터 CAS가 나았을 수도. 이번엔 @Version으로 진행.)

> bulk가 version 충돌을 확인하느냐? 안 한다. bulk는 read-modify-write가 아니라 "내가 읽은 기준 version"이 없다. bulk가 version을 +1 올리는 건 자기 확인용이 아니라 **반대편(엔티티 단위 use) 트랜잭션의 낙관적 락을 동작시키려는 신호**다. 근데 reservation은 status가 그 신호 역할을 이미 하니 version이 중복이다 — 이게 CAS가 자연스러운 더 깊은 이유.

### 트랜잭션 경계 — 왜 진 쪽이 PG 호출 전에 차단되나

이 가드가 "PG 청구 전 차단"을 보장하는 건 트랜잭션 경계 덕이다(기존 설계: 외부 PG 호출은 트랜잭션 밖, 결제·주문 DB 쓰기는 한 트랜잭션 안).

- `approve()`는 트랜잭션 없는 orchestration. 그 안에서 승인 기록(`create()`)과 승인 완료(`succeedApproval()`)가 각각 **짧은 @Transactional**이고, 그 사이의 **PG approve 호출은 트랜잭션 밖**이다.
- `saveUsed`의 `saveAndFlush` **조기 flush가 낙관적 락 충돌을 `create()` 트랜잭션 안에서 확정**한다(load-bearing). `create()`가 PG 호출보다 앞이라 진 쪽은 별도 처리 없이 자동으로 PG 청구 전에 차단된다. 일반 `save`(지연 flush)였다면 충돌이 커밋 시점(프록시 경계, 메서드 리턴 후)에 터져 `create()` 안에서 못 잡고 변환 안 된 채 전파됐을 것.
- 충돌 시 트랜잭션은 **rollback-only**가 된다 → 진 쪽은 예약 사용도 결제 INSERT도 전부 롤백, 어느 행도 안 남긴다(원하는 정합성). 그래서 catch 후 같은 트랜잭션에서 추가 DB 쓰기를 하면 `UnexpectedRollbackException`이 나고, 차단 예외만 던져야 한다.
- catch 위치는 **adapter**로 뒀다. 기존에 "주문당 성공결제 유니크 위반을 `saveAndFlush`에서 잡아 도메인 예외로 번역"하는 패턴이 이미 있어, 낙관적 락 충돌도 같은 자리에 두는 게 일관적이다 — "DB 제약/락 위반을 도메인 예외로 번역하는 건 adapter 책임". application 유스케이스에서 잡는 안도 있었지만 기존 컨벤션 쪽을 택했다.

### PAYMENT_DUPLICATE vs ALREADY_USED — 막는 단위가 다르다

진입 차단은 `PAYMENT_DUPLICATE`("이미 다른 결제가 완료된 주문입니다"), 예약 이중 use는 `PAYMENT_RESERVATION_ALREADY_USED`("이미 다른 승인이 예약을 소비했습니다"). 네이밍 스타일이 달라 보이지만 **막는 단위가 다르다**:
- `ALREADY_USED` = **예약 단위** (한 예약을 다른 pgPaymentId가 경합 소비)
- `PAYMENT_DUPLICATE` = **주문 단위** (주문에 이미 성공 결제 존재)

진입 차단은 주문 기준(`existsApprovedByOrderId`)이라 DUPLICATE가 맞고, 게다가 최종 보루(주문당 성공결제 1개를 보장하는 DB 유니크 위반 → 같은 DUPLICATE로 번역)와 **같은 사실을 PG 호출 전/후 다른 시점에 감지**하는 앞단·뒷단 관계라 같은 코드를 쓰는 게 일관적이다.

### "RESERVED 예약 + SUCCEEDED 결제 공존"은 어디서 생기나

같은 reservation에서는 **발생 불가능**하다 — 승인 기록(`create()`)이 예약 사용 + 결제 INSERT를 한 트랜잭션에서 원자적으로 하므로, 결제가 존재하면 그 예약은 반드시 USED다. 진입 차단이 실제로 잡는 건 **다른 reservation 간**이다: 주문 O를 예약 R1(USED)로 결제 성공한 뒤, 같은 주문에 새 예약 R2(RESERVED, 다른 merchantPayKey)를 발급해 재결제하는 경우. orderId를 공유하는 SUCCEEDED 결제가 있어 진입 차단이 동작한다.

### 예약 미발견 = PAYMENT_RESERVATION_NOT_FOUND 분리 (PR 리뷰 후속)

원래 예약 역조회 실패도 `PAYMENT_NOT_FOUND`("결제를 찾을 수 없습니다")를 썼는데, 이건 결제(Payment, PG 사건) 미발견 느낌이라 예약(결제창 준비물) 미발견에 의미가 안 맞았다. 조회 단일화로 남의 키까지 이 코드로 흡수하면서 더 두드러졌고, 이번에 만든 예약 전용 `ALREADY_USED`와도 비대칭이었다. → 예약 미발견 전용 `PAYMENT_RESERVATION_NOT_FOUND`를 신설해 승인 진입 역조회 실패만 교체. PG/history 경로의 `PAYMENT_NOT_FOUND`는 그대로. 도메인 엔티티(Reservation vs Payment)가 다르면 미발견 코드도 분리하는 게 옳다는 사례.

## 막힌 점

**진입 차단이 동시성 테스트를 깨뜨렸는데 자동 실행이 못 잡았다.** 진입 차단(`PAYMENT_DUPLICATE`)이 "같은 주문 동시 요청"의 race 결과에 새 에러코드를 추가했는데, 그 동시성 테스트가 이를 반영 못 했다. PR 리뷰 처리 중 전체 동시성 테스트를 돌려서야 드러났다:

- **같은 pgPaymentId 동시 멱등/AlreadyComplete 테스트들**: winner가 성공 commit한 뒤 RESERVED로 읽은 패자가 진입 차단에 걸리는 **현실적 flaky race**. race 허용 목록에 `PAYMENT_DUPLICATE`를 추가해 해결.
- **"이미 SUCCEEDED 결제 있으면 동시 요청 멱등" 테스트**: setup이 예약 사용(`use()`)을 빼고 결제만 SUCCEEDED로 만든 **비현실적 상태**(같은 예약에선 불가능). 진입 차단이 이를 막아 결정적 실패. → setup을 예약 USED로 현실화해 멱등 경로 복원.

원인: **진입 차단을 추가한 step의 수용 기준(AC)에 동시성 테스트가 없었다.** 동시성 테스트는 분리 태스크라 기본 test 실행에서 빠지는데, 그걸 AC에 안 넣어서 자동 검증이 통과해버렸다. (이 운영 교훈은 AI-메타 파일에 더 적었다.)

## 다음 단계 / 열어둔 것

- **만료(EXPIRED) 후처리 배치 도입 시**: @Version과 bulk UPDATE의 궁합 문제(version bump 챙겨야 누수 안 남)를 다시 마주친다. 그때 reservation 상태 전이 전체를 CAS로 통일할지 재검토. 단방향 상태 머신이라 status가 자연 version이므로 CAS가 자연스러운 후보.
- 진입 차단은 정합성의 대체가 아니라 빈도 저감이다. 진짜 동시 race는 100% 못 막으므로 #230 최종 보루(주문당 성공결제 유니크 + 보상)가 계속 필수.
