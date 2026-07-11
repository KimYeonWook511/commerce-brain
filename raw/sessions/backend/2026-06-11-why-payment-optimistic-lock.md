---
platform: backend
author: KimYeonWook511
created: 2026-06-11
---

# Payment 동시 상태 전이 lost update 방어 — 낙관 락 도입 설계

핵심 도메인 엔티티(Order·PaymentReservation·CartItem·Stock)는 모두 `@Version`(낙관적 락 — 행 버전 컬럼으로 동시 수정 충돌을 감지)을 갖는데 `Payment`(PG로 보낸 승인·취소 요청 1건과 그 상태 — REQUESTED에서 SUCCEEDED/FAILED/UNKNOWN으로 전이 — 를 담는 엔티티)만 없다. 그래서 같은 payment 행을 두 트랜잭션이 read-modify-write로 동시 전이하면 — 특히 승인 성공(succeed)과 실패 처리(fail)가 겹치면 — 나중 커밋이 앞을 덮는 lost update가 나서 "돈은 받았는데 FAILED로 기록"되는 사고가 가능하다. 이 세션은 그 방어책을 정하는 구현 전 설계 논의로, 방향은 확정했지만 아직 코드는 안 짠 상태다(이슈 #243).

## 결정한 것

### 1. payment에 `@Version` 추가 (낙관 락), 비관 락 아님

Payment의 종착 전이(fail·markUnknown — 더 이상 안 바뀌는 최종 상태로의 전이)는 지금 메모리 상태 가드(status가 REQUESTED/UNKNOWN이 아니면 예외)뿐이라 동시 전이를 못 막는다. 여기에 `@Version`을 얹어 같은 행 동시 write 충돌을 DB 버전 불일치(`OptimisticLockException`)로 감지하기로 했다.

- **왜 낙관인가(세션 논의의 핵심):**
  - payment+order 상태를 바꾸는 트랜잭션은 매우 짧다(상태를 두 번 바꾸는 게 전부) → 충돌 확률이 낮다. 낮은 충돌·짧은 tx 워크로드는 낙관 락의 전형적 적합 조건이다.
  - 부수효과(메일·알림·포인트 적립·Redis 캐시 갱신)는 결제 성공 커밋 이후에만 실행하고, 외부 PG 호출은 트랜잭션 밖에 두며 payment+order DB 쓰기만 짧은 한 트랜잭션에 묶는다 — 이 두 원칙은 이미 이 코드베이스에 확립돼 있다. 그래서 기능이 늘어도 상태 변경 tx은 구조적으로 계속 짧게 유지된다 → 낙관 락이 앞으로도 적합하게 남는다.
  - 낙관 락도 동시 write 시 먼저 update한 쪽이 행 X-lock을 잡고, 두 번째는 flush 시점에 락을 기다린다. 앞 tx가 짧아 금방 커밋하면 두 번째는 version 불일치로 즉시 `OptimisticLockException`을 받고(재시도를 안 하면 DB 커넥션을 빨리 반환한다), 락을 오래 쥐지 않는다.
- **비관 락이 유리한 순간은 tx 길이가 아니라 충돌 빈도가 높을 때다**(예: 배치가 같은 행을 동시 수정). 긴 tx는 비관 락에도 독이다(락 보유가 길어짐). 지금 그런 워크로드는 없다.
- **재시도 정책:** 이 작업은 자동 재시도 루프를 두지 않는다. 충돌은 흡수(fail·markUnknown) 또는 전파(succeed)로만 처리한다(아래 결정 4).

### 2. order의 비관 락(FOR UPDATE)은 유지 — 이번 범위 밖

- 승인 반영 경로(`succeedApproval`)는 order를 PK 비관 락(`findByIdForUpdate`)으로 잠가 같은 주문의 동시 승인 반영을 직렬화한다. 다만 잠그는 건 order 행뿐이고 payment 행 락은 없다.
- **왜 order는 비관인가:** 결제 도메인 재설계에서 확립한 동시성 원칙 — 존재 보장(없으면 INSERT, 있으면 막기)은 unique 제약으로, 여러 행을 합산해 내리는 계산 기반 판단(예: 부분취소의 과다취소 검증)은 Order PK 단일 행 FOR UPDATE로 — 이 근거다. 또 승인 반영은 PG 과금 이후라 충돌 시 재시도가 위험하다 → 비관 락의 "대기 후 멱등 흡수(재진입 시 이미 SUCCEEDED면 흡수)"가 낙관의 "예외 → 재시도"보다 깔끔하다.
- **둘은 상호보완이다:** order 락은 order 락을 잡는 경로끼리만 직렬화한다. fail은 order 락을 안 잡으므로, order 락으로 못 막는 succeed-vs-fail lost update를 payment `@Version`이 막는다.
- order를 낙관으로 바꾸는 것도 타당한 고민이지만 별도 작업이다(그 재설계 결정을 supersede해야 하고 + 승인 반영의 흡수 로직을 재작성해야 함). #243에는 끼우지 않는다. "낙관이 정답"은 지금의 단순 상태 전이에 한정된 것이고, 부분취소(합산 검증)가 도입되면 비관이 다시 정당해진다.

### 3. escalation: 조건부 UPDATE(CAS) → 도메인 메서드 전환

- **현재 형태:** escalation(6시간 초과 — 자동 대사가 이미 스캔 윈도우 상한으로 쓰던 값을 계승 — 인 미확정 결제를 운영자에게 위임하는 표시)은 repository 조건부 UPDATE로 되어 있다: `UPDATE Payment SET escalatedAt=now WHERE id=? AND escalatedAt IS NULL AND status IN (UNKNOWN,REQUESTED)`. 영향 행 수가 1인 트랜잭션이 통지 주체(동시 escalation 시도 중 실제로 운영자 통지를 보낼 단 하나)다.
- **비대칭:** 이 WHERE의 status 조건과 `escalatedAt IS NULL`은 사실 도메인 규칙(어떤 상태에서 escalation이 가능한가, 한 번만)이 SQL로 표현된 것이다. succeed·fail은 같은 종류의 규칙을 엔티티 메서드 가드로 갖는데 escalation만 SQL로 나가 있다.
- **왜 SQL이었나:** escalation 설계 결정이 "payment에 `@Version`이 없어 메모리 가드로는 동시 race를 못 막는다 → DB 레벨 원자성(영향 행 수)으로 멱등 통지를 보장한다"는 의도로 내린 것이다.
- **전환:** `@Version`이 생기면 그 전제가 사라진다 → `escalate()`를 도메인 메서드로 올려 네 전이(succeed·fail·markUnknown·escalate)를 모두 엔티티 가드에 모은다. 통지 주체 판정은 `find → escalate() → save` 성공이면 주체(통지), `OptimisticLockException`이면 skip. 이는 앞선 escalation 설계 결정(escalation 종착·통지를 새 status 대신 `escalatedAt` 직교 필드로 표현)의 멱등 메커니즘 부분만 supersede하고, `escalatedAt` 직교 필드·status 불변·통지 정신은 그대로 유지한다.
- **검토한 대안 — CAS 유지 + version bump:** `@DynamicUpdate`(변경된 컬럼만 UPDATE하는 Hibernate 옵션)가 이 코드베이스에 없어 엔티티 save는 전체 컬럼을 UPDATE한다 → `fail()` save가 `escalatedAt`까지 stale 값(null)으로 덮어쓴다. 그래서 CAS를 유지하려면 UPDATE에 `version=version+1`을 수동으로 넣어야 한다(안 그러면 `@Version` 도입이 오히려 escalation에 새 lost update를 유발한다). 이 "JPA 자동 version을 JPQL에서 수동으로 bump"라는 비표준성 + 비대칭 유지 때문에 도메인 메서드 쪽을 택했다.

### 4. 충돌 흡수 범위: fail·markUnknown만 흡수, succeed는 전파

- **fail·markUnknown은 단조 종착이라 충돌 = 이미 다른 주체가 종착시킴** → 재시도가 아니라 멱등 흡수(skip)한다.
- **succeed는 전파한다.** succeed가 졌다 = 누가 먼저 종착시켰다는 뜻이다. 상대가 SUCCEEDED면 재호출 시 find 흡수로 자연히 처리되지만, 상대가 FAILED/UNKNOWN이면 "PG는 승인(과금)했는데 우리는 실패로 기록"한 모순이라 조용히 흡수하면 돈 문제가 묻힌다 → 드러나야 한다. 단조 종착 흡수는 "종착시키려는 의도"(fail)에만 맞고, succeed는 "성공시키려는 의도"라 흡수하면 의미가 왜곡된다.
- **영향:** 기존 동시성 테스트(succeed vs succeed는 order 비관 락으로 직렬화돼 `@Version` 충돌이 안 남)는 통과를 유지한다. succeed vs fail은 신규 시나리오다.

## 곁가지로 이해한 것 (범위 밖)

- **`PaymentRepository.save` vs `saveApproved` 차이:** 둘 다 `saveAndFlush`다. `saveApproved`만 `uk_payment_approved_order_key`(주문당 SUCCEEDED 1건을 보장하는 unique) 위반을 `PAYMENT_DUPLICATE`(409)로 변환해 이중결제 차단을 도메인 결과로 표면화한다. 일반 `save`의 멱등 키 unique 위반은 변환하지 않고 안전망(500)에 위임한다(find-first — 전이/생성 전에 기존 행을 먼저 조회해 있으면 그대로 반환하는 멱등 흐름 — 이 정상 흐름을 흡수하고, 동시 INSERT 경합은 드문 최종 방어). 409 변환/500 위임의 비대칭 기준은 "클라이언트·운영이 의미 있게 받아야 하는가"다.
- **Payment의 unique 2개:** `uk_payment_approved_order_key`(succeed만 채움) + `(merchant_pay_key, provider, pg_payment_id, type)` 멱등 키(주문·PG 결제 요청을 식별해 중복 생성을 막는 키, 모든 생성에서 작동). `@Version`은 생성(INSERT, 새 row)과 무관하고 UPDATE 전이만 보호한다.

## 미해결·열린 질문

- **succeed의 `OptimisticLockException` 전파를 호출자가 어떻게 다루는가:** 실시간 승인 경로(`NaverPayApprovalService`)와 대사 루프(`PaymentReconciliationService`) 각각에서 전파된 충돌을 어떻게 처리할지는 구현 상세를 짜면서 확정한다.
- **escalation의 트랜잭션 경계·순서:** escalation 건별 트랜잭션 경계, 통지가 커밋 이후에 오도록 하는 순서, `escalate()` 멱등 skip과 통지 주체 판정의 코드 표현을 구현 상세에서 확정한다.
- **order 비관 락 → 낙관 전환:** 타당한 고민이나 별도 이슈 후보다. 부분취소(합산 검증) 기능이 들어오면 그때 다시 판단한다.
