---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [scope-management, stacked-pr, process, code-review, technical-debt, workflow]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-08-pr-226-review-and-scope-lessons-ai]]"
  - "[[raw/sessions/backend/2026-06-08-pr-224-harness-gate-and-review-lessons-ai]]"
  - "[[raw/sessions/backend/2026-06-07-pr-218-multi-ai-review-lessons-ai]]"
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# 스코프 규율 — 한 PR 한 목적, 발견한 근본 개선·인접 부채는 즉석 패치 말고 처리처로 분리

**한 줄 정의.** 작업 중 발견한 근본 개선이나 리뷰가 드러낸 인접 부채를 현재 PR에 끼워넣지 않고, "그걸 안 고치면 현재 작업이 틀리거나 못 끝나나?"라는 프레임으로 갈라 별도 이슈 또는 stacked PR로 떼어내는 규율. 한 PR = 한 목적이라야 리뷰도 추적도 무너지지 않는다.

## 한 PR = 한 목적 — 판단 프레임("안 고치면 현재 작업이 틀리나?")

PR #226(승인 보상·예외 처리 정리) 도중 더 깔끔한 근본 개선을 발견했다: JDBC `SQLException`을 Spring DAO 예외로 번역하는 전역 설정 빈(`JpaConfig`의 `SQLErrorCodeSQLExceptionTranslator`)을 제거하면 Hibernate `getConstraintName()` 경로가 되살아나 이중결제 식별을 메시지 파싱 대신 구조적 API로 단순화할 수 있다. 하지만 그 빈은 운영 로그에서 unique 위반을 `DuplicateKeyException`으로 구분하려는 전역 설정이라, 제거는 전역 예외 분류·로깅을 바꾸는 별개 사안이다.

핵심 판단 프레임은 **"그걸 안 고치면 현재 작업이 틀리거나 못 끝나나?"**

- **아니오** → 현 제약 안에서 마무리하고 발견은 근거와 함께 별도 이슈로 떼어낸다.
- **예** → 현재 작업을 멈추고 그 선행 개선을 먼저 별도 PR로 머지한 뒤 그 위에서 재개한다.

이번 건은 전역 빈을 안 빼도 이중결제 매핑이 `SQLException` 메시지 매칭으로 올바르게 완결되고 나중에 되돌림 없이 개선을 얹을 수 있으므로 → **#227로 분리**했다(이후 다음 PR에서 실제로 그 빈을 제거하고 식별을 `getConstraintName()` 기반으로 전환 — [[sql-translator-빈-제거-제약명-이중결제-식별]]). 이 scope 규율은 스키마를 건드리지 않는 리팩터 시리즈의 메타 원칙과도 같은 뿌리다(→ [[schema-무변경-decouple-series-메타원칙과-scope-규율]]).

## 별도 이슈 vs stacked PR 갈림

같은 프레임에서 판정이 "예"로 갈리면 선행 개선을 먼저 머지하고 그 위에 후속 PR을 쌓는 **stacked PR**로 간다. "아니오"면 별도 이슈로만 떼어내고 현재 작업은 현 제약 안에서 완결한다. 갈림의 기준은 "지금 안 고치면 현재 변경이 *틀린 채로* 완결되거나 *못 끝나는가*" 하나다. #227은 "안 빼도 올바르게 완결"이라 별도 이슈였다.

## 리뷰가 드러낸 인접 부채는 즉석 패치 금지 — 처리처로 매핑 분리

다각도 코드 리뷰(정확성 / 리팩터로 사라진 동작 / 테스트 커버리지 등 여러 관점 병렬)는 "정책 한 줄"이 아니라 인접 부채를 드러낸다. PR #224(후처리 정책 재설계)의 리뷰가 정책 본체를 넘어 인접한 실시간 보상·예외 전략 부채까지 끌어냈다.

- **결정:** 드러난 발견을 본 PR에 즉석 "한 줄 패치"로 메우지 않고, 각 발견을 "이 PR / 후속 이슈 / Epic"이라는 처리처로 매핑해 분리한다.
- 결과적으로 정책 본체만 #224로 깨끗이 머지하고, 인접 부채는 처리처로 떼어냈다:
  - **#225** — 정상 승인 후 기록 실패 시 환불 대신 완료 재시도 + application 계층의 DB 무결성 예외 직접 catch 제거·이중결제 보상 순서 일관화(→ [[결제승인완료-보상-완료우선-이중결제-adapter매핑]], [[requires-new-격리-제거-보상판단-트랜잭션정책]]).
  - **#219** — `AlreadyComplete` 후 UNKNOWN 보존(→ [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]).
  - **#222** — 주문 만료 취소와 결제 대사 타이밍 정합(→ [[미확정차단-대사스캔-정합성-starvation-escalation]]).
  - **Epic #208** — 운영 배치·스캔 wiring.

## 즉석 패치가 틀린 모델을 굳힐 뻔한 사례

PR #224에서 처음엔 발견 하나를 본 PR에 즉석 패치로 메우려 했는데, 사용자가 그 처리 방향 자체가 틀렸음을 짚었다 — 정상 승인이 끝난 결제를 transient 기록 실패만으로 환불하는 건 "완료 재시도"여야 할 것을 "실패 처리"로 굳히는 잘못된 모델이었다. 섣불리 패치했으면 잘못된 모델을 코드에 박제했을 것이다. 즉 인접 부채를 즉석으로 메우는 위험은 단순 scope 오염을 넘어 **틀린 도메인 모델을 코드로 고착**시키는 데 있다. 발견을 정확한 처리처로 매핑하는 디시플린이 핵심이었다(올바른 재정합은 [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]).

## 리뷰 누적은 commit 분리가 아니라 PR scope에서 온다

cart 도입(PR #166)은 커밋 수가 일반 PR 기준 outlier였는데, 근본 원인은 커밋 분리 방식이 아니라 **PR scope가 컸고 + 리뷰가 여러 차례 누적**된 것이었다. 한 PR에 (1) cart 도메인 코드 (2) order 결합 코드 (3) 응용 계층 컨벤션(method-level 트랜잭션·명시적 영속화) (4) 결정 문서 색인 (5) concurrency 테스트 gradle 태스크 — 다섯 책임이 섞였다. squash merge라 main history엔 영향이 없어도 PR 페이지 검토 비용은 그대로 누적된다. → 도메인 PR과 cross-cutting 컨벤션 PR을 분리하고 Draft PR로 출발해 1차 리뷰 후 ready로 전환하는 방향으로 정리했다.

## 관련 링크

- [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]] — 심각도 도메인 보정 결과가 후속 분리로 이어지는 짝 규율.
- [[하네스-프로세스-무게와-문서-동작-정합]] — 게이트 경량화도 같은 PR #224에서 나온 프로세스 판단.
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]] — 스키마 무변경 시리즈의 scope 규율.

## 열린 질문

- 도메인 PR / cross-cutting 컨벤션 PR을 나누는 실험을 아직 실제로 적용해보지 않았다.
- 분리한 후속들(#225·#222·Epic #208)의 처리처·조건만 확정한 상태다.

## 근거

- [[raw/sessions/backend/2026-06-08-pr-226-review-and-scope-lessons-ai]] — #227 분리, "안 고치면 틀리나?" 프레임.
- [[raw/sessions/backend/2026-06-08-pr-224-harness-gate-and-review-lessons-ai]] — 인접 부채 처리처 매핑(#225·#222·Epic #208), 틀린 모델 굳힐 뻔한 사례.
- [[raw/sessions/backend/2026-06-07-pr-218-multi-ai-review-lessons-ai]] — 심각도 보정 후 #219 분리.
- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]] — 다섯 책임 혼재 → 리뷰 누적은 PR scope에서.
