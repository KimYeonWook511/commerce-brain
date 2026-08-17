---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, exception-strategy, infra-failure, cache, redis, refresh-token, auth, adapter, restcontrolleradvice, yagni, logging, error-code]
created: 2026-06-03
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-197-auth-redis-exception-mapping]]"
---

# Auth Redis 장애를 인프라 도메인 예외 매핑 + 도메인 전용 어드바이스로 통일

> [!warning] 부분 반전(EVOLUTION) — pr-288
> 이 노트의 응답 매핑 방식(**도메인 전용 예외 클래스 + 도메인 전용 `AuthExceptionHandler` + 500 응답**)은 이후 [[인프라-일시장애-503-예외-매핑-판단축]](pr-288)에서 뒤집혔다. 거기서는 인프라 일시장애를 **공통 도메인 예외의 자동 매핑(503)**으로 통일하고 도메인 전용 어드바이스를 폐기했다. 즉 아래 **결정 2**와 **결정 5**의 구체 메커니즘은 현재 유효하지 않다.
> 다만 **결정 1의 상위 규약 — "외부 캐시/저장소 장애는 infra adapter가 도메인 예외로 변환한다"** — 는 그대로 살아남았고 오히려 후속 재설계가 그 위에 올라탔다. 그래서 노트 전체를 superseded로 내리지 않고 상위 규약은 `accepted`로 보존하되, 응답 위치·상태코드 결정만 반전됐음을 양쪽 다 남긴다.

## 컨텍스트 — Auth만 캐시 장애 규약을 이탈

refresh token 저장소(Redis)의 장애 처리가 다른 도메인의 캐시 장애 규약과 어긋나 있었다(Issue #181 / PR #197). 직전에 주문 멱등성 캐시를 단순화하면서(PR #180, [[redis-장애-멱등캐시-fallback-boolean-예외분리]]) "외부 캐시 장애는 infra adapter가 도메인 예외로 변환하고, application/presentation이 정책을 결정한다"는 규약이 신설됐는데, Auth만 이 규약을 안 따르고 application 서비스가 Spring `DataAccessException`을 직접 catch해 비즈니스 예외로 감싸고 있었다.

동작 결함은 없었다(Redis 장애 시 응답은 그대로 500 `AUTH-500-1`). 문제는 두 가지 부담이었다. (1) application 계층이 Spring DAO 예외에 직접 의존한다 — port(`RefreshTokenStore`)가 시그니처상 기술 예외를 노출하지 않아도 *사용처 의존*으로는 노출이다. (2) 앞으로 Payment·Stock 같은 캐시 사용처가 늘 때마다 "Auth처럼 할지 다른 도메인처럼 할지"를 매번 판단해야 하는 일관성 비용이 누적된다.

## 결정 1 — infra adapter 도메인 예외 매핑으로 일원화 (상위 규약, 유지)

Auth도 다른 도메인과 같은 매핑 패턴을 따르게 했다. Redis 어댑터(`RedisRefreshTokenStore`, `RefreshTokenStore` port의 Redis 구현체)가 `save`/`get`에서 `DataAccessException`을 catch해 도메인 예외 `RefreshTokenStoreUnavailableException`으로 변환하고 ERROR + stack을 남긴다. 새 도메인 예외는 `RuntimeException`을 직접 상속한다.

- **검토한 갈래.** (현행 유지) "fallback 가능 여부"를 구조 분기점으로 두는 해석 — Auth는 장애 시 대체 흐름이 없으니 application이 직접 catch하는 게 자연스럽다. vs (패턴 통일) fallback 가능 여부와 무관하게 *매핑 단계(infra adapter)는 공통*으로 두고 *catch 위치·정책 결정만* 도메인별로 분기.
- **패턴 통일을 택했다.** fallback 가능 여부를 *구조 분기*가 아니라 *catch 위치 결정 분기*로 격하하면, 매핑 구조 자체가 모든 사용처 공통 규약으로 격상되고 도메인별 정책 차이는 catch 위치에서만 드러난다. 그러면 신규 사용처가 어느 쪽을 따를지 판단하는 비용이 사라진다.
- 이 상위 규약("외부 캐시 장애 = infra가 도메인 예외로 변환")은 이후 [[인프라-일시장애-503-예외-매핑-판단축]]과 [[persistence-exception-노출-경계-추상수준]], 그리고 [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]의 "adapter가 위반을 도메인 예외로 번역"과 한 계열이다.

## 결정 2 — 응답 매핑 위치 4안 비교 → 도메인 전용 @RestControllerAdvice (B-2, 이후 반전됨)

변환된 도메인 예외를 *어디서 받아 응답으로 매핑할지* 네 안을 비교했다.

- **(A) application이 catch해 `AuthException(INTERNAL_ERROR)`로 변환** → 기각. `AuthException`은 auth *비즈니스* 예외(`TOKEN_INVALID` 등)의 의미인데 *인프라 장애*를 이 옷에 입히면 시멘틱 부정합이고, fallback도 없는 단순 변환 두 줄이라 catch에 담을 정책 결정 사실이 없다. catch를 없애면 서비스에 happy path만 남는다.
- **(B-1) `common`의 `GlobalExceptionHandler`가 직접 받기** → 기각. `common → auth` 역의존을 신설하고, 다른 도메인도 같은 사례를 만나면 `common`이 모든 도메인 예외를 import하는 누적 부담이 생긴다.
- **(B-2, 채택) 도메인 전용 `@RestControllerAdvice` 신설** — `com.commerce.auth.exception.AuthExceptionHandler`를 만들어 `RefreshTokenStoreUnavailableException`을 `AUTH-500-1`로 매핑. 의존 방향이 정합(`auth → common`)하고 Auth 응답 정책이 Auth 모듈에 응집된다.
- **(B-3) `RefreshTokenStoreUnavailableException extends CustomException`으로 자동 매핑** → 기각. 상속하면 `GlobalExceptionHandler.handleCustomException`이 자동 응답 매핑을 해버려 "인프라 장애는 도메인 예외로 두고 정책은 명시적으로 결정"이라는 catch 의도를 우회한다. 도메인 예외를 `RuntimeException` 직접 상속으로 둔 이유와 정면 충돌.

> 이 B-2 채택(도메인 전용 어드바이스 + 500)이 pr-288에서 공통 자동매핑 503으로 반전된 부분이다. 위 warning 콜아웃 참조.

## 결정 3 — application의 인프라 장애 log.error 제거 (로그 단일화)

catch를 걷어내면서 application 서비스(`AuthTokenIssueService`, `AuthTokenReissueService`)에 있던 인프라 장애 `log.error`도 없앴다. 같은 장애를 infra adapter가 이미 ERROR + stack으로 기록하므로 중복이다. 같은 장애에 ERROR가 두 번 찍히면 stack trace도 두 번 남아 근원 추적이 오히려 느려진다. 로그를 *기술적 사실을 처음 인지한 지점*(infra adapter) 한 곳으로 단일화했다.

## 결정 4 — 공통 StoreUnavailable 베이스 추출 보류 (YAGNI)

`OrderIdempotencyStoreUnavailableException`과 이번 `RefreshTokenStoreUnavailableException`이 같은 구조(`RuntimeException(Throwable cause)`)라 공통 베이스 추출이 가능했지만 보류했다.

- **근거:** 사용처가 2곳뿐이고 *공통 catch* 시나리오가 없다 — 주문 쪽은 application에서 catch(fallback 진입), Auth는 presentation에서 catch(응답 매핑)라 애초에 한데 잡을 일이 없다. 두 도메인이 같은 베이스를 공유하면 한쪽 변경이 다른 쪽에 새는 *우연한 결합*이 생긴다.
- **추출 트리거를 명시:** Payment·Stock 등 캐시 사용처가 3곳 이상 등장하고 공통 catch가 실제로 필요해지는 시점에 재검토한다. 그때는 부모 추가 + import 조정뿐이라 IDE 지원으로 비용이 낮다 — N=1 도메인 전용 → 사용처 누적 후 공통 추출의 과도기 패턴.

## 결정 5 — 리뷰 반영: @Order(HIGHEST_PRECEDENCE)로 만능 fallback보다 먼저 매칭

`AuthExceptionHandler`에 `@Order(Ordered.HIGHEST_PRECEDENCE)`를 붙였다. `GlobalExceptionHandler`에는 `@ExceptionHandler(Exception.class)` 만능 fallback이 있어 `RefreshTokenStoreUnavailableException`(RuntimeException)도 잡을 수 있다. 여러 `@RestControllerAdvice`는 `@Order` 순으로 먼저 시도되고(**어드바이스 간 정렬이 어드바이스 내부 specificity보다 우선**), 우선순위가 같으면 비결정적이다. `@Order` 기본값이 `LOWEST_PRECEDENCE`라 도메인 어드바이스에 HIGHEST를 주는 것이 만능 fallback보다 확실히 먼저 잡히게 하는 유일한 실효 방안이었다. (이 어드바이스 자체가 pr-288에서 폐기되어 현재는 무효.)

## 결정 6 — 리뷰 반영: TTL stub 값 지역 변수 분리 테스트

`RedisRefreshTokenStoreTest`에서 `save`의 TTL 검증 시, `jwtProperties.getRefreshExpiration()` stub 값을 지역 변수(`refreshExpirationMs`)로 뽑아 stub 설정과 verify(`eq(Duration.ofMillis(refreshExpirationMs))`)가 *같은 원천*을 참조하게 했다. `any()`(검증 공백)와 `eq(하드코딩 상수)`(fixture 변경에 brittle) 사이의 절충으로, fixture가 바뀌어도 안 깨지면서 TTL 계산 로직 회귀는 잡는다. `@Mock` 인스턴스를 직접 만들어 Spring context 없이 검증(=`@Value`·`application.yml`·`.env.local` 무관, stub만 진실 원천).

## 트레이드오프·미해결

- **단일 어드바이스 → 도메인별 어드바이스 컨벤션 전환 부담.** 한 사례로 컨벤션을 바꾸는 셈이지만 의존 방향 보존 가치와 확장성이 더 크다고 봤다. (pr-288이 이 컨벤션 자체를 되돌렸으므로, 이 트레이드오프는 결과적으로 재정산됐다.)
- **공통 베이스 추출**은 캐시 사용처 3곳+ & 공통 catch 필요 시점의 별도 작업.
- **Auth에 fallback 가능한 캐시 사용처가 생기면** catch 위치 결정을 재검토 — fallback 진입은 application catch가 정답이라 그때는 주문 쪽 패턴을 부분 도입.
- **infra adapter ERROR 로그만으로 운영 인지가 부족하면** 핸들러 로그/메트릭 도입 검토.
- 관련: strict 정책 대신 soft-fail을 택한 [[redis-장애-strict-정책-soft-fail-기각]], refresh token 저장소 자체의 [[refreshtokenstore-delete-제거-로그아웃-미구현]]·[[jwt-redis-하이브리드-rtr-ttl-근거]].

> [!note] 같은 판단축의 후속
> 여기서 쓴 "인프라 예외를 도메인 예외로 옮기고 도메인 전용 어드바이스가 받는다"가 결제 쪽에서 한 번 더 정밀해졌다 — 무결성 위반은 한 타입으로 도착해 **타입만으로는 무엇에 부딪혔는지 알 수 없어 제약 이름으로 가른다**([[무결성위반-도메인예외-번역을-제약이름으로-가른다]]). 번역 여부의 판단축("안쪽이 그 예외를 실제로 다루는가")은 양쪽이 공유한다.

## 근거

- [[raw/sessions/backend/2026-06-03-pr-197-auth-redis-exception-mapping]]
