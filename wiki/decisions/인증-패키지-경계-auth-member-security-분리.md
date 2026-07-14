---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, member, security, package-boundary, ddd, hexagonal, thread-local, spring-security]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# 인증 패키지 경계 — auth/member/security 3분할과 의존 방향

## 컨텍스트·문제(인증 책임 3분할)

인증을 member(도메인 owner)·auth(인증 유스케이스)·security(HTTP 경계 서블릿 어댑터)로 나누고 단방향 의존(`security → auth.application → member.application`)으로 묶었다. 이 노트는 그 분리의 *왜*를 담는다(구조 서술은 [[auth-member-security-도메인-구조-개요]]).

## auth↔member 분리(application 위임, 무결성 캡슐화)

`AuthSignUpService`가 `MemberRegistrationService.register()`를, `AuthLoginService`가 `MemberQueryService.findByEmail()`을 호출한다 — auth가 member의 *application 레이어*에 의존한다.

- **(A) auth → `MemberRepository` 직접 의존**: member의 도메인 규칙(이메일 중복 검증, `Member.createUser` 입력 검증)이 auth에서 재구현되거나 우회됨.
- **(B) auth → `member.application` service 의존 ✓**: member 도메인 무결성이 register 안에 캡슐화된 채 유지.

경계 분담이 흥미롭다: **raw password는 auth에만 노출**되고 member는 *해시만* 받아 저장한다(비밀번호 hashing은 auth 책임). member는 인증 외에도 주문·배송 등에 엮이는 핵심 도메인이라 인증과 분리해두는 게 자연스럽다. "infrastructure repository는 절대 가로지르지 않고, 도메인 간 협력은 application service끼리"라는 규칙은 [[ddd-이관-컨벤션-adapter-command-query-네이밍]]의 헥사고날 도출과 강하게 연결된다.

## security↔auth 분리(어댑터 vs 유스케이스)

security를 별도 패키지로 둔 이유: 인증 필요 경로 화이트리스트, JWT 추출, ThreadLocal 관리, `@RequireRole` 인터셉터는 *서블릿 인프라*의 횡단 관심사다. auth 안에 Filter가 같이 있으면 auth가 *도메인이자 인프라*가 되어 책임이 비대해진다. security는 *모든 도메인의 인증/인가*를 책임지는 **수평** 패키지, auth는 인증 유스케이스라는 **세로** 패키지다.

| 역할 | 위치 |
|---|---|
| 토큰 발급/검증 알고리즘 | `auth.infrastructure.jwt` |
| 토큰 검증 유스케이스(memberId 파싱) | `auth.application.TokenAuthenticationService` |
| HTTP에서 토큰 추출 | `security.filter.JwtAuthenticationFilter` |
| 인증 결과 전파(ThreadLocal) | `security.context.AuthenticationContext` |
| role 기반 인가 | `security.interceptor.AuthorizationInterceptor` |
| Controller에 memberId 주입 | `security.resolver.AuthenticatedMemberIdArgumentResolver` |

## ThreadLocal 범위 한정(비동기 호환)

`AuthenticationContext`(ThreadLocal)의 memberId는 *Controller에서 추출*된 뒤 이후 모든 로직에는 *DTO/파라미터로 명시 전달*한다. 즉 ThreadLocal은 HTTP 진입 → Controller 입구까지의 짧은 구간만 책임진다. 향후 `@Async`/virtual thread가 도입돼도 비동기 작업은 DTO/파라미터에서 memberId를 꺼내 넘기므로 ThreadLocal이 깨질 위험이 없다 — 비동기 도입 시 별도 작업이 거의 필요 없는 구조.

> 대비: traceId는 *자동 전파*가 가치라 이벤트 객체 동봉·Outbox 컬럼 저장 같은 명시적 동봉 메커니즘으로 경계를 넘겨야 했다(참고 [[mdc-정리-스코프-오너십-2규칙]]). memberId는 *데이터*라 애초에 명시 전달이 자연스럽다. 같은 "경계 넘기"라도 성격이 다르다.

## Spring Security 미사용(학습 목적)

Spring Security를 의도적으로 안 썼다. 학습 비용·과잉 추상화 회피도 이유지만 **결정적 동기는 학습**이다. 직접 Filter + Interceptor를 만들어 *불편함을 느껴야* Spring Security의 장점을 깊이 알고, 그제서야 의존성을 쓰는 의미가 깊어진다는 접근. "추상화 도구를 도입하기 전에 그것이 푸는 문제를 먼저 직접 경험"하는 학습 우선은 [[paymentattempt-호출정책-문서-javadoc-archunit-보류]](ArchUnit 보류)와 같은 결이다.

## 솔직한 메모(강한 결정적 이유는 약함)

auth↔member 분리는 명시적·결정적 동기라기보다 *약한 직관*에 가깝다. 가장 가까운 이유는 "member가 핵심 도메인이라 인증과 분리해두면 자연스럽다"이고, security↔auth와 함께 *DDD 학습 과정에서 자연 도출된 헥사고날 구조*에 가깝다. 강한 의식적 설계로 포장하지 않는다.

## 미해결·후속(WHITELIST 분리, OAuth, 다중 세션)

- **WHITELIST 하드코딩** — `Set.of("/auth/login","/auth/signup","/auth/reissue","/products","/payments/naverpay/return")` + `/products/` prefix가 Filter 안에 박혀 있고 "// 나중에 설정 파일로 분리하기" 코멘트만 남았다. 변경 빈도가 낮아 미뤄둔 상태.
- **OAuth 도입** — `MemberRegistrationCommand`가 이미 해시된 passwordHash를 받는 구조라, 비밀번호 없는 가입이 추가되면 command 재설계 필요. 도입 계획은 있으나 방향(password optional / 별도 credential 엔티티 / 기타)은 미정. member 도메인 모델 재검토 대상.
- **다중 세션** — [[jwt-redis-하이브리드-rtr-ttl-근거]]의 한 회원 한 활성 refresh 참고.

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]] — "결정한 것 5·6(auth↔member, security↔auth 분리)"
