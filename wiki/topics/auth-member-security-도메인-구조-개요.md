---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, member, security, jwt, redis, authentication, package-boundary]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# auth·member·security 도메인 구조 개요 — 3패키지 분리와 인증 데이터 흐름

## 한 줄 정의

인증 도메인을 **member**(회원 도메인 owner) · **auth**(인증 유스케이스) · **security**(HTTP 경계 서블릿 어댑터) 세 패키지로 나누고, 그 위에 JWT(access) + Redis(refresh) 하이브리드 인증을 올린 구조. 이 노트는 그 뼈대를 이루는 결정들의 진입점(허브)이다.

## 패키지 3개의 책임 분리

- **member** — 회원이라는 *도메인 자체*의 owner. `Member` 엔티티 / `MemberRepository` / 등록·조회 유스케이스.
  - `MemberRegistrationService` — 이메일 중복 체크 + 저장(자체 트랜잭션)
  - `MemberQueryService` — `findById` / `findByEmail` / `getById`(조회 전용)
- **auth** — *인증 유스케이스*의 owner. 비밀번호 검증, JWT 발급/검증, refresh token 저장 흐름.
  - `AuthSignUpService`(회원가입, member application에 위임 + 토큰 발급) / `AuthLoginService`(검증 + 발급) / `AuthTokenIssueService`(JWT 발급 + Redis 저장, 내부 공용) / `AuthTokenReissueService`(refresh 검증 + 신규 발급) / `TokenAuthenticationService`(access 검증 → memberId·role 추출, Filter가 호출)
- **security** — *HTTP 경계의 인증/인가 어댑터*. 유스케이스가 아니라 Filter / Interceptor / ArgumentResolver.
  - `JwtAuthenticationFilter`(Bearer 파싱 → `TokenAuthenticationService` → `AuthenticationContext` 저장) / `AuthorizationInterceptor`(`@RequireRole` 검사) / `AuthenticatedMemberIdArgumentResolver`(`@AuthenticatedMemberId Long` 주입) / `AuthenticationContext`(`ThreadLocal<memberId, role>`)

## 단방향 의존 구조

`security → auth.application → member.application`. security는 `auth.application.TokenAuthenticationService`만 호출하고, auth는 `member.application`의 query/registration만 호출한다. infrastructure의 repository는 도메인을 가로질러 호출하지 않는다("도메인 간 협력은 application service끼리"). auth를 세로 패키지(인증 유스케이스), security를 수평 패키지(모든 도메인 인증 책임)로 본다. 이 분리의 *왜*는 [[인증-패키지-경계-auth-member-security-분리]]에 담았다.

## 데이터 흐름(회원가입·로그인·재발급·인증요청)

```
# 회원가입
AuthSignUpService.signUp() [NOT_SUPPORTED]
  → MemberRegistrationService.register() [REQUIRED → 자체 commit]
  → AuthTokenIssueService.issue() → TokenIssuer.create*Token()
  → RefreshTokenStore.save() → Redis (DB commit 이후)

# 로그인
AuthLoginService.login() → MemberQueryService.findByEmail()
  → PasswordHasher.matches() → AuthTokenIssueService.issue() → Redis save(덮어쓰기)

# 토큰 재발급
AuthTokenReissueService.reissue() → TokenValidator.validateRefreshToken()
  → RefreshTokenStore.get() → 저장값 비교 → issue()(RTR: 덮어쓰기)

# 인증된 요청
JwtAuthenticationFilter(Order HIGHEST_PRECEDENCE+30)
  → TokenAuthenticationService.authenticateAccessToken()
  → AuthenticationContext.set(memberId, role) → MDC에 memberId 기록
  → filterChain.doFilter() → finally: AuthenticationContext.clear()
```

회원가입 흐름의 `NOT_SUPPORTED`가 왜 필요한지는 [[회원가입-not-supported-트랜잭션-분리]] 참고.

## tbl_member 스키마·외부 연동(Redis/JWT/BCrypt)

- **tbl_member**: `id(PK)` / `email(UNIQUE uk_member_email, VARCHAR 50)` / `password(VARCHAR 60)` / `username(VARCHAR 12)` / `role(ENUM→VARCHAR, @JdbcTypeCode(SqlTypes.VARCHAR))`. role은 `ROLE_ADMIN` / `ROLE_USER` 두 값, 신규 가입은 `Member.createUser` 안에서 `ROLE_USER` 고정. admin 승격 API 없음.
- **Redis**: `RefreshTokenStore`(port) → `RedisRefreshTokenStore`(infra). 키 `refresh:{memberId}`, TTL = `jwt.refresh.expiration`. 동일 memberId의 새 refresh는 기존을 덮어쓴다(한 회원 = 한 활성 refresh).
- **JWT**: `jjwt`(io.jsonwebtoken). access/refresh가 각각 다른 secret key·만료시간. `type` claim(`ACCESS_TOKEN`/`REFRESH_TOKEN`)으로 종류 검증.
- **BCrypt**: `BcryptPasswordHasher`(`PasswordHasher` port 구현, jbcrypt). hash 길이 60 bytes → `password VARCHAR(60)`.

## 세션 이후 네이밍/트랜잭션 변화 메모

> [!warning] EVOLUTION — 이 노트는 세션 시점(2026-05-29) 스냅샷
> 본문의 `*Service` 이름들(`AuthSignUpService`, `MemberRegistrationService` 등)은 세션 당시 이름이다. 세션 이후(2026-06 중순) application 계층을 역할별 접미사 컨벤션으로 정리하며 대체로 `*UseCase`로 개명됐다. 또한 당시 쓰던 class-level `@Transactional(readOnly=true)` + method-level override 패턴은 이후 전 도메인에서 폐지되고 method-level 명시 부여로 전환됐다(자세한 맥락은 [[회원가입-not-supported-트랜잭션-분리]]). 심볼 이름을 코드와 대조할 때는 이 drift를 감안할 것.

## 관련 결정 링크

- [[jwt-redis-하이브리드-rtr-ttl-근거]] — TTL·RTR·한 회원 한 활성 refresh
- [[redis-장애-strict-정책-soft-fail-기각]] — Redis 단일 장애점 대응 정책
- [[회원가입-not-supported-트랜잭션-분리]] — DB commit 시점과 Redis 저장 경계
- [[인증-패키지-경계-auth-member-security-분리]] — 3분할과 의존 방향의 *왜*
- [[refreshtokenstore-delete-제거-로그아웃-미구현]] — 미사용 추상화 제거·범위 제한
- 다른 도메인 개요와 같은 결: [[order-도메인-구조-개요]] · [[payment-도메인-구조-개요]] · [[product-도메인-구조-개요]]

## 열린 질문

- Redis HA(Sentinel/Cluster) — strict 정책의 전제인데 아직 단일 노드.
- 로그아웃 기능 / WHITELIST 설정 분리 / 다중 디바이스·세션 / OAuth·소셜 로그인 / 비동기 경계의 memberId 전파 / admin 승격 API 부재.

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]
