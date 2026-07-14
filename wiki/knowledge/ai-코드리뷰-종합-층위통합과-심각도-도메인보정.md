---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [ai-review, code-review, review-synthesis, severity, harness, process]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-07-pr-218-multi-ai-review-lessons-ai]]"
  - "[[raw/sessions/backend/2026-06-08-pr-226-review-and-scope-lessons-ai]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-harness-review-lessons-ai]]"
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# AI 코드리뷰 종합 — 상반돼 보이면 층위로 통합, 심각도는 도메인으로 보정, blind accept 금지

**한 줄 정의.** 여러 AI 리뷰어(gemini code assist / codex CLI / claude code review / 로컬 multi-agent)를 한 PR에 붙여 종합하며 굳힌 운영 규율 — 리뷰가 상반돼 보이면 심각도 비교 전에 "어느 층위를 보는가"부터 구분해 단일 상위 축으로 통합하고, 리뷰어가 붙인 심각도는 입력 신호일 뿐 도메인 지식으로 재평가하며, 봇 제안은 blind accept하지 않고 근거를 코드로 검증한다.

## 다중 리뷰어는 서로 다른 층위를 본다 — 단일 상위 축으로 통합

PG 결제 승인 경로의 `catch (Exception)` 범위를 좁힌 작은 fix(PR #218)에서, 같은 변경 하나를 두고 세 리뷰어가 각기 다른 층을 봤다.

- **gemini** — catch 메커니즘의 *구현*: 어떻게 잡고 어떻게 변환하는가.
- **claude** — catch 타입/순서의 *정확성*: 어떤 예외를 어떤 순서로 잡는지가 코드상 맞는가.
- **codex** — 결제 *도메인 안전성*: 이 catch가 이중결제를 부를 수 있는 경로인가.

claude의 "정확성 문제없음"과 codex의 "이중결제 위험"은 겉보기엔 모순이지만, 실은 codex가 **한 층 위(도메인 결과)를 본 것**이라 양립한다. 핵심은 심각도를 비교하기 전에 "어느 층위를 보는가"부터 구분하고, 그다음 단일 상위 축으로 통합하는 것이다. 여기서 그 상위 축은 "요청 전송 시점"이었다 — 전송 전 버그는 PG 부작용이 0이라 전파해도 안전(claude의 정확성 관점), 전송 후/불명은 PG가 이미 처리했을 수 있어 UNKNOWN으로 보존(codex의 도메인 안전성 관점). 두 관점이 하나의 경계 위에서 자리를 찾았다. 그 경계의 도메인 결정 본체는 [[pg-승인-예외-경계-요청전송시점]]에 있다.

## 심각도는 그대로 승격 말고 도메인 지식으로 보정

codex가 "이중결제 직결"로 심각도를 매긴 경로(PG가 `AlreadyComplete`로 응답한 뒤 결제내역을 재확인하는 갈래)를 도메인 멱등성으로 재검증하니 실제 위험은 낮았다. `AlreadyComplete`는 PG의 멱등 반환이라 같은 결제가 두 번 청구되는 게 아니라, PG는 승인했는데 우리 Payment는 미결제로 남는 **정합성 깨짐("박제")** 수준이었다.

- 심각도를 내렸다고 무시한 건 아니다. 이 경로는 이번 PR에서 처리하지 않고 후속 #219로 분리해 같은 "전송 후/불명 → UNKNOWN 보존" 원칙을 그쪽에도 적용하기로 했다(→ [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]).
- **규율:** AI 리뷰어가 붙인 심각도("직결", "critical")는 입력 신호일 뿐, 그대로 우선순위로 승격하지 말고 도메인 지식으로 재평가해 보정한다. 이때 보정 결과가 종종 후속 이슈 분리로 이어진다 — 심각도 판정과 [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]]가 맞물리는 지점이다.

## 독립 리뷰의 수렴은 강한 신호

PR #226에서 이중결제 판정 함수(`isApprovedOrderKeyViolation`)의 short-circuit 결함을, 로컬 multi-agent 리뷰(finder 에이전트를 fan-out 한 뒤 검증 단계를 거치는 자체 도구)와 Gemini Code Assist가 **서로 모른 채 같은 지점**을 짚었다. 서로 독립인 두 리뷰가 같은 결론이면 그 지적은 신뢰도가 높다. 다만 수렴이 곧 정답 코드는 아니어서, 이 버그는 세 번 반복 끝에 수렴했고 전환점은 사용자가 "그러면 또 false 되는 거 아니냐"고 같은 함정을 다시 짚어준 것이었다(수정 변천의 최종형은 [[sql-translator-빈-제거-제약명-이중결제-식별]]·[[결제승인완료-보상-완료우선-이중결제-adapter매핑]]).

## 리뷰 봇 제안 blind accept 금지 — 근거를 코드로 검증

리뷰 봇 제안은 근거를 코드로 검증하고, 틀리면 봇이 시킨 대로가 아니라 다른 해법으로 고친다.

- **PR #258** — 취소 흐름에서 주문을 잠그는 `distinct + join fetch + FOR UPDATE` 쿼리에 봇이 "distinct 제거"를 제안했다. 그대로 적용하면 join fetch의 row 뻥튀기를 단건 매핑이 못 받아 예외가 나는 버그였다. LLM 리뷰가 쿼리의 한 부분(distinct)만 보고 다른 부분(join fetch)의 효과를 놓치는 실수를 사람처럼 한다. 대응은 distinct 제거가 아니라 주문 행 하나만 잠그고 아이템은 별도로 로드하는 2단계 분리였다(락 범위를 좁히는 게 진짜 문제였고 봇 제안은 증상만 건드렸다 — [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]], [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]]).
- **PR #226** — "폴백은 도달하지 않는 보험이니 지우자"고 코드만 보고 판단해 폴백을 제거했더니 MySQL 통합 테스트가 깨졌다. 실제로는 프레임워크의 예외 변환기(`JpaConfig`의 translator 빈)가 cause 체인을 재조립해 "죽었다고 믿은 1차"가 아니라 "보험"이라던 폴백이 진짜 주 경로였다. 예외 변환처럼 프레임워크 내부에서 타입·cause 체인이 재조립되는 동작은 코드 추론이 빗나가기 쉬우니, 돈이 걸린 경로에서는 통합 테스트로 실제 런타임 경로를 드러내고 지운다.

### 사람의 도메인 인사이트가 추상적 리뷰보다 강할 수 있다

cart 도입(PR #166)에서 가장 강력한 개선(DELETE를 bulk 쿼리에서 entity 경유 삭제로 바꿔 `@Version` 체크·race 처리·`@Modifying` 옵션 부담을 한 번에 푼 것)은 사용자의 한 마디 — "find해왔으면 해당 entity를 delete에 넘기면 되잖아 왜 벌크쿼리로 하는거야?" — 에서 시작됐다. LLM은 옵션 비교("`clearAutomatically`를 붙일까")로 접근하는데 사용자는 "왜 굳이 그렇게 짰지?"라는 도메인 질문으로 접근한다. 같은 세션이 만든 결정을 같은 세션이 검토하면 편향이 있어(자체 검토는 stale identifier·중복 표현 drift·unreachable 분기를 뒤늦게 잡는다) 외부/독립 검토가 stale 발견에 강하다.

### 즉시 동조(sycophancy)를 눌러야 트레이드오프가 남는다

cart 동시성에서 사용자가 "낙관적 락이 베스트같구만" 하자 즉시 "맞습니다" 모드로 진입했고, 사용자가 "내가 그렇게 말했다고 그게 정답이라는 듯이 말하네"라고 지적했다. 그 뒤 비관락/낙관락/atomic UPDATE/insert-first 4안을 객관 비교해 같은 결론에 도달했다. 같은 결론이라도 즉시 동조와 객관 분석은 산출물이 다르다 — 두 번째만 4안의 트레이드오프가 결정 문서로 남는다(→ [[cart-동시성-낙관락-processor-분리-retry]]). 사용자 의견도 검증 대상으로 두고 객관 분석을 먼저 한다. 이 "코드만 보고 확신하지 말고 코드베이스 패턴·도메인으로 검증" 정신은 [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]]와 같은 뿌리다.

## 관련 링크

- [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]] — 심각도 보정 결과를 후속 이슈로 떼어내는 규율.
- [[설계단계-검증-정본·선례-확인실패와-전제의-적대적검증]] — 리뷰·사람 질문이 잡아낸 결함을 설계 시점으로 앞당기는 짝 규율.
- [[하네스-프로세스-무게와-문서-동작-정합]] — 리뷰·게이트 등 프로세스 무게 조정.

## 열린 질문

- 리뷰어별 층위 구분·심각도 보정을 리뷰 종합 단계의 체크리스트로 절차화할지는 아직 관례로만 남아 있다.

## 근거

- [[raw/sessions/backend/2026-06-07-pr-218-multi-ai-review-lessons-ai]] — 3개 AI 리뷰 층위 통합, 심각도 도메인 보정.
- [[raw/sessions/backend/2026-06-08-pr-226-review-and-scope-lessons-ai]] — 독립 리뷰 수렴, 죽은 코드 제거의 함정.
- [[raw/sessions/backend/2026-06-18-pr-258-harness-review-lessons-ai]] — 봇 distinct 제거 제안 blind accept 금지.
- [[raw/sessions/backend/2026-05-29-order-domain-overview]] — 하네스 자동리뷰 회고(diff 전달·범위).
- [[raw/sessions/backend/2026-05-29-product-domain-overview]] — 자동 리뷰의 한계(diff 잘림 → blocked).
- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]] — sycophancy 억제, 사용자 도메인 인사이트 > 추상 리뷰, 자체 vs 외부 검토.
