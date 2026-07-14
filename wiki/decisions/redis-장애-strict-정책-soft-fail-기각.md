---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, redis, strict-policy, availability, error-handling, fail-fast]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# Redis 장애 시 strict 정책 — soft fail의 함정

## 컨텍스트·문제(Redis 장애 시 동작)

[[jwt-redis-하이브리드-rtr-ttl-근거]]가 refresh token 저장소로 Redis를 도입하면서 Redis 단일 장애점이 인증 가용성에 직접 걸리게 됐다. Redis 저장/조회가 실패할 때 어떻게 동작할지가 문제다. 결론은 **strict** — `AuthException(INTERNAL_ERROR, 500)`으로 즉시 실패하고 신규 로그인/회원가입/재발급을 일시 불가로 둔다.

## 왜 soft fail을 안 했나(예측 불가 깨짐)

soft fail(Redis 실패해도 토큰은 발급)은 다음 순서로 터진다.

1. 클라이언트는 refresh를 받았다고 생각
2. access 만료 → `/auth/reissue` 호출
3. Redis에 없으므로 `REFRESH_TOKEN_NOT_FOUND`
4. 사용자 입장: "방금 로그인됐는데 갑자기 인증이 풀림"

즉 "동작하는 것처럼 보이지만 실제로는 망가진" 상태가 *예측 불가능한 시점에 터지는 버그*로 인지된다.

## 근거의 본질(refresh 진실 source = Redis)

refresh token의 *진실 source*가 Redis다. Redis 없는 refresh는 발급돼도 절대 못 쓰는 종이쪼가리다. soft fail은 "쓸 수 없는 자원"을 발급하는 것에 가깝다. 반면 유효 access 보유자는 Redis 없이도 JWT 서명 검증만으로 통과하므로, 장애 영향 범위는 신규 로그인/재발급에 한정된다.

## 구현 부담 측면(soft fail이 분기 비용도 손해)

soft fail은 Redis 죽었을 때 로그인 → refresh 저장 실패 → *발급 실패 처리 분기*를 코드에 추가해야 한다(복잡도 증가). accessToken만 발급돼도 30분 후 또 로그인이라 UX는 결국 떨어진다. "억지로 accessToken을 발급해 서비스를 쓰게 하는 것"이 더 별로다. 이 "부분 실패 분기 부담을 피한다"는 판단은 [[회원가입-not-supported-트랜잭션-분리]]에서 AFTER_COMMIT을 제외한 논리와 같은 사고 흐름이다.

## strict의 적극적 가치(운영 모드 명확성)

strict는 단순한 fail-fast가 아니라 *운영 모드의 명확성*을 산다. "지금 인증 못 함"을 홈페이지에서 깔끔히 안내하고 이후 작업을 막아 빠르게 복구하는 게, 부분 동작/부분 실패가 섞인 모호한 상태보다 운영·복구 측면에서 단순하다.

## 영향 범위 한정·에러 메시지 추상화

`AuthErrorCode.INTERNAL_ERROR` 메시지는 "인증 처리 중 오류가 발생했습니다"로 추상화한다. (1) **보안** — 인프라 구조(Redis 사용 사실) 외부 노출 회피, (2) **UX** — 사용자가 내부 동작 실패 원인을 알 필요 없음. 두 동기가 함께 작동한다.

## 미해결·후속(Redis HA 전제)

이 정책은 Redis HA(Sentinel/Cluster)로 단일 장애점을 인프라 레벨에서 해결하는 것을 전제로 한 임시 정책 성격이다. 현 시점은 가용성까지 학습 곡선이 부담스러워 백엔드 정책 정립에 집중했고, HA 도입 시 strict가 여전히 맞는지 재평가 예정.

> 이 노트의 "동작하는 것처럼 보이지만 망가진 > 즉시 실패" 원칙은 인증 밖에도 적용된다. 결제에서 멱등 재요청 금액이 어긋날 때 조용히 진행하지 않고 명시적 예외로 거부하는 [[payment-amount-mismatch-이중검증-409-vs-400-분리]]도 같은 정신이다 — knowledge로 일반화할 후보.

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]] — "결정한 것 2. Redis 장애 시 strict 정책"
