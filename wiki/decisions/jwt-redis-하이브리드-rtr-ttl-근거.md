---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, jwt, redis, rtr, refresh-token, session]
created: 2026-05-29
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# JWT + Redis 하이브리드 + RTR — TTL 근거와 한 회원 한 활성 refresh

## 컨텍스트·문제(인증 방식 선택)

access token은 JWT(stateless, **30분** = 1800000ms), refresh token은 Redis 저장 + RTR(1회용, **7일** = 604800000ms)의 하이브리드로 정했다. "짧은 access + 긴 refresh"는 정석 조합이지만, 각 값의 *왜 그 값인가*와 stateless를 왜 포기했는가를 남긴다. 전체 도메인 맥락은 [[auth-member-security-도메인-구조-개요]] 참고.

## TTL 각 값의 근거(access 30분 / refresh 7일)

- **access 30분** — 탈취돼도 *서버가 무효화할 수단이 없다*(무효화하려면 매 요청 Redis 조회로 너무 stateful해짐). 그래서 짧게 가져간다.
- **refresh 7일** — access만 쓰면 30분마다 로그인해야 해 UX 저하. refresh를 두되 *기간이 짧으면 무의미*(1시간이면 1시간 뒤 재접속에 또 로그인). 3~4일 뒤 다시 와도 무자료 재발급되도록 7일.
- 각 값은 이 UX·보안 트레이드오프에서 직접 도출됐다.

## 왜 RTR을 적용했나

refresh 기간이 길수록(7일) 탈취 시 노출 시간이 길다 → **1회용으로 만들어 탈취 영향을 축소**. `reissue` 호출 시 기존 refresh 무효화 + 새 refresh 발급을, 같은 키(`refresh:{memberId}`)에 덮어쓰는 구조로 자연 구현했다. 탈취된 refresh가 먼저 쓰인 뒤 정상 사용자가 같은 refresh로 시도하면 거부된다(Redis의 token 값이 이미 달라짐).

## 왜 완전 stateless로 안 갔나

refresh도 JWT로만 검증하는 옵션은, 한 번 발급된 refresh를 만료 전까지 *서버가 강제 무효화할 수단이 없다*. 도용 토큰을 끊을 방법이 없어 **RTR 자체가 성립 안 함**. Redis 저장이면 "RTR + 강제 무효화" 둘 다 된다. 이 선택이 아래 다중 디바이스 정책과 [[redis-장애-strict-정책-soft-fail-기각]]의 전제를 만든다.

## 왜 access까지 Redis에 안 두나

매 요청마다 Redis 왕복 → 인증 부하 폭증. access는 JWT 서명 검증만으로 충분. 이 성질 덕에 Redis 장애 시에도 유효 access 보유자는 인증을 통과한다.

## 다중 디바이스 정책(한 회원 한 활성 refresh)

refresh가 단일 키(`refresh:{memberId}`)에 저장되므로, 다른 디바이스에서 로그인하면 *마지막 로그인만 살아남는다*. 이는 별도 결정이라기보다 **RTR의 자연스러운 결과**다. 다중 디바이스 별도 세션을 원하면 키 구조(`refresh:{memberId}:{deviceId}`)를 바꾸고 RTR 정책도 디바이스별로 재설계해야 한다. 현 시점 주요 요구사항이 아니라고 판단해 의식적으로 미도입. 이 "다른 기기 로그인 시 이전 세션 자동 무효화" 성질은 [[refreshtokenstore-delete-제거-로그아웃-미구현]]에서 로그아웃 미구현을 실용적으로 커버하는 논거가 된다.

## 트레이드오프(Redis 단일 장애점 전제)

refresh 저장소 운영이 필요하고, Redis 단일 장애점이 인증 가용성에 직접 영향을 준다. 이 노출을 어떻게 다룰지가 [[redis-장애-strict-정책-soft-fail-기각]]의 출발점이다. Redis HA(Sentinel/Cluster)는 아직 미도입.

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]] — "결정한 것 1. JWT + Redis 하이브리드 + RTR"
