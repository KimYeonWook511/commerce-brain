---
platform: backend
author: KimYeonWook511
created: 2026-06-07
origin:
  - { type: pr, repo: commerce-backend, ref: 218 }
---

## 한 일
- 작은 단일 버그 fix(이슈 #206, PR #218)를 무거운 개발 워크플로우 없이 경량 경로(worktree + plan + 직접 구현)로 진행.
- PR에 3개 AI 리뷰어(gemini code assist, codex CLI, claude code review) 결과를 받아 종합.
- 같은 PR의 도메인/설계 결정은 [[raw/sessions/backend/2026-06-07-pr-218-pg-approve-exception-boundary]]에 별도 기록.

## 배운 것
- 다중 AI 리뷰어는 서로 다른 "층위"를 본다: 같은 변경을 두고 gemini는 catch 메커니즘 구현, claude는 catch 타입/순서 정확성, codex는 결제 도메인 안전성(이중결제 발생 경로)을 봤다. claude의 "정확성 문제없음"과 codex의 "이중결제 위험"은 모순이 아니라 codex가 한 층 위였다. → 리뷰가 상반돼 보이면 "어느 층위를 보는가"를 먼저 구분하고 단일 상위 축(여기선 "요청 전송 시점")으로 통합.
- 리뷰어의 심각도를 그대로 수용하지 말 것: codex가 "이중결제 직결"이라 한 경로를 도메인 멱등성으로 재검증하니 직접 이중결제보다 박제(PG는 승인했는데 우리 Payment는 미결제로 남는 정합성 깨짐) 수준이었다. 심각도는 도메인 지식으로 보정.
- 작은 fix에 무거운 워크플로우(task 문서 5종 + 다단계 실행기)는 과함: 1 step짜리 변경은 worktree+plan+직접 구현으로 충분. 워크플로우 스킬은 큰 작업용.
