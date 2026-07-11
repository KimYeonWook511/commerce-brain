---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

# auth·member·security 도메인 개요 — JWT+Redis 하이브리드 인증과 그 뼈대를 이루는 결정들

인증 도메인을 `auth`·`member`·`security` 세 패키지로 어떻게 나눴고, 그 위에 올린 JWT+Redis 하이브리드 인증이 *왜 지금 형태인가*를 한 번에 정리한 세션이다. TTL 값·RTR·Redis 장애 정책·회원가입 트랜잭션 경계·패키지 의존 방향 같은 결정 각각을 "그때 무엇을 저울질했나"까지 남겨, 나중에 재고할 때 근거를 다시 파지 않아도 되게 하는 게 목적이다.

## 도메인 개요

### 패키지 3개의 책임 분리

`auth` / `member` / `security` 가 *다른 책임*을 지는 3개 패키지로 갈려 있다.

- **`member`** — 회원이라는 *도메인 자체*의 owner. `Member` 엔티티 / `MemberRepository` / 등록·조회 유스케이스.
  - `MemberRegistrationService` — 이메일 중복 체크 + 저장 (자체 트랜잭션)
  - `MemberQueryService` — `findById` / `findByEmail` / `getById` (조회 전용)
- **`auth`** — *인증 유스케이스*의 owner. 비밀번호 검증, JWT 발급/검증, refresh token 저장 흐름.
  - `AuthSignUpService` — 회원가입 (member 의 application 계층에 위임 + 토큰 발급)
  - `AuthLoginService` — 이메일/비밀번호 검증 + 토큰 발급
  - `AuthTokenIssueService` — JWT 발급 + Redis 저장 (내부 공용)
  - `AuthTokenReissueService` — refresh token 검증 + 신규 발급
  - `TokenAuthenticationService` — access token 검증 → memberId/role 추출 (Filter 가 호출)
- **`security`** — *HTTP 경계의 인증/인가 어댑터*. 유스케이스가 아니라 Filter / Interceptor / ArgumentResolver.
  - `JwtAuthenticationFilter` — `Authorization: Bearer ...` 파싱 → `TokenAuthenticationService` 호출 → `AuthenticationContext` 에 저장
  - `AuthorizationInterceptor` — `@RequireRole` 애너테이션 검사
  - `AuthenticatedMemberIdArgumentResolver` — `@AuthenticatedMemberId Long` Controller 파라미터 주입
  - `AuthenticationContext` — `ThreadLocal<Long memberId, String role>` 보관소

핵심 흐름은 `auth` 가 *유스케이스*를, `security` 가 *서블릿 어댑터*를, `member` 가 *도메인*을 담는 형태로 일관되게 분리돼 있다. `security` 는 `auth.application` 의 `TokenAuthenticationService` 만 호출하고, `auth` 는 `member.application` 의 query/registration 만 호출한다 — 단방향 의존.

> 이 세션 이후(2026-06 중순) application 계층 클래스명을 역할별 접미사 컨벤션으로 정리하면서 위 `*Service` 들이 대체로 `*UseCase` 로 개명됐다. 여기 이름은 세션 시점(2026-05-29) 기준.

### 외부 연동

- **Redis** — `RefreshTokenStore` (port) → `RedisRefreshTokenStore` (infra). 키는 `refresh:{memberId}`, TTL = `jwt.refresh.expiration`. 동일 memberId 의 새 refresh token 은 *기존을 덮어쓴다* (한 회원 = 한 활성 refresh).
- **JWT** — `jjwt`(io.jsonwebtoken) 라이브러리. access/refresh 가 *각각 다른 secret key 와 다른 만료시간*. `type` claim (`ACCESS_TOKEN` / `REFRESH_TOKEN`) 으로 토큰 종류를 검증한다.
- **BCrypt** — `BcryptPasswordHasher` (`PasswordHasher` port 구현, `org.mindrot.jbcrypt.BCrypt`). hash 길이 = 60 bytes → `tbl_member.password VARCHAR(60)`.

### 데이터 흐름

```
# 회원가입
AuthController → AuthSignUpService.signUp() [NOT_SUPPORTED]
  → MemberRegistrationService.register() [REQUIRED → 자체 commit]
  → AuthTokenIssueService.issue()
      → TokenIssuer.createAccessToken() / createRefreshToken()
      → RefreshTokenStore.save() → Redis (DB commit 이후)

# 로그인
AuthController → AuthLoginService.login()
  → MemberQueryService.findByEmail()
  → PasswordHasher.matches()
  → AuthTokenIssueService.issue() → Redis save (덮어쓰기)

# 토큰 재발급
AuthController → AuthTokenReissueService.reissue()
  → TokenValidator.validateRefreshToken()
  → RefreshTokenStore.get() → 저장된 것과 비교
  → AuthTokenIssueService.issue() (RTR: 기존 덮어쓰기)

# 인증된 요청
JwtAuthenticationFilter (Order: HIGHEST_PRECEDENCE + 30)
  → TokenAuthenticationService.authenticateAccessToken()
  → AuthenticationContext.set(memberId, role)
  → 로깅 컨텍스트(MDC)에 memberId 기록
  → filterChain.doFilter()
  → finally: AuthenticationContext.clear()
```

### tbl_member 스키마

- `id (PK)` / `email (UNIQUE uk_member_email, VARCHAR 50)` / `password (VARCHAR 60)` / `username (VARCHAR 12)` / `role (ENUM → VARCHAR, `@JdbcTypeCode(SqlTypes.VARCHAR)`)`
- role 은 `ROLE_ADMIN` / `ROLE_USER` 두 값. 신규 가입은 `ROLE_USER` 로 고정 (`Member.createUser` 안에서 강제). admin 승격 API 는 없음.

---

## 결정한 것

### 1. JWT + Redis 하이브리드 + RTR

access token = JWT (stateless, **30분** = `1800000ms`), refresh token = Redis 저장 + RTR (1회용, **7일** = `604800000ms`).

**TTL 결정 — 각 값의 근거**:
- **access 30분** — 탈취돼도 *서버가 무효화할 수단이 없다* (무효화하려면 매 요청 Redis 조회로 너무 stateful 해짐). 그래서 짧게 가져간다.
- **refresh 7일** — access 만 쓰면 사용자가 30분마다 로그인해야 해 UX 저하. refresh 를 두되 *기간이 짧으면 무의미* (1시간이면 1시간 뒤 재접속 시 또 로그인). 3~4일 뒤 다시 와도 무자료 재발급되도록 7일.
- *짧은 access + 긴 refresh* 의 정석 조합인데, 각 값의 *왜 그 값인가*는 위 UX·보안 트레이드오프에서 직접 도출됐다.

**왜 RTR (Refresh Token Rotation) 을 적용했나**:
- refresh 기간이 길수록(7일) 탈취 시 노출 시간이 길다 → **1회용으로 만들어 탈취 영향을 축소**.
- `reissue` 호출 시 기존 refresh 무효화 + 새 refresh 발급. 같은 키(`refresh:{memberId}`)에 덮어쓰는 구조로 자연 구현.
- 탈취된 refresh 가 먼저 쓰인 뒤 정상 사용자가 같은 refresh 로 시도하면 거부된다 (Redis 의 token 값이 이미 달라짐).

**왜 완전 stateless 로 안 갔나** — refresh 를 JWT 로만 검증하는 옵션:
- 한 번 발급된 refresh 를 만료 전까지 *서버측에서 강제 무효화할 수단이 없다*. 도용된 토큰을 서버가 끊을 방법이 없다 → **RTR 자체가 성립 안 함**.
- Redis 저장이면 "RTR + 강제 무효화" 둘 다 된다.

**왜 access 까지 Redis 에 두지 않나**:
- 매 요청마다 Redis 왕복 → 인증 부하 폭증. access 는 JWT 서명 검증만으로 충분.

**다중 디바이스 정책 — "한 회원 = 한 활성 refresh"** (RTR 의 자연스러운 결과):
- 한 회원의 refresh 가 단일 키(`refresh:{memberId}`)에 저장되므로, 다른 디바이스에서 로그인하면 *마지막 로그인만 살아남는다*.
- 의식적 선택이다 — RTR 을 적용한 결과로 자연 도출됐다. 다중 디바이스 별도 세션을 원하면 키 구조(`refresh:{memberId}:{deviceId}`)를 바꾸고 *RTR 정책도 디바이스별로 재설계*해야 한다.
- 현 시점에서 다중 디바이스 별도 세션은 *주요 요구사항이 아니라고 판단*해 미도입.

**트레이드오프**: refresh token 저장소 운영이 필요하다. Redis 단일 장애점이 인증 가용성에 영향 → 아래 Redis 장애 strict 정책의 *전제 조건*이 된다.

### 2. Redis 장애 시 strict 정책 — soft fail 의 함정

Redis 저장/조회 실패 시 `AuthException(INTERNAL_ERROR)` (500) 으로 즉시 실패. 신규 로그인/회원가입/재발급 모두 일시적 불가.

**왜 soft fail 을 안 했나** — *예측 불가능한 깨짐*:
1. soft fail 이면 클라이언트는 refresh 를 받았다고 생각
2. access 만료 → `/auth/reissue` 호출
3. Redis 에 없으므로 `REFRESH_TOKEN_NOT_FOUND`
4. 사용자 입장: "방금 로그인됐는데 갑자기 인증이 풀림"

→ "동작하는 것처럼 보이지만 실제로는 망가진" 상태가 *예측 불가능한 시점에 터지는 버그*로 인지된다.

**근거의 본질**: refresh token 의 *진실 source* 가 Redis 다. Redis 없는 refresh 는 *발급돼도 절대 못 쓰는* 종이쪼가리. soft fail 은 "쓸 수 없는 자원"을 발급하는 것에 가깝다.

**구현 부담 측면에서도 soft fail 이 손해**:
- Redis 죽었을 때 로그인 → refresh 저장 안 됨 → **발급 실패 처리 분기**를 코드에 추가해야 함 (복잡도 증가)
- accessToken 만 발급되면 30분 후 또 로그인 → UX 는 결국 떨어진다
- "억지로 accessToken 을 발급하며 서비스를 이용하게 하는 것"이 더 별로

**strict 의 적극적 가치 — 운영 모드의 명확성**:
- 홈페이지에서 "로그인이 안 된다"는 안내 표시 → 이후 모든 작업을 깔끔하게 막고 **빠르게 복구**
- 부분 동작/부분 실패가 섞인 모호한 상태보다, 명확한 "지금 인증 못 함" 상태가 운영·복구 측면에서 단순

**기존 세션은 무사** — 유효한 access 보유 사용자는 Redis 없이도 access 검증(JWT 서명)만으로 인증 통과 → *Redis 장애 영향 범위 = 신규 로그인/재발급에 한정*.

**HA 도입 시점에 정책 재검토 계획** — 현 시점은 가용성까지 학습 곡선이 부담스러워 백엔드 정책 정립에 집중. Redis HA (Sentinel / Cluster) 가 들어오면 strict 가 여전히 맞는지 재평가 예정.

**`AuthErrorCode.INTERNAL_ERROR` 메시지 추상화** — "인증 처리 중 오류가 발생했습니다":
- **보안**: 인프라 구조(Redis 사용 사실) 외부 노출 회피
- **UX**: 사용자가 Redis 같은 내부 동작 실패 원인을 알 필요 없음
- 두 동기가 함께 작동

**관련 부채 → 향후 과제**: Redis 단일 장애점. Sentinel / Cluster 구성으로 인프라 레벨 해결을 전제로 한 정책이다.

### 3. 회원가입 트랜잭션 분리 `NOT_SUPPORTED` — REQUIRED / AFTER_COMMIT / NOT_SUPPORTED 3자 비교

`AuthSignUpService.signUp()` 에 `@Transactional(propagation = NOT_SUPPORTED)`.

**문제 상황** — 단순히 `@Transactional` 만 붙이면:
- `signUp()` 이 외부 트랜잭션을 열고
- `register()` 가 `REQUIRED` 로 합류
- Spring 의 commit 시점 규칙: *트랜잭션을 시작한 메서드가 종료될 때 commit*
- 따라서 `register()` 가 반환된 시점에도 DB 는 *아직 미커밋*
- 그 사이 `issue()` 가 Redis 저장 → **DB commit 전 Redis 저장 불일치**
- register 후 어떤 이유로 rollback 발생 시 → Redis 엔 refresh 살아있는데 DB 엔 member 없음 = *유령 토큰*

**후보 비교**:

| 후보 | 결과 | 채택 안 한 이유 |
|---|---|---|
| `@Transactional` 그대로 | Redis 저장이 DB commit 전 | 위의 불일치 문제 |
| method-level `@Transactional` 제거 | class-level `readOnly=true` 합류 → Hibernate flush MANUAL | register 의 save 가 의도와 다르게 동작 |
| `@TransactionalEventListener(AFTER_COMMIT)` | 응답 *반환 후* 이벤트 실행 | 두 가지 문제(아래) → **strict 정책과 양립 불가** |
| `Propagation.NOT_SUPPORTED` ✓ | signUp 트랜잭션 없음 / register 자체 트랜잭션으로 commit / issue 는 DB commit 이후 | 행위 정확 + strict 호환 |

**AFTER_COMMIT 은 진지하게 검토했다가 제외** — 검토 과정에서 발견한 두 문제:
1. 응답 반환 *후* 이벤트 실행이라 Redis 실패를 클라이언트 응답에 반영 못 함 → strict 정책 위반
2. 회원가입만 성공하고 Redis 저장이 실패하면 *후속 분기 처리*가 필요 → 코드 복잡도 증가

→ **위 Redis strict 결정의 "soft fail 은 분기 비용도 손해" 통찰과 같은 패턴**이 여기서도 나타난다. 부분 실패가 만드는 분기 부담을 회피하는 게 *일관된 사고 흐름*.

**핵심 통찰**: AFTER_COMMIT 패턴은 "성공 보장 X, 실패해도 클라이언트에 안 알려도 OK"인 경우에만 쓸 수 있다. 회원가입은 *Redis 저장 실패를 클라이언트에 즉시 알려야* 하므로(= strict 정책) 같은 트랜잭션 흐름 안에서 동기로 처리돼야 한다.

**트레이드오프 — 부분 실패 시나리오**:
- member 가 DB 에 commit 됐는데 Redis 저장 실패 → 클라이언트는 500
- 사용자가 같은 이메일로 재시도 → `DUPLICATE_EMAIL` 응답 (이미 DB 에 있음)
- 또는 로그인 시도 → 비밀번호 알면 성공
- 즉 *유령 회원*은 안 만들어지고, 사용자 입장에서도 *복구 가능한 상태*

**기존 패턴과의 일관성**: `OrderCreateService.createOrder()` 가 동일하게 `NOT_SUPPORTED` 패턴. 코드베이스 안에 *이미 검증된 패턴*이 있었다.

**class-level `@Transactional(readOnly = true)` 패턴 — 통일성 위해 현재 유지, 향후 리팩토링 대상**:
- 현재는 코드 전반의 통일성을 위해 *클래스 단위에 `readOnly=true` 를 붙이고 method-level 에서 override* 하는 패턴을 쓴다.
- 다만 이 패턴의 한계 — **트랜잭션 경계가 한눈에 안 들어온다**. 어떤 메서드는 트랜잭션이 없어야 하고, 어떤 메서드는 readOnly 이면 안 되는데, 클래스 단위 애너테이션이 *기본값을 숨긴다*.
- → *향후 리팩토링 대상*: 클래스 단위에 `@Transactional` 을 안 붙이고 각 메서드에 명시적으로 부여하는 방향. signUp 이 `NOT_SUPPORTED` 로 class-level 을 override 하는 현 구조 자체가 *어색함의 신호*. (실제로 이 세션 이후 응용 Service 의 `@Transactional` 을 method-level 로만 두는 방향이 컨벤션으로 정해지고, 이어 class-level `@Transactional` 은 전 도메인에서 폐지됐다.)

### 4. `RefreshTokenStore.delete()` 제거 — 로그아웃 미구현의 흔적

`RefreshTokenStore` 인터페이스에서 `delete(Long memberId)` 메서드 자체를 *삭제*. 로그아웃 기능이 미구현이라 호출부가 없었다.

**근거**:
- CLAUDE.md 원칙: "불필요한 추상화와 과한 설계를 피한다"
- 호출부 없는 인터페이스 메서드 = 잠재적 혼란 (다른 사람이 보면 "어디서 호출되겠지" 하고 추측)
- Git history 가 *제거 이유*와 *과거 존재*를 기록하므로 정보 손실은 없다
- 로그아웃 구현 시 *그 PR 안에서* `delete()` 와 Redis 실패 정책을 *함께* 설계하는 게 더 안전

**로그아웃 미구현 — 의도된 범위 제한**:
- 토큰 만료(30분 access + 7일 refresh + RTR)로 *실용적으로 충분히 커버*된다고 판단.
- 어려운 기능이 아닌데 *지금 크게 의미 있는 기능이 아닌 것을 도입하기 싫었다* — 필요한 시점에 도입하면 된다.

**로그아웃 도입 시점의 가치 — "사용자의 의식적 종료 의사 표현"**:
- 예: 다른 기기에서 로그인한 경우 명시적으로 로그아웃하고 싶을 때.
- RTR 이 *다른 기기 로그인 시 이전 기기 세션 자동 무효화*는 이미 해주므로(결정 1), logout API 의 *추가 가치*는 사용자의 *의식적 종료 의사 표현*에 한정된다 — 자동/자연 발생이 아닌 *의도적 트리거*만 logout 의 영역.

**향후 과제**: 로그아웃 구현 시 `delete()` 재추가 + Redis 실패 정책 *재*결정 필요. 회원가입/로그인의 strict 와 같은 정책이 로그아웃에도 적용될지는 *별도 판단* (로그아웃은 보안 목적이라 "Redis 안 되어도 일단 access 비활성화"가 의미 있을 수도).

### 5. auth ↔ member 분리 — auth 가 member.application 에 위임하는 구조

`AuthSignUpService` 가 `MemberRegistrationService.register()` 를, `AuthLoginService` 가 `MemberQueryService.findByEmail()` 을 호출. *auth 가 member 의 application 레이어에 의존*한다.

**왜 auth 가 직접 `MemberRepository` 를 안 쓰나** — 대안 비교:
- **(A) auth → `MemberRepository` 직접 의존**: member 의 *도메인 규칙*(이메일 중복 검증, `Member.createUser` 입력 검증)이 auth 에서 *재구현*되거나 *우회*됨
- **(B) auth → `member.application` service 의존 ✓**: member 도메인 무결성 규칙이 *register 안에 캡슐화*된 채 유지

**의미**:
- `member` 는 "회원 도메인 정합성 owner" — 이메일 중복, role 기본값, 입력 검증 책임
- `auth` 는 "회원 도메인을 사용해 인증 유스케이스를 조립하는 layer" — 비밀번호 hashing 은 auth 책임 (BCrypt 해시된 결과만 member 에 넘긴다)
- **`password hash` 가 member 에 들어가는 경계 분담이 흥미로움**: raw password 는 auth 에만 노출, member 는 *해시만* 받아서 저장

**솔직한 분리 의도 메모 — 강한 결정적 이유는 없음**:
- 명시적이고 결정적인 동기가 있었다기보다 *약한 직관*에 가깝다.
- 그나마 가장 가까운 이유: **member 는 인증 외에도 주문·배송 등 다른 도메인에 엮이는 핵심 도메인**이라 인증과 분리해두면 자연스럽다.
- 아래 security↔auth 분리와 같은 결 — 강한 의식적 설계라기보다 *DDD 학습 과정에서 자연 도출된 헥사고날 구조*.

**향후 과제 — OAuth 도입 시 재고민**:
- `MemberRegistrationCommand` 가 *이미 해시된 passwordHash* 를 받는 구조. OAuth/소셜 로그인 같은 *비밀번호 없는* 가입이 추가되면 이 command 구조 재설계 필요.
- 현 시점 예상 방향은 미정. OAuth 도입 시점에 결정 (password optional / 별도 credential 엔티티 분리 / 다른 방향).
- OAuth 도입 계획 자체는 있다.

### 6. security ↔ auth 분리 — Filter/Interceptor vs UseCase

`security` 가 별도 패키지로 존재하고 `auth.application.TokenAuthenticationService` 만 호출한다.

**왜 security 가 별도 패키지인가?**
- HTTP 요청 경계의 *어댑터*는 도메인 흐름과 다른 *횡단 관심사*. 인증 필요 경로 화이트리스트, JWT 추출, ThreadLocal 관리, `@RequireRole` 인터셉터 등은 *서블릿 인프라* 책임이다.
- 만약 `auth` 안에 Filter 가 같이 있으면 `auth` 가 *도메인이자 인프라*가 됨 → 책임 비대.
- security 는 *모든 도메인의 인증/인가*를 책임지므로 *수평적* 패키지. auth 는 인증 *유스케이스*라는 *세로* 패키지.

**구체적 책임 분담**:

| 역할 | 위치 |
|---|---|
| 토큰 자체의 발급/검증 알고리즘 | `auth.infrastructure.jwt` |
| 토큰 검증 *유스케이스* (memberId 파싱 등) | `auth.application.TokenAuthenticationService` |
| HTTP 요청에서 토큰 *추출* | `security.filter.JwtAuthenticationFilter` |
| 인증 결과 *전파* (ThreadLocal) | `security.context.AuthenticationContext` |
| role 기반 인가 | `security.interceptor.AuthorizationInterceptor` |
| Controller 에 memberId 주입 | `security.resolver.AuthenticatedMemberIdArgumentResolver` |

**ThreadLocal 영향 범위를 *Filter ~ Controller 입구*로 한정 — 비동기 호환을 위한 의식적 설계**:
- `AuthenticationContext`(ThreadLocal)에 저장된 memberId 는 *Controller 에서 추출*되어 이후 모든 로직에는 *DTO/파라미터로 명시 전달*.
- 즉 ThreadLocal 은 *HTTP 요청 진입 → Controller 입구*까지의 *짧은 구간*만 책임진다. 그 이후 흐름은 ThreadLocal 에 의존하지 않는다.
- 향후 비동기(`@Async` / virtual thread)가 도입돼도 비동기 작업으로 *DTO/파라미터에서 memberId 를 추출해 넘기므로* → ThreadLocal 이 깨질 위험이 없다.
- 즉 *비동기 도입 시 별도 작업이 거의 필요 없는* 구조. (같은 시기 별도로 다룬 traceId 의 비동기/이벤트 경계 전파와는 결이 다르다 — traceId 는 자동 전파가 가치라 이벤트 객체 동봉·Outbox 컬럼 저장 같은 *명시적 동봉 메커니즘*으로 경계를 넘겨야 했지만, memberId 는 *데이터*라 애초에 명시 전달이 자연스럽다.)

**WHITELIST 가 Filter 안에 하드코딩** — `Set.of("/auth/login", "/auth/signup", "/auth/reissue", "/products", "/payments/naverpay/return")` + `/products/` prefix. 코멘트 "// 나중에 설정 파일로 분리하기".
- 분리 안 한 이유: *변경 빈도가 낮고*, 우선 *미뤄둔 것*. 추후 정리 대상.

**Spring Security 를 의도적으로 안 씀 — 학습 목적이 제일 큼**:
- 학습 비용 회피 + 과잉 추상화 회피도 이유이지만, **결정적 동기는 학습 목적**.
- 직접 Filter + Interceptor 를 만들어보고 *불편함을 느껴야* Spring Security 의 장점을 *깊이 느낄 수 있고*, 그제서야 의존성을 쓰는 의미가 깊어진다는 판단.
- 즉 *추상화 도구를 도입하기 전에 그것이 푸는 문제를 먼저 직접 경험*하는 학습 접근.

---

## 배운 것 — 도메인 경계에서

### `@Transactional` 의 외부/내부 트랜잭션 *commit 시점* 차이

REQUIRED 전파의 핵심 함정: "내부 메서드가 *반환*되어도 DB 는 미커밋". 따라서 *내부 메서드 반환 직후 외부 시스템(Redis)을 다루면* 그 시점은 **DB commit 이전**이다. 이게 회원가입 트랜잭션 분리 결정의 출발점.

→ "트랜잭션 안에서 외부 시스템과 상호작용하는 코드"는 항상 *commit 시점*을 의식해야 한다는 일반 규칙.

### "동작하는 것처럼 보이지만 실제로는 망가진" 은 *항상* 즉시 실패보다 나쁘다

Redis strict 정책의 본질. 인증 시스템에서 silent 한 부분 실패는 사용자에게 *예측 불가능한 시점에 터지는* 버그로 인지된다.

→ 이 원칙은 인증 밖에도 적용된다. 결제에서 멱등 재요청의 금액이 어긋날 때 조용히 진행하지 않고 *명시적 예외로 거부*하는 결정도 같은 정신이다.

### 미사용 인터페이스 메서드는 *제거*가 *유지*보다 정직

`RefreshTokenStore.delete()` 제거 결정. 사용처 없는 추상화는 "여기서 호출되겠지"라는 *잘못된 추론*을 유도한다. Git history 가 *과거의 존재*를 기억하므로 정보 손실은 0.

→ 같은 원칙: 안 쓰는 helper, 안 쓰는 설정 값, "혹시 모르니 남겨둔" 코드 모두.

### 도메인 *application 레이어*가 다른 도메인의 의존 surface

`auth → member.application`. infrastructure 의 repository 는 절대 가로질러 호출하지 않는다. → "도메인 간 협력은 application service 끼리 한다"는 일관된 컨벤션.

---

## 미해결·열린 질문

- **Redis HA (Sentinel / Cluster)** — strict 정책의 *전제*인데 아직 단일 노드. 운영 단계에서 우선순위.
- **로그아웃 기능** — `RefreshTokenStore.delete()` 와 Redis 실패 정책을 *함께* 설계 필요 (회원가입/로그인처럼 strict 인지, 아니면 별도 정책인지).
- **WHITELIST 설정 분리** — 코멘트로만 남은 TODO. 새 공개 경로 추가 시마다 코드 수정.
- **다중 디바이스 / 다중 세션** — refresh token 키가 `refresh:{memberId}` 단일 키라 디바이스별 세션 불가. 디바이스 ID 또는 sessionId 도입 시 키 구조 변경 필요.
- **OAuth / 소셜 로그인** — `MemberRegistrationCommand.passwordHash` 가 필수라 비밀번호 없는 가입 경로 미지원. 도입 시 member 도메인 모델링 재검토.
- **비동기 경계의 memberId 전파** — `AuthenticationContext` 가 `ThreadLocal`. traceId 는 별도 결정에서 비동기 전파를 명시적 동봉으로 풀었지만, memberId 는 데이터라 지금은 명시 전달로 충분하다고 봤다. 다만 실제 비동기 도입 시 이 전제가 맞는지 재확인 필요.
- **admin 승격 API 부재** — 신규 가입은 모두 `ROLE_USER` 고정. admin 은 DB 직접 수정으로만 만들 수 있는 상태.

**다시 본다면 (회고 의사)** — 시간이 지난 뒤 다음을 되짚어볼 생각:
- JWT + Redis 의 *실제 운영* 경험 (Redis 장애가 실제로 난 적 있는지, strict 정책의 *체감 비용*)
- access/refresh 만료 시간을 다시 정한다면
- "한 회원 = 한 활성 refresh" 정책의 한계 체감 여부
- Spring Security 를 안 쓴 결정에 대한 현재 시점 평가
- 로그아웃 미구현이 실제로 불편했는지
