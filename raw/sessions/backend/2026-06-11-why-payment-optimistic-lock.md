---
platform: backend
author: KimYeonWook511
created: 2026-06-11
---

# Payment 동시 상태 전이 lost update 방어 — 낙관 락 도입 설계 논의

## 배경 / 한 일
#243(Payment 동시 상태 전이 lost update 방어) 구현 전 설계 논의. harness Explore/Discuss 단계까지. 코드/ADR 탐색 후 방향 확정. 아직 구현 전.

## 핵심 문제
- 핵심 엔티티(Order/PaymentReservation/CartItem/Stock)는 모두 @Version(낙관적 락 — 행 버전 컬럼으로 동시 수정 충돌을 감지) 보유. Payment만 없음.
- Payment의 종착 전이(fail/markUnknown — 더 이상 안 바뀌는 최종 상태로의 전이)는 메모리 상태 가드(status가 REQUESTED/UNKNOWN이 아니면 예외)뿐. 같은 payment 행을 두 트랜잭션이 read-modify-write로 동시 전이하면(특히 succeed vs fail) 나중 커밋이 앞을 덮는 lost update 가능 — 돈은 받았는데 FAILED로 기록되는 식.

## 결정한 것

### 1. payment에 @Version 추가 (낙관적 락), 비관 락 아님
- 왜 낙관인가(세션 논의 핵심):
  - payment+order 상태 변경 tx은 매우 짧다(상태 두 번 변경뿐) → 충돌 확률 낮음.
  - 부수효과(메일/알림/포인트 적립)는 결제 성공 commit 후에만 실행해야 함. 이 코드베이스는 이미 "Redis 등 부수효과는 트랜잭션 커밋 후 별도 실행", "외부 PG 호출은 트랜잭션 밖, payment+order DB 쓰기만 짧은 트랜잭션 안" 원칙이 확립돼 있음(정본: docs/adr.md, docs/tasks/payment-order-redesign/adr.md). 그래서 기능이 늘어도 핵심 상태 변경 tx은 구조적으로 짧게 유지됨 → 낙관 락이 계속 적합.
  - 낙관 락도 동시 write 시 먼저 update한 쪽이 행 X-lock, 두 번째는 flush 시점 락 대기 → 앞 tx 짧아 금방 commit → 두 번째는 version 불일치로 즉시 OptimisticLockException(재시도 안 하면 DB connection 빨리 반환).
  - → 이 task는 자동 재시도 루프를 두지 않고, 충돌은 흡수(fail/markUnknown) 또는 전파(succeed)로만 처리한다(상세 4번).
- 비관 락이 유리한 순간 = tx 길이가 아니라 충돌 빈도가 높을 때(예: batch가 같은 행 동시 수정). 긴 tx는 비관에도 독(락 보유 길어짐). 현재 그런 워크로드 없음.

### 2. order의 비관 락(FOR UPDATE)은 유지 (이번 범위 밖)
- succeedApproval은 order를 PK 비관 락(findByIdForUpdate)으로 잠가 같은 주문 동시 승인 반영을 직렬화. 단 payment 행 락은 없음(order 행만 잠금).
- 왜 order는 비관인가: 정본 결정(docs/tasks/payment-order-redesign/adr.md)이 "존재 보장은 unique 제약, 계산 기반 판단(여러 행 합산 후 결정, 예: 부분취소 과다취소 검증)은 Order PK 단일행 FOR UPDATE"로 둠. 또 승인 반영은 PG 과금 후라 충돌 시 재시도가 위험 → 비관 락의 "대기 후 멱등 흡수(재진입 시 이미 SUCCEEDED면 흡수)"가 낙관의 "예외→재시도"보다 깔끔.
- payment @Version과 order 비관 락은 상호보완: order 락은 order 락 잡는 경로끼리만 직렬화. fail은 order 락을 안 잡으므로, order 락으로 못 막는 succeed-vs-fail lost update를 payment @Version이 막는다.
- order를 낙관으로 바꾸는 건 타당한 고민이나 별도 작업(정본 ADR supersede 필요 + 승인 반영 흡수 로직 재작성). #243에 안 끼움. "낙관이 정답"은 현재 단순 상태전이 한정 — 부분취소(합산 검증) 도입되면 비관이 다시 정당.

### 3. escalation: 조건부 UPDATE(CAS) → 도메인 메서드 전환
- 현재 escalation(6시간 초과 — 기존 escalation 정책에서 계승한 값 — 미확정 결제를 운영자에게 위임 표시)은 repository 조건부 UPDATE: UPDATE Payment SET escalatedAt=now WHERE id=? AND escalatedAt IS NULL AND status IN (UNKNOWN,REQUESTED). 영향 행 수=1이면 통지 주체(= 동시 escalation 시도 중 실제로 운영자 통지를 보낼 단 하나의 트랜잭션).
- 이 WHERE의 status 조건·escalatedAt IS NULL은 사실 도메인 규칙(어떤 상태에서 escalation 가능한가, 한 번만)이 SQL로 표현된 것. succeed/fail은 같은 종류 규칙을 엔티티 메서드 가드로 갖는데 escalation만 SQL에 나가 있는 비대칭.
- 왜 SQL이었나: 정본(docs/tasks/payment-escalation/adr.md)이 "payment에 @Version이 없어 메모리 가드로는 동시 race를 못 막음 → DB 레벨 원자성(영향 행 수)으로 멱등 통지"를 보장하려 의도적으로 내린 것.
- @Version이 생기면 그 전제가 사라짐 → escalate()를 도메인 메서드로 올려 네 전이(succeed/fail/markUnknown/escalate)를 모두 엔티티 가드에 모음. 통지 주체 판정: find→escalate()→save 성공=주체→통지, OptimisticLockException=skip. 정본 payment-escalation adr을 supersede.
- 대안(CAS 유지 + version bump): @DynamicUpdate(변경된 컬럼만 UPDATE하는 Hibernate 옵션)가 코드베이스에 없어 엔티티 save가 전체 컬럼 UPDATE → fail() save가 escalatedAt까지 stale 값(null)으로 덮어씀. 그래서 CAS 유지하려면 UPDATE에 version=version+1 수동 추가가 필수(안 그러면 @Version 도입이 오히려 escalation에 새 lost update 유발). 이 "JPA 자동 version을 JPQL에서 수동 bump"라는 비표준성 + 비대칭 유지 때문에 도메인 메서드 선택.

### 4. 충돌 흡수 범위: fail·markUnknown만 흡수, succeed는 전파
- fail/markUnknown은 단조 종착이라 충돌=이미 다른 주체가 종착 → 재시도 아닌 멱등 흡수(skip).
- succeed는 전파: succeed가 졌다=누가 먼저 종착시킴. 상대가 SUCCEEDED면 재호출 시 find 흡수로 자연 처리, FAILED/UNKNOWN이면 "PG 승인(과금)했는데 우리는 실패로 기록"한 모순이라 조용히 흡수하면 돈 문제가 묻힘 → 드러나야 함. 단조 종착 흡수는 "종착시키려는 의도"(fail)에만 맞고 succeed는 "성공시키려는 의도"라 흡수가 의미 왜곡.
- 영향: 기존 동시성 테스트(succeed vs succeed는 order 비관 락으로 직렬화돼 @Version 충돌 안 남)는 통과 유지. succeed vs fail은 신규 시나리오.

## 곁가지로 정리한 것 (범위 밖이지만 이해)
- PaymentRepository.save vs saveApproved 차이: 둘 다 saveAndFlush. saveApproved만 uk_payment_approved_order_key(주문당 SUCCEEDED 1건 보장하는 unique) 위반을 PAYMENT_DUPLICATE(409)로 변환(이중결제 차단을 도메인 결과로 표면화). 일반 save의 멱등 키 unique 위반은 변환 안 하고 안전망(500) 위임(find-first — 전이/생성 전 기존 행을 먼저 조회해 있으면 그대로 반환하는 멱등 흐름 — 가 정상 흐름 흡수, 동시 INSERT 경합은 드문 최종 방어). 409변환/500위임 비대칭 기준은 "클라/운영이 의미 있게 받아야 하나".
- Payment unique 2개: approved_order_key(succeed만 채움) + (merchant_pay_key, provider, pg_payment_id, type) 멱등 키 — 주문·PG 결제 요청을 식별해 중복 생성을 막는 키(모든 생성에서 작동). @Version은 생성(INSERT, 새 row)과 무관, UPDATE 전이만 보호.

## 다음 단계
- task 이름 잠정 payment-optimistic-lock. phase 1개, step 3개 초안(version 추가 / 종착 전이 흡수+동시성 테스트 / escalation 도메인 전환).
- File Drafting에서 확정: succeed OptimisticLockException 전파가 호출자(실시간 승인 NaverPayApprovalService, 대사 PaymentReconciliationService 루프)에서 어떻게 처리되는지 / escalation 건별 트랜잭션 경계·통지 커밋-후 순서 / escalate() 멱등 skip·통지 주체 판정 코드 표현.
- 별도 이슈 후보: order 비관 락을 낙관으로 전환 검토(부분취소 기능 도입 시 재판단).

추적: 이슈 #243.
