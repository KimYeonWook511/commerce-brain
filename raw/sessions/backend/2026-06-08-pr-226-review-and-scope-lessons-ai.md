---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 226 }
---

# PR #226 리뷰·스코프 회고 — "죽은 코드" 제거의 함정과 근본 개선의 분리(#227)

이슈 #225의 결제 승인 보상·예외 처리 정리(PR #226)를 진행하며 얻은 **개발 프로세스·리뷰·범위 관리** 쪽 교훈을 모은 메모다. 구현은 자체 개발 하네스(harness-v2 workflow)로 돌렸고 — 두 개의 구현 step을 developer 모델(sonnet)이 작성하고 reviewer 모델(opus)이 검토, commit 모델(haiku)이 커밋하는 구성 — PR 리뷰는 로컬 multi-agent 도구와 Gemini Code Assist 두 채널을 독립으로 받았다. 이 PR에서 실제로 내린 도메인 결정(승인 후 transient 실패의 완료 우선 처리, 이중결제 탐지의 adapter 도메인 예외 매핑, 예외 변환 디테일)은 같은 PR을 다룬 별도 메모에 있고, 여기서는 "그 과정에서 무엇이 나를 헛짚게 했고 무엇을 어디까지 이번 PR에 담을지"만 남긴다.

## 결정한 것

### 발견한 근본 개선을 이번 작업에 끼우지 않고 별도 이슈(#227)로 분리

작업 도중 더 깔끔한 근본 개선을 발견했다: JDBC `SQLException`을 Spring DAO 예외로 번역해주는 전역 설정 빈(`JpaConfig`에 등록된 `SQLErrorCodeSQLExceptionTranslator`)을 제거하면, Hibernate의 제약명 접근자(`getConstraintName()`) 경로가 되살아나 이중결제 식별을 메시지 파싱 대신 구조적 API로 단순화할 수 있다. 하지만 그 빈은 운영 로그에서 unique 위반을 `DuplicateKeyException` 타입으로 구분하려는 전역 설정이라, 제거는 **전역 예외 분류·로깅을 바꾸는 별개 사안**이다. 이중결제 버그를 고치는 이번 이슈에 곁다리로 끼우면 묻혀버린다.

- **핵심 판단 프레임:** "그걸 안 고치면 *현재 작업이 틀리거나 못 끝나나?*"
  - **아니오** → 현 제약 안에서 마무리하고, 발견은 근거와 함께 **별도 이슈로 떼어낸다.**
  - **예** → 현재 작업을 멈추고 그 선행 개선을 먼저 별도 PR로 머지한 뒤 그 위에서 재개한다(선행 PR 위에 후속 PR을 쌓는 stacked PR 방식).
- **이번 건의 판정:** 전역 빈을 안 빼도 이중결제 매핑은 `SQLException` 메시지 매칭으로 올바르게 완결되고, 나중에 되돌림 없이 그 위에 개선을 얹을 수 있다 → 현재 작업이 틀리지 않으므로 **#227로 분리**했다. (이 #227은 이후 다음 PR에서 실제로 처리돼, 그 빈을 제거하고 식별을 `getConstraintName()` 기반으로 전환했다.)
- **원칙:** 한 PR = 한 목적.

## 막힌 점·해결

### 테스트가 가정을 뒤집었다 — "안 탈 폴백"이 실은 주 경로였다

이중결제 제약명을 식별하는 초기 구현은 두 단계였다: cause 체인에서 Hibernate `ConstraintViolationException.getConstraintName()`을 1차로 보고, `SQLException` 메시지 매칭을 2차 폴백으로 뒀다. 리뷰 중 "폴백은 도달하지 않는 보험이니 지우고 1차만 남기자"고 **코드만 보고 판단해 폴백을 제거**했더니, MySQL 통합 테스트가 깨졌다.

- **원인:** 실패 예외가 결정적 증거였다 — `DuplicateKeyException: ... for key 'tbl_payment.uk_payment_approved_order_key'`에 스택트레이스 최상위가 예외 번역기였다. `JpaConfig`가 그 translator 빈을 등록하기 때문에 unique 위반이 `DuplicateKeyException`(cause=JDBC `SQLException`)으로 변환되고, **cause 체인에 Hibernate `ConstraintViolationException`이 아예 없다.** 즉 `getConstraintName()` 분기는 처음부터 한 번도 타지 않는 dead 경로였고, 실제 매핑은 폴백인 `SQLException` 메시지 매칭이 담당하고 있었다.
- **교훈으로 남은 것:** 죽었다고 믿은 1차가 아니라 "보험"이라던 폴백이 진짜 주 경로였다. 코드만 보고 "이건 안 탈 것"이라고 확신한 추론이 프레임워크의 예외 변환 실제 동작 앞에서 정확히 뒤집혔다.

### 이중결제 판정 함수의 short-circuit 버그 — 세 번 만에 수렴

- **두 리뷰가 독립으로 같은 버그를 짚었다:** 제약명 판정 함수(`isApprovedOrderKeyViolation`)가 첫 매치에서 `false`를 **단정**해, 같은 cause 체인 뒤쪽의 폴백에 도달하지 못하는 short-circuit 결함.
- **한 번에 안 수렴했다 — 3회 반복.** 사용자가 "그러면 또 false 되는 거 아니냐"고 **같은 함정을 다시 짚어준 게 전환점**이었다. 내가 만든 수정이 같은 함정에 또 빠지는 걸 사람이 잡아줬다. (1차: 제약명이 null이면 즉시 `false` → 폴백 못 감. 2차: null일 때만 폴백을 열었으나 제약명이 non-null이지만 다른 값이면 여전히 `false` 단정. 최종: 어떤 분기도 `false`를 조기 단정하지 않고 "일치할 때만 true, 끝까지 못 찾으면 false"인 OR 구조로 정리. 수정 변천의 상세와 최종 형태는 같은 PR의 도메인 결정 메모에 있다.)

## 배운 것

- **독립 리뷰의 수렴은 강한 신호.** 로컬 multi-agent 리뷰(finder 에이전트 여러 개를 fan-out 한 뒤 검증 단계를 거치는 자체 리뷰 도구)와 Gemini Code Assist가 서로 모른 채 같은 short-circuit 버그를 짚었다. 서로 독립인 두 리뷰가 같은 결론이면 그 지적은 신뢰도가 높다.
- **죽은 코드처럼 보여도 제거 전에 테스트로 실제 런타임 경로를 확인.** 특히 예외 변환처럼 프레임워크 내부에서 타입·cause 체인이 재조립되는 동작은 코드 추론이 빗나가기 쉽다. 돈이 걸린 경로에서는 더더욱, 진짜 죽었는지 통합 테스트로 드러내고 지운다.
- **발견한 근본 개선을 현재 작업에 끼워넣지 않는다.** "안 고치면 현재 작업이 틀리거나 못 끝나나?"로 가르고, 아니면 별도 이슈, 맞으면 stacked PR. 한 PR에 목적이 하나여야 리뷰도 추적도 무너지지 않는다.
