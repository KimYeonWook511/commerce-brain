---
platform: backend
author: KimYeonWook511
created: 2026-06-18
origin:
  - { type: pr, repo: commerce-backend, ref: 258 }
---

## 한 일
- PAID(결제 완료) 주문을 사용자가 취소하면 전액 환불 + 재고 전체 복구가 일어나도록 빈 흐름을 채움.
  기존엔 INIT(결제 전) 주문만 취소 가능했음(order.cancel()이 INIT 상태만 허용, 환불은 시스템 보상
  경로에만 있어 사용자 주도 환불이 없었음). + 이를 뒷받침하는 standalone CANCEL 결제 대사 신설.
- 정본: docs/tasks/paid-order-cancel-refund/adr.md, 회고 docs/tasks/paid-order-cancel-refund/retrospective.md.
  PR #258. 같은 PR의 harness/AI 운영 교훈은 별도 메모 2026-06-18-pr-258-harness-review-lessons-ai.

## 결정한 것

**환불 보장 — 단일 tx 환불 의도 + 대사 안전망**
- 핵심 문제: PG 취소 호출은 외부 I/O라 주문 취소 트랜잭션 안에 못 넣음. "주문 CANCELED 커밋 → 그 다음
  환불 트리거" 순서면 둘 사이 프로세스가 죽을 때 주문은 취소됐는데 환불 기록이 없는 상태(돈 안 돌려줌).
- 택한 방식: 한 RDB 트랜잭션에 `CANCEL 결제 REQUESTED(환불 의도) + 주문 취소 + 재고 복구`를 원자적으로
  영속화 → 커밋 후 best-effort PG 환불 → 실패·결과불명·중단은 CANCEL 대사가 마무리. 단일 DB라서
  이벤트/Outbox 없이 트랜잭션으로 cross-aggregate 정합 확보.
- 검토한 대안: Outbox 이벤트(주문 트랜잭션에 "환불 필요" 이벤트 원자 기록 + consumer가 환불). 경계는
  더 깨끗하지만 환불 전용 이벤트·consumer 신설이 현재 스코프에 과함. 결합이 커지면 그때 승격(영속된
  CANCEL REQUESTED 행이 이벤트와 같은 역할이라 전환 자연스러움).
- 응답: 취소 접수(트랜잭션 커밋) 시점에 끊고 환불 결과는 best-effort로 담음(완료/처리중). 완전 비동기
  인프라는 현재 불필요.

**append-only 원장 — 환불은 승인을 수정하지 않는다**
- 환불을 "승인 행을 취소됨으로 수정"이 아니라 "별도 CANCEL 레코드 append"로 표현. 승인 성공(SUCCEEDED)
  상태는 그대로 보존.
- 왜: 결제는 사건을 쌓는 불변 원장. 승인 성공과 환불은 별개 사실. 승인 사실 보존이 감사·분쟁·미래
  부분취소에서 일관. "결제취소 했나"는 CANCEL 레코드 존재·상태로 판단.
- 시스템 보상 경로(runPgCancel)는 "애초에 승인되면 안 됐던" 결제를 되돌리느라 승인을 실패로 마킹함.
  사용자 환불은 정당한 승인을 되돌리는 거라 의미가 달라, 그대로 재사용 안 하고 별도 환불 실행 경로를
  둠(보상 경로 회귀 위험 회피).

**standalone CANCEL 대사 신설 (이번 작업의 진짜 핵심)**
- 발견: 초안 설계는 "기존 CANCEL 대사가 영속된 CANCEL REQUESTED를 집어 재시도한다"에 기댔는데, 코드
  확인하니 기존 결제 대사(ReconcilePaymentUseCase)는 승인(APPROVE) 타입만 스캔하고 취소 대사 분기는
  SKIP함. standalone CANCEL을 구동하는 경로가 아예 없었음. 안전망이 글로만 있고 코드엔 없었음.
- 정책 뼈대(후처리 대상/흐름 정책의 CANCEL 분기)는 이미 있고 배선만 죽어 있어서, 스캔 쿼리(CANCEL
  타입 stale 후보)와 대사 루프의 CANCEL 처리만 추가하면 죽은 정책이 살아남. 새 정책·새 PG 로직 없음.
  PG 이력 조회로 이미 취소됐으면 확정/아직 승인 상태면 취소 재시도.
- 확정적 환불 실패(FAILED)는 자동 재시도 안 하고 운영자 통지(escalation)로 넘김. CANCEL이 일정 시간
  (REQUESTED 기준 약 6시간) 초과로 안 풀려도 통지. FAILED를 통지로만 처리하는 근거는 둘:
  (1) 같은 요청 재전송으로 안 풀리는 거절이라(의미론), (2) FAILED CANCEL이 아래 4컬럼 unique 슬롯을
  점유해 같은 결제에 새 CANCEL 재생성이 막혀, 전체취소 스코프에선 자동 재시도가 구조적으로 불가(구조).
  자동 재처리 엔진은 별도 이슈(#208)로 분리.

**환불 멱등 — 5겹, 그리고 이미 하드였음**
- 5겹: (1) CANCEL 단일 생성, (2) REQUESTED일 때만 PG 호출하는 상태 가드, (3) 결과 불명은 UNKNOWN으로
  보존, (4) 대사가 재전송 전에 PG에 현재 상태를 물어봄(query-before-retry — 모르면 다시 보내는 게
  아니라 PG에 조회), (5) PG가 이미 취소된 건엔 alreadyCanceled로 응답.
- 정정: 논의 중 "CANCEL 생성에 DB unique 없어 주문 행 잠금으로만 직렬화(소프트 보장)"라 적었는데
  오진이었음. 기존 결제 테이블 unique는 4컬럼 — merchant_pay_key(어느 주문 결제에서 비롯됐는지
  식별하는 키) / provider(PG사) / pg_payment_id(PG가 발급한 거래 ID) / type(결제 종류 APPROVE 또는
  CANCEL). type이 키에 들어 있어 한 (merchantPayKey, provider, pgPaymentId)에 APPROVE 행 하나·CANCEL
  행 하나만 존재 가능(둘은 type이 달라 안 부딪힘). 즉 전체취소에선 한 결제당 CANCEL이 하나로 이미 하드
  보장이었음. 사용자가 "승인 REQUESTED는 중복 생성되나?" 물어서 실제 스키마 확인하다 발견.
  → 멱등 근거를 추론하지 말고 실제 제약을 확인할 것.
- 테스트 parity: 그 unique가 Flyway 마이그레이션엔 있으나 Payment 엔티티 @Table 선언엔 없어서, H2
  테스트 프로파일(create-drop, 엔티티에서 스키마 생성)엔 제약이 없었음. 엔티티에 미러링해 H2에서도
  멱등이 검증되게 함. "테스트 통과"가 운영 MySQL 거동을 보장 안 하는 silent zone (#189와 같은 결).

**락 전략 — RTT 한 번보다 좁은 락 범위**
- 취소 service가 주문을 FOR UPDATE로 잠가 동시·중복 취소를 직렬화(먼저 든 취소가 이기고, 둘째는
  CANCELED 상태를 보고 거부). 비관적 락인 이유: 한 트랜잭션에 여러 aggregate 쓰기라 낙관적 락은 커밋
  시점에야 충돌을 감지해 헛일+롤백, 비관적은 앞에서 직렬화해 둘째가 헛일 안 함.
- 리뷰에서 `distinct + join fetch + FOR UPDATE` 락 쿼리가 도마에 오름. 리뷰 봇은 "distinct 제거"를
  제안했으나 그건 join fetch의 row 뻥튀기(주문 1건이 자식 아이템 수만큼 행으로 복제됨)를 놓친 것 →
  단건 조회 매핑에서 결과 여러 개 예외(NonUniqueResult). 진짜 문제는 join fetch를 락 쿼리에 합치면
  락이 자식(order_item) 행까지 실행계획·인덱스 순서에 의존해 번져 락 범위가 넓어지는 것.
- 택함: 주문 행만 잠그는 단일 행 락 + 아이템 lazy 로드로 분리. 트레이드오프 = fetch join 한 쿼리는
  DB 왕복(RTT) 한 번을 아끼지만, 취소는 사용자 단발 동작이라 RTT 한 번 추가는 미미한 반면 락 범위를
  주문 행 하나로 좁히는 이득(미래 데드락 예방)이 큼. 게다가 distinct/단건매핑·SQL DISTINCT 생성 거동
  의존·자식 락 순서의 실행계획 의존 같은 모호함이 다 사라져 락이 검증 가능하게 확실해짐(H2·MySQL·
  Hibernate 버전 무관).
- (참고: Hibernate 6은 fetch join용 distinct를 메모리 내 중복제거로 처리해 SQL에 DISTINCT를 안 내보낼
  수 있음 → 그러면 distinct 유지가 맞고 봇 제안이 틀린 게 되지만, 버전·설정 의존이라 돈 걸린 락을 그
  거동에 기대는 게 취약. 그래서 단일 행 락으로 회피.)

**부분취소 미래 설계 (이번엔 제외)**
- 위 4컬럼 unique는 전체취소에선 type=CANCEL·provider가 고정이라 실질 신원이 (merchantPayKey,
  pgPaymentId) 둘로 좁혀짐. 부분취소는 같은 (merchantPayKey, pgPaymentId)에 CANCEL이 여럿(금액만
  다름)이라 이 unique로 표현 불가. amount를 unique에 넣는 건 답 아님(같은 금액 두 번 취소 가능, amount는
  신원이 아님).
- 올바른 모델: 취소 요청마다 고유 키(승인 결제가 결제 단위로 멱등 키 merchantPayKey를 하나 갖듯, 취소
  요청 단위로 하나 — 보통 클라이언트 발급 또는 결정론적 파생) + unique를 그 키에. 단 "동시에 다른 키로
  같은 물건을 두 번 환불"은 멱등키로 못 막음 → 그건 잠금 + "취소액 합 ≤ 승인액 한도 검증"이 막음.
  멱등키=재시도 중복 방지, 잠금+한도=동시 과다취소 방지, 별개 장치라 둘 다 필요.

## 다음 단계
- 부분취소: 위 취소 요청 키 + 한도 검증 모델로 별도 설계.
- 확정적 환불 실패의 자동 재시도+백오프: #208.
- fetch join + FOR UPDATE의 자식 락 순서·데드락 가능성 검증: #259 (가설: order_id 인덱스 순서대로 자식
  행 락. 추후 다른 인덱스로 order_item에 락 거는 로직이 생기면 데드락 가능 — 항상이 아니라 겹치는 행+
  다른 락 순서+엇갈린 타이밍에서 성립).
