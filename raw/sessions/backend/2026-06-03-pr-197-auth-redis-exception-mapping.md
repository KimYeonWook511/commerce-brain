---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 197 }
---

# Auth Redis 장애 처리를 인프라 도메인 예외 매핑 + 도메인 전용 예외 어드바이스로 통일

refresh token 저장소(Redis)의 장애 처리가 다른 도메인의 캐시 장애 규약과 어긋나 있던 걸 맞춘
세션이다(Issue #181 / PR #197). 직전에 주문 멱등성 캐시를 단순화하면서(PR #180) "외부 캐시 장애는
infra adapter가 도메인 예외로 변환하고, application/presentation이 정책을 결정한다"는 규약이
신설됐는데, Auth만 이 규약을 안 따르고 application 서비스가 Spring `DataAccessException`을 직접
catch해 비즈니스 예외로 감싸고 있었다. 동작 결함은 없었다(Redis 장애 시 응답은 그대로 500
`AUTH-500-1`). 문제는 application 계층이 Spring DAO 예외에 직접 의존한다는 점, 그리고 앞으로 Payment·
Stock 같은 캐시 사용처가 늘 때마다 "Auth처럼 할지 다른 도메인처럼 할지"를 매번 판단해야 하는
일관성 부담이 쌓인다는 점이었다.

## 결정한 것

### 1. 외부 캐시 장애를 infra adapter의 도메인 예외 매핑으로 일원화

Auth도 다른 도메인과 같은 매핑 패턴을 따르게 했다. refresh token 저장소의 Redis 구현체
(`RedisRefreshTokenStore`, `RefreshTokenStore` port의 Redis 어댑터)가 `save`/`get`에서
`DataAccessException`을 catch해 도메인 예외 `RefreshTokenStoreUnavailableException`으로 변환하고
ERROR + stack을 남긴다. 새 도메인 예외는 `RuntimeException`을 직접 상속한다.

- **검토한 갈래.** (현행 유지) "fallback 가능 여부"를 구조 분기점으로 두는 해석 — Auth는 장애 시
  대체 흐름(fallback)이 없으니 application이 직접 catch하는 게 자연스럽다. vs (패턴 통일)
  fallback 가능 여부와 무관하게 *매핑 단계(infra adapter)는 공통*으로 두고, *catch 위치·정책 결정만*
  도메인별로 분기시킨다.
- **패턴 통일을 택했다.** 근거: (1) application이 Spring `DataAccessException`에 직접 의존하면
  port(`RefreshTokenStore`)가 시그니처상으로는 기술 예외를 노출하지 않아도 *사용처 의존*으로는
  노출이다. infra 매핑을 거치면 application/presentation은 도메인 예외만 알면 된다. (2) 미래에
  캐시 사용처가 추가될 때 Auth만 다른 패턴으로 남으면 신규 사용처가 어느 쪽을 따를지 매번 판단해야
  한다 — 통일하면 그 의사결정 비용이 사라진다. (3) Auth까지 같은 매핑을 따르면 "외부 캐시 장애 =
  infra가 도메인 예외로 변환"이 모든 사용처가 따르는 규약으로 격상되고, fallback 가능 여부는 그
  하위의 정책 결정 분기로 내려간다.
- **동작은 보존.** 응답 코드(500 `AUTH-500-1`)는 그대로 유지했다. 감수한 비용은 도메인 예외 클래스
  1개 신설과, 예외 처리 전략 문서의 캐시 장애 섹션을 "fallback 가능 여부 = 정책 결정 분기"로
  재정리하는 것.

### 2. catch/응답 매핑 위치 — 매핑 방식 4안 비교 후 도메인 전용 `@RestControllerAdvice` 채택

매핑 패턴을 통일하기로 한 뒤에도, 변환된 도메인 예외를 *어디서 받아 응답으로 매핑할지*가 남았다.
네 가지를 비교했다.

- **(A) application이 catch해 `AuthException`으로 변환** — application이 `RefreshTokenStoreUnavailableException`을
  catch해 `AuthException(INTERNAL_ERROR)`로 바꿔 던지고, `GlobalExceptionHandler`가 자동 매핑해 500.
  기각 이유 두 가지: `AuthException`은 auth *비즈니스* 예외(`TOKEN_INVALID`,
  `REFRESH_TOKEN_NOT_FOUND` 등)의 의미인데 *인프라 장애*를 이 옷에 입히는 건 시멘틱 부정합이고,
  Auth의 catch 본문은 fallback도 없이 *단순 변환 두 줄*이라 (fallback 진입 같은) 정책 결정 사실을
  담지 못해 가치가 없다. → 기각. application의 인프라 장애 try-catch를 없애면 서비스에 happy path만
  남는다.
- **(B-1) `common`의 `GlobalExceptionHandler`가 직접 받기** — 기존 단일 어드바이스에
  `@ExceptionHandler(RefreshTokenStoreUnavailableException.class)` 한 줄 추가. 가장 작은 변경이지만
  `common → auth` 역의존을 신설한다. 다른 도메인도 같은 사례를 만나면 `common`이 모든 도메인 예외를
  import하게 돼 부담이 누적된다. → 기각.
- **(B-2, 채택) 도메인 전용 `@RestControllerAdvice` 신설** — `com.commerce.auth.exception.AuthExceptionHandler`
  (`@RestControllerAdvice`)를 만들어 `RefreshTokenStoreUnavailableException`을 `AUTH-500-1` 응답으로
  매핑한다. `GlobalExceptionHandler`는 손대지 않는다. 의존 방향이 정합(`auth → common`)하고, 미래에
  다른 도메인도 인프라 장애 직접 매핑이 필요하면 같은 컨벤션(도메인 모듈 안 어드바이스 신설)으로
  확장 가능하다. Auth 응답 정책(`AuthErrorCode` 사용)이 Auth 모듈 안에 응집된다.
- **(B-3) `RefreshTokenStoreUnavailableException extends CustomException`으로 자동 매핑** — 추가 핸들러
  불필요. 하지만 `CustomException`을 상속하면 `GlobalExceptionHandler.handleCustomException`이 자동
  응답 매핑을 해버려, "인프라 장애는 도메인 예외로 두고 정책은 명시적으로 결정한다"는 catch 의도를
  우회하는 구조가 된다. 도메인 예외를 `RuntimeException` 직접 상속으로 둔 이유와 정면 충돌 → 기각.

**트레이드오프:** 단일 어드바이스 컨벤션에서 *도메인별 어드바이스* 컨벤션으로 넘어가는 전환 부담이
있다. 한 사례만으로 컨벤션을 바꾸는 셈이지만, 의존 방향(`common`이 도메인을 모름) 보존 가치와 미래
확장성이 더 크다고 봤다. 어드바이스가 여러 개로 늘면 우선순위·범위 한정이 필요할 수 있는데, 그건
사용처가 늘었을 때 재검토한다.

### 3. application의 인프라 장애 `log.error` 제거

catch를 걷어내면서 application 서비스(`AuthTokenIssueService`의 토큰 발급 경로,
`AuthTokenReissueService`의 저장 토큰 검증 경로)에 있던 인프라 장애 `log.error`도 함께 없앴다. 같은
장애를 infra adapter가 이미 ERROR + stack으로 기록하므로 중복이다. 같은 장애에 ERROR가 두 번 찍히면
운영 모니터링 노이즈가 커지고 stack trace도 두 번 남아 어느 게 근원인지 추적이 오히려 느려진다.
로그를 *기술적 사실을 처음 인지한 지점*(infra adapter) 한 곳으로 단일화했다.

### 4. `StoreUnavailableException` 공통 베이스 클래스는 추출하지 않는다

`OrderIdempotencyStoreUnavailableException`(주문 멱등성 캐시용)과 이번
`RefreshTokenStoreUnavailableException`이 같은 구조(`RuntimeException(Throwable cause)`)라 공통 베이스
추출이 가능했지만, YAGNI로 보류했다.

- **근거:** 사용처가 2곳뿐이고 *공통 catch* 시나리오가 없다 — 주문 쪽은 application에서 catch(fallback
  진입), Auth는 presentation에서 catch(응답 매핑)라 애초에 한데 잡을 일이 없다. 두 도메인이 같은
  베이스를 공유하면 한쪽 변경이 다른 쪽에 새는 *우연한 결합*이 생기는데, 지금은 패키지 분리로 격리돼
  있다.
- **추출 트리거는 명시해뒀다:** Payment·Stock 등 캐시 사용처가 3곳 이상 등장하고 공통 catch가 실제로
  필요해지는 시점에 베이스 추출을 재검토한다. 그때는 기존 예외의 부모 추가 + import 조정을 함께 하면
  되는데 변경 범위가 작고 IDE 지원이 돼 비용이 낮다.

### 5. 리뷰 반영 — 도메인 어드바이스에 최고 우선순위(`@Order(HIGHEST_PRECEDENCE)`) 부여

리뷰를 받아 `AuthExceptionHandler`에 `@Order(Ordered.HIGHEST_PRECEDENCE)`를 붙였다.
`GlobalExceptionHandler`에는 `@ExceptionHandler(Exception.class)` 만능 fallback이 있어
`RefreshTokenStoreUnavailableException`(RuntimeException)도 잡을 수 있다. 여러
`@RestControllerAdvice`는 `@Order` 순으로 먼저 시도되는데(어드바이스 간 정렬이 어드바이스 내부
specificity보다 우선), 우선순위가 같으면 어느 어드바이스가 먼저 매칭되는지가 비결정적이다.
`@Order` 기본값이 `LOWEST_PRECEDENCE`라 `GlobalExceptionHandler`에 LOWEST를 명시해도 차별화 효과가
없고, *도메인 어드바이스에 HIGHEST를 주는 것*이 도메인 전용 핸들러가 만능 fallback보다 확실히 먼저
잡히게 하는 유일한 실효 방안이었다.

### 6. 리뷰 반영 — TTL 검증을 stub 값 지역 변수로 분리

`RedisRefreshTokenStoreTest`에서 `save`가 올바른 TTL로 저장하는지 검증할 때, TTL 원천이 되는
`jwtProperties.getRefreshExpiration()` stub 값을 지역 변수(`refreshExpirationMs`)로 뽑아 stub 설정과
verify(`eq(Duration.ofMillis(refreshExpirationMs))`) 양쪽이 *같은 원천*을 참조하게 했다. `any()`(검증
공백)와 `eq(하드코딩 상수)`(fixture 변경에 brittle) 사이의 절충으로, fixture가 바뀌어도 깨지지 않으면서
"TTL 계산 로직"의 회귀는 잡는다.

## 배운 것

- **패턴 분기점은 구조가 아니라 정책 결정 내용일 수 있다.** fallback 가능 여부를 *구조 분기*로 두면
  사용처 추가마다 의사결정 비용이 누적된다. 그걸 *catch 위치 결정 분기*로 격하하면 매핑 구조 자체는
  공통 규약으로 격상되고, 도메인별 정책 차이는 catch 위치에서만 드러난다.
- **인프라 장애를 비즈니스 예외로 감싸지 않는 게 시멘틱적으로 정직하다.** 인프라 장애는 도메인 예외
  그대로 presentation까지 올려 매핑하는 게 책임 분리에 맞는다. 비즈니스 예외 타입에 억지로 입히면
  "정책 결정 사실"도 없이 의미만 흐려진다.
- **`common` 모듈의 도메인 의존 회피 가치는 사용처가 늘수록 커진다.** 한 사례만 보면 `common`에
  핸들러 한 줄 추가가 최소 변경이지만, 사용처가 늘면 `common`이 모든 도메인을 import하는 누적 부담이
  생긴다. 도메인 전용 어드바이스 컨벤션을 일찍 도입하는 편이 낫다.
- **`@RestControllerAdvice` 매칭 순서.** 어드바이스들은 `@Order`로 정렬돼 *어드바이스 간 순서가
  어드바이스 내부 specificity보다 우선*하고, 우선순위가 같으면 비결정적이다. 기본값이 LOWEST라
  만능 fallback(`GlobalExceptionHandler`)보다 확실히 먼저 잡히려면 도메인 어드바이스에 HIGHEST를
  명시해야 한다.
- **mock 기반 외부 의존 값 검증.** stub 값을 지역 변수로 분리해 SUT와 verify가 같은 원천을 보게 하면
  fixture 변경에 brittle하지 않으면서 계산 로직 회귀는 잡는다 — `any()`와 `eq(하드코딩)`의 중간.
- **`@Component` properties의 단위 테스트는 Spring context 없이.** `@Mock`으로 인스턴스를 직접
  만들면 `@Value` 주입·`application.yml`·`.env.local`이 모두 무관해지고 stub 값만 진실 원천이 된다.
- **과도기 추상화 — N=1에선 도메인 전용, N≥2/N≥3 시점에 공통 추출.** 사용처 1곳에서 섣불리 공통 베이스를
  뽑지 않고 도메인 패턴으로 일단 정착시키는 게 premature abstraction을 피한다. 예외 클래스 설계에도
  YAGNI가 그대로 적용된다 — IDE 지원과 변경 범위가 작아 나중에 뽑아도 비용 차가 미미하다.
- **중복 로그는 운영 추적을 오히려 어렵게 한다.** 같은 장애에 infra ERROR + application ERROR가 각각
  찍히면 stack trace도 두 번 남아 근원 파악이 느려진다. 로그를 *기술적 사실을 처음 인지한 지점*으로
  단일화하면 추적이 단순해진다.

## 미해결·열린 질문

- **공통 베이스 클래스 추출**은 캐시 사용처가 3곳 이상 등장하고 *공통 catch 시나리오*가 실제로
  필요해지는 시점의 별도 작업으로 미뤘다.
- **Auth에 fallback 가능한 캐시 사용처(예: 토큰 검증 캐시)가 생기면** 그때는 catch 위치 결정을
  재검토해야 한다. fallback 진입은 application catch가 정답이라, 그 시점엔 Auth에도 주문 쪽과 같은
  application catch 패턴을 부분 도입한다.
- **도메인 어드바이스가 여러 개로 늘어 우선순위 충돌이 나면** `@Order`/`basePackages` 정책 도입을
  검토한다. 지금은 `RefreshTokenStoreUnavailableException` 단일 예외 처리라 충돌이 없다.
- **infra adapter의 ERROR 로그만으로 운영 인지가 부족한 사례가 나오면** 핸들러 로그 또는 메트릭
  도입을 검토한다.
- 일반화 후보로 남긴 것: *시멘틱 분류 단위(4xx/5xx × 비즈니스/인프라)로만 공통 베이스를 추가*하는
  `common` 폭증 통제 규칙, *`@RestControllerAdvice` 매칭 순서와 `@Order`* 라는 Spring 일반 지식,
  *N=1 도메인 전용 → 사용처 누적 후 공통 추출* 의 과도기 패턴.
