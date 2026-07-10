---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 224 }
---

## 한 일

- 결제 후처리(대사/재시도) 결정 정책을 재설계했다 (PR #224가 이슈 #221을 closes). test-side `postprocess` 패키지의 대상선별/실행 정책 + 테스트만 손댔고 production·API·DB는 무변경.
- 배경: 이 결제 도메인엔 `PaymentStatus.UNKNOWN`(PG 결과 불명 = 처리됐는지 모름. 이 status 행이 있으면 그 주문의 재결제를 차단)이 도입돼 있는데, 기존 후처리 정책은 그 이전 모델(결과 불명을 실패코드로 식별) 기준이라 어긋나 있었다.

## 결정한 것

정본: `docs/tasks/postprocess-unknown-redesign/`(이 PR에서 생성), 루트 `docs/adr.md`.

- **대상 식별을 실패코드 열거 → status 중심으로.** 결과 불명이 이제 UNKNOWN status로 보존되니, 실패코드(PG 네트워크오류 등)로 "결과 불명"을 식별하던 분기가 죽었다. → UNKNOWN, 그리고 응답 없이 매달린 stale REQUESTED(요청 보냈으나 성공/실패 미확정)를 대사 대상으로.
- **대사 시작 임계를 NaverPay 승인 가능 시간(10분)에서 파생.** NaverPay는 사용자 인증 후 10분 안에 승인(capture)해야 하고 넘으면 만료된다. 기존 5분 임계는 그 윈도우 한가운데라 *아직 정상 진행 중*인 걸 stale로 오판한다. 그래서 분리했다:
  - UNKNOWN은 짧게(~1분)부터 대사 — 재결제 차단 상태라 빨리 풀어야 하고, 캡처는 됐는데 성공 응답만 유실된 경우 PG 이력조회가 즉시 APPROVED를 줘 바로 복구된다.
  - stale REQUESTED는 윈도우 닫힌 뒤(~15분) — 차단도 아니고 일찍 물어도 "진행 중"만 나온다.
  - 장기 미해소(임계 ~수 시간, 운영 config로 확정 예정)면 수동검토로 승급(escalation).
  - **교훈: 시간 임계는 매직넘버가 아니라 외부 PG 시간 상수에서 파생돼야 false positive(정상인데 stale 오판)와 과차단을 동시에 막는다.**
- **mismatch·환불거절·escalation 초과 → 수동검토 격리.** mismatch = 승인 응답의 가맹점키가 *존재하나 우리가 보낸 키와 다른* 경우(정상 사용자에겐 안 생기는 신호 — 공격 시도 또는 정합성 위반). 자동 대사로 안 풀리니 사람에게 보낸다.
  - 옛 정책엔 mismatch 시 "그 키의 실제 주문이 미결제(INIT)면 보상취소, 결제완료(PAID)면 무시"처럼 관련 주문 상태(PAID/INIT/없음)를 보는 분기와 그 enum이 있었는데, mismatch를 일괄 수동검토로 보내며 그 enum의 사용처가 사라져 제거했다. 주문 상태 결합 판단은 #222로 분리.

## 다음 단계

- #222: 주문 만료 취소 ↔ 결제 UNKNOWN 대사의 타이밍 충돌 (승인이 뒤늦게 SUCCEEDED로 확정됐는데 주문은 이미 만료취소 + 재고복구된 경우 = 돈은 받았는데 줄 주문이 없음).
- #208 배치(Epic): 이 정책을 실제로 돌리는 전달 메커니즘(배치/스케줄러/이벤트)·리포지토리 status 스캔·PG 조회 wiring은 이번 범위 밖.

같은 PR의 보상·예외 부채 메모: [[raw/sessions/backend/2026-06-08-pr-224-payment-compensation-exception-findings]]. AI 운영 교훈: [[raw/sessions/backend/2026-06-08-pr-224-harness-gate-and-review-lessons-ai]].
