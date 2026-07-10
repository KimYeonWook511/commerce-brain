---
platform: backend
author: KimYeonWook511
created: 2026-06-11
origin:
  - { type: pr, repo: commerce-backend, ref: 242 }
---

## 한 일
- escalation 종착·통지 구현(#238, PR #242). 6시간 초과 미확정 결제를 운영자에게 통지하고 종착 표시. 통지는 NotificationPort 추상화 + 현재 로그 어댑터(실제 채널은 후속). 엔티티 `escalatedAt`(↔ 컬럼 `escalated_at`) 직교 필드 + 조건부 UPDATE 멱등 + 주문 없음 통지.

## 결정/이해
- escalation 종착 표현: 새 결제 상태 vs `escalatedAt` 직교 필드 vs 별도 테이블 → `escalatedAt`(nullable timestamp) 직교 필드. status는 결제에 일어난 사실(미확정 등)만 담고 "운영자에게 위임됐나"는 직교 축. 새 상태는 "결론 났나/처리됐나"를 한 값에 뭉갠다 — 과거 MANUAL_REVIEW(자동 처리 포기·운영자 수동 확인) 상태를 바로 그 이유(두 축을 한 값에 섞음)로 철회한 선례가 있다. 별도 테이블은 한 번 통지+종착뿐이라 과하다(YAGNI). 정본: docs/adr.md, docs/tasks/payment-escalation/adr.md.
- 멱등(중복 통지 방지): 조건부 UPDATE(`escalated_at IS NULL AND status IN (미확정군)`일 때만 갱신)의 영향 행 수로. 영향 1인 호출만 통지. 메모리 객체 가드(로드한 객체의 escalatedAt==null 검사)는 동시 트랜잭션 race를 못 막으므로 쓰지 않는다.
- escalation 스캔은 기존 자동 대사 스캔(짧은 시간 윈도우)과 별도 쿼리로 두고 대사 루프 말미에 통합. 범위는 승인(APPROVE)만 — 취소(CANCEL) 대사 자체가 아직 미구현이라 escalation도 제외(별도 이슈).
- 주문 없음(order==null): 자동 환불하지 않고(원인 불명 정합성 붕괴 상태의 자동 PG 취소는 위험) 실패 종착 후 통지.
- 시간 기준: 미확정(UNKNOWN)은 PG 응답 받은 시각(responded_at), 요청중(REQUESTED)은 생성 시각(created_at) 기준으로 6시간을 잰다. 응답을 받았는지의 차이.

## 막힌 점/발견
- 멱등 방식이 처음엔 메모리 객체 가드로 설계됐다가 조건부 UPDATE(영향 행 수)로 교정됐다 — 메모리 가드는 동시 race를 못 막기 때문.
- Payment 엔티티에 @Version(낙관 락)이 없다는 걸 발견. Order/PaymentReservation/CartItem/Stock엔 다 있는데 Payment만 빠졌다. 그래서 종착 전이(실패·미확정 마킹)에서 같은 행 동시 수정 시 lost update 가능 → 별도 이슈 #243으로 분리. 발현 조건은 비현실적(succeed/fail이 같은 행에 동시에 갈리려면 서로 다른 PG 응답을 봐야 하고, 단일 인스턴스 운영 전제라 대사 순차)이라 PR #242 머지는 막지 않음. 단 다중 인스턴스면 codex 지적대로 통지 중복이 나므로 #243로 일반화해 둔 것.
- Payment 동시성은 @Version 없이 경로별 개별 메커니즘으로 막아왔다: 결제 생성=예약의 @Version, 이중 성공=주문당 성공 1개 unique, 동시 승인=주문 행 비관 락, escalation=조건부 UPDATE. 통합된 행 단위 보호(@Version)만 빠진 것.
- PR 리뷰: Gemini는 조건부 UPDATE 메서드에 REQUIRES_NEW 전파를 제안 → reject(호출처가 트랜잭션을 안 열어 이미 독립 커밋되므로 커밋 후 통지가 보장됨 + repository에 전파 정책 박는 건 레이어 오염). codex는 주문 없음 통지의 다중 인스턴스 중복 지적 → #243(lost update의 증상)으로 위임. 둘은 다른 축 — Gemini는 트랜잭션 커밋 타이밍, codex는 동시 수정 lost update.

## 배운 것
- 동시성을 "고려했나"가 아니라 "어떤 형태로 고려했나"를 봐야 한다. 중복·이중·진입은 다 막았어도 행 단위 lost update 방어가 한 엔티티에만 빠질 수 있다. @Version 적용 일관성을 점검하라.
- 멱등은 메모리 가드가 아니라 DB 레벨(조건부 UPDATE 영향 행 수, 또는 낙관 락)로 보장해야 동시 race에 안전하다. 메모리 가드는 단일 트랜잭션 재진입만 막는다.

## 다음
- #243 Payment @Version 도입(종착 전이 lost update 방어, 다른 엔티티와 일관성).
- CANCEL escalation은 CANCEL 대사 구현과 묶어 별도 이슈.
- 같은 세션: [[raw/sessions/backend/2026-06-11-payment-order-facade-decoupling]]
