---
platform: backend
author: KimYeonWook511
created: 2026-06-12
origin:
  - { type: pr, repo: commerce-backend, ref: 245 }
---

# Payment 낙관 락 충돌 처리 — 설계를 구현·리뷰하며 확정된 것

## 한 일
- 선행 메모 [[raw/sessions/backend/2026-06-12-pr-245-optimistic-lock-conflict-handling]]가 확정한 구조(transition은 트랜잭션 안에서 충돌 전파 / useCase는 트랜잭션 밖에서 skip / adapter saveChecked가 변환)를 실제 구현. saveChecked는 saveAndFlush로 flush를 adapter 프레임 안에서 일으켜 ObjectOptimisticLockingFailureException을 PaymentException(PAYMENT_CONCURRENTLY_MODIFIED)으로 변환하는 신규 저장 경로. 독립 리뷰 + Gemini 인라인 4건(전부 outdated·이미 기각한 REQUIRES_NEW 재제안 → reject) 처리 + 루트 문서 동기화 + 회고까지. 이슈 #243, PR #245.

## 구현·리뷰에서 확정/변경된 것
- **failIfPending과 fail의 병합**: skip 판단을 useCase로 올리니, 사전 상태 체크 후 흡수하던 보상용 failIfPending(미확정이면 FAILED, 아니면 skip)이 무조건 전이 fail과 같은 형태(find orElseThrow → 도메인 fail() → saveChecked)가 되어 하나로 합쳐짐. markUnknownIfRequested(REQUESTED일 때만 UNKNOWN 마킹하던 조건부 메서드)도 markUnknown으로 개명. 도메인 가드가 상태 안 맞으면 PAYMENT_STATUS_TRANSITION_NOT_ALLOWED, 이력 없으면 PAYMENT_RECORD_NOT_FOUND를 던지고, skip은 useCase private 래퍼가 SKIPPABLE 3종(충돌 PAYMENT_CONCURRENTLY_MODIFIED / 가드위반 / 미존재)만 흡수한다.
- **PAYMENT_STATUS_TRANSITION_NOT_ALLOWED 미결 해소**: 설계 메모가 "이 가드 위반 코드(현재 HTTP 500)가 skip 대상 경로에서 부적절할 수 있어 재검토"로 남긴 것. 결론 — 새 코드를 만들 필요 없이, 그 코드를 useCase skip 래퍼의 SKIPPABLE 집합에 넣어 skip 경로에서는 흡수(클라이언트 비노출). 무조건 전이(대사 fail, 승인확정 succeedApproval = payment 전이 + order 완료를 한 트랜잭션에 묶음)가 이미 종착 상태에서 가드 위반을 내면 진짜 버그 신호라 500 전파가 맞다. → "skip 경로면 흡수 / 무조건 경로면 전파".
- **CANCEL succeed/fail 충돌은 흡수 — '전파' 원칙은 APPROVE 종착 한정 (독립 리뷰 발견)**: saveChecked 도입의 부수효과로, 취소(CANCEL) 결제의 succeed/fail 충돌이 이제 PaymentException(PAYMENT_CONCURRENTLY_MODIFIED)이 되어 보상 흐름 runPgCancel(PG 취소 호출 + 결과 기록 묶음)의 기존 catch(PaymentException)에 흡수된다(이전엔 raw DAO 예외라 전파됐음). "succeed·무조건 fail은 전파" 원칙은 사실 승인(APPROVE) 종착 기준이다 — 목적이 "과금됐는데 실패로 기록"되는 모순 방지라서. CANCEL succeed 충돌의 진 쪽은 이미 다른 주체가 같은 CANCEL 레코드를 종착시킨 중복 보상이므로 흡수가 멱등적으로 옳고, 미해소분은 REQUESTED로 남아 CANCEL 대사(아직 미구현)에서 재확정된다. 이 경계를 task ADR에 한 줄 명확화(코드 변경 없음).
- **escalation 환원**: Payment.escalate(now)가 escalation 가능 상태(UNKNOWN/REQUESTED)·미escalation이면 escalatedAt 기록 후 true, 아니면 no-op false를 반환하고, transition(별도 빈)이 그 boolean과 saveChecked 성공으로 통지 주체를 판정한다(충돌 진 쪽은 PAYMENT_CONCURRENTLY_MODIFIED로 skip → 통지 정확히 1회). escalateIfPending(조건부 UPDATE 영향 행 수로 멱등 보장하던 옛 방식)은 완전 제거.
- **결정적 충돌 테스트는 layer를 갈라야 했다**: 설계 메모가 "충돌 진 쪽을 결정적으로 강제해 검증하라"고 했는데, transition이 자기 트랜잭션 안에서 재조회(find)하므로 단일 스레드로 transition 내부 충돌을 결정적으로 만들 수 없다(재조회가 항상 최신 version을 로드). 그래서 둘로 갈라 결정적 검증: (1) adapter saveChecked는 같은 행을 두 번 조회한 detached 복사본 중 하나를 먼저 저장해 version을 bump한 뒤 stale 복사본을 저장 → PAYMENT_CONCURRENTLY_MODIFIED 변환 확인(통합). (2) useCase skip 래퍼는 transition을 mock해 충돌을 던지게 하고 흡수/비-SKIPPABLE 재전파 확인(단위). 실제 race의 lost-update 부재는 기존 동시성 테스트가 담당. 교훈: "결정적 검증"의 위치는 메커니즘 변환 지점(adapter)과 정책 분기 지점(useCase 래퍼)이지 풀 플로우가 아니다.

## 루트 동기화 (Stage 8)
- docs/adr.md append — ADR-050(Payment @Version 낙관 락 도입), ADR-051(충돌을 transition tx 안 전파 + useCase tx 밖 skip + adapter saveChecked 변환), ADR-052(escalation 멱등을 @Version + escalate() 도메인 메서드로 환원). 옛 escalation 멱등 결정(조건부 UPDATE 영향 행 수)에 supersede 노트 추가(직교 필드·통지 1회 정신은 유지).
- docs/db-schema.md — tbl_payment에 version 컬럼(BIGINT NOT NULL DEFAULT 0, @Version) 추가 반영, escalation 멱등 비고를 조건부 UPDATE → @Version + escalate() 기준으로 갱신.
- 미완: docs/exception-strategy.md "낙관 락 충돌 처리" 섹션 정본화는 별도 작업으로 잔류(선행 메모의 다음 단계 그대로).

## 다음 단계
- CANCEL 대사가 미구현이라 CANCEL 전용 동시 충돌 재현 테스트는 별도 Epic으로 위임.
