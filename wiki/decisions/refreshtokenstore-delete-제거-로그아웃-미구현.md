---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, refresh-token, dead-code, logout, abstraction, git-history]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# RefreshTokenStore.delete() 제거 — 로그아웃 미구현의 흔적

## 컨텍스트·문제(호출부 없는 메서드)

`RefreshTokenStore` 인터페이스에서 `delete(Long memberId)` 메서드 자체를 삭제했다. 로그아웃 기능이 미구현이라 호출부가 없었다. 이 저장소의 위치·역할은 [[auth-member-security-도메인-구조-개요]] 참고.

## 결정과 선택 이유(미사용 추상화 제거)

- "불필요한 추상화와 과한 설계를 피한다"는 원칙.
- 호출부 없는 인터페이스 메서드는 잠재적 혼란을 만든다 — 다른 사람이 보면 "어디서 호출되겠지" 하고 *잘못된 추론*을 한다.
- Git history가 *제거 이유*와 *과거 존재*를 기록하므로 정보 손실은 0이다.
- 로그아웃 구현 시 *그 PR 안에서* `delete()`와 Redis 실패 정책을 함께 설계하는 게 더 안전하다.

> "미사용 추상화 제거 + git history가 과거를 기억" 패턴은 [[order-concurrency-service-학습코드-격리]]의 정리 방침과 동일 원칙이다 — knowledge로 일반화할 후보(안 쓰는 helper·설정 값·"혹시 모르니 남긴" 코드 모두 같은 처리).

## 로그아웃 미구현(의도된 범위 제한)

토큰 만료(30분 access + 7일 refresh + RTR)로 실용적으로 충분히 커버된다고 판단했다. 어려운 기능이 아니지만 *지금 크게 의미 없는 기능을 미리 도입하기 싫었다* — 필요한 시점에 도입하면 된다. 의식적 범위 제한이다.

## 로그아웃 도입 시점의 가치(의식적 종료)

[[jwt-redis-하이브리드-rtr-ttl-근거]]의 RTR이 *다른 기기 로그인 시 이전 세션을 자동 무효화*하는 것은 이미 해준다. 따라서 logout API의 *추가 가치*는 "사용자의 의식적 종료 의사 표현"에 한정된다 — 자동/자연 발생이 아닌 *의도적 트리거*만 logout의 영역이다(예: 다른 기기에서 명시적으로 끊고 싶을 때).

## 향후 과제(delete 재추가 + Redis 정책 재결정)

로그아웃 구현 시 `delete()` 재추가 + Redis 실패 정책 *재*결정이 필요하다. 회원가입/로그인의 strict([[redis-장애-strict-정책-soft-fail-기각]])와 같은 정책이 로그아웃에도 적용될지는 별도 판단 — 로그아웃은 보안 목적이라 "Redis 안 되어도 일단 access 비활성화"가 오히려 의미 있을 수 있다.

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]] — "결정한 것 4. RefreshTokenStore.delete() 제거"
