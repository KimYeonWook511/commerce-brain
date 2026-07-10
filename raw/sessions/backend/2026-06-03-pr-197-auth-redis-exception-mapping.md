---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 197 }
---

## 한 일

- Auth Redis 장애 처리 패턴 통일 (Issue #181 / PR #197): 도메인 예외 매핑 + 도메인-specific `@RestControllerAdvice` 위임.
  - 신규: `RefreshTokenStoreUnavailableException`, `AuthExceptionHandler`.
  - 수정: `RedisRefreshTokenStore` 매핑 추가, application 인프라 try-catch 제거.
  - 문서: `docs/exception-strategy.md` 갱신, ADR-1~4 + 회고록.
- review 2건 처리:
  - `AuthExceptionHandler`에 `@Order(HIGHEST_PRECEDENCE)` 부여.
  - `RedisRefreshTokenStoreTest`의 TTL 검증을 stub 값 지역 변수로 분리.

정본: `commerce-backend/docs/tasks/auth-refresh-token-store-unavailable/adr.md` (ADR-1~4) + `retrospective.md`.

## 결정한 것

- 외부 캐시 장애 매핑 패턴 4안 비교 후 **B-2 (도메인-specific `@RestControllerAdvice`)** 채택.
  - A안 (`AuthException` 단순 변환): 인프라 장애를 비즈니스 예외로 감싸는 시멘틱 부정합 → 비채택.
  - B-1 (`GlobalExceptionHandler` 직접 받기): `common → auth` 역의존 → 비채택.
  - B-3 (`CustomException` 자동 매핑): catch 의도 우회 위험 → 비채택.
- application의 인프라 `log.error` 제거 — infra adapter ERROR + stack으로 운영 인지가 충분 (중복 회피).
- `@Order(Ordered.HIGHEST_PRECEDENCE)` 부여 — `GlobalExceptionHandler`의 `Exception.class` fallback과 advice 매칭 순서가 같으면 비결정적이라.
- TTL 검증 modify — hardcode 상수 (brittle) vs `any()` (검증 공백) 의 절충으로 stub 값을 지역 변수 분리.

자세한 결정 근거는 정본 ADR 참조.

## 배운 것

- **`@RestControllerAdvice` 매칭 동작**: advice들은 `@Order` 순으로 정렬되어 *advice 간 정렬 순서가 advice 내부 specificity보다 우선*. 우선순위가 같으면 어느 advice가 먼저 시도되는지가 비결정적. `@Order` 기본값이 `LOWEST_PRECEDENCE`라 `GlobalExceptionHandler`에 LOWEST를 명시해도 차별화 효과 없음 → *도메인 advice에 HIGHEST 부여* 가 유일한 실효 방안.
- **mock 기반 외부 의존 값 검증 패턴**: stub 값을 지역 변수로 분리해 SUT와 verify 양쪽이 *같은 source* 를 참조하면, *fixture 변경에 brittle하지 않으면서* *계산 로직 회귀* 는 잡힌다. `any()`(검증 공백)과 `eq(hardcode)`(brittle)의 중간.
- **`@Component` Properties의 단위 테스트**: Spring context를 안 띄움 → `@Mock`으로 mock 인스턴스 직접 생성, `@Value` 주입 / `application.yml` / `.env.local` 모두 무관. stub 값만이 진실 원천. (확인 차원의 정리.)
- **과도기 패턴 — N=1에선 도메인-specific → N≥2 시점에 common 추출**: 사용처 1곳일 때 *섣불리 common에 베이스 추출* 하지 않고 도메인 패턴으로 일단 정착 → 사용처 누적 후 통합. *premature abstraction 회피 + 사용처 누적 후 추출* 의 구체 사례. 1회 도입의 부담이 *컨벤션 누락 위험 누적* 보다 작을 때만 유효.
- **common 베이스 통합의 트레이드오프**: 도메인-specific advice는 *advice 추가 시마다 `@Order` 부여* 라는 컨벤션 누락 위험이 누적. common 베이스 통합이 장기적으로 안전하지만, *자동 매핑의 catch 의도 우회 위험* 은 `*UnavailableException` 네이밍 + 문서화로 완화. *시멘틱 분류 단위* (4xx/5xx × 비즈니스/인프라) 로만 베이스 추가하면 common 폭증 통제 가능.

## 다음 단계

(일반화 가능 패턴 후보)

- *시멘틱 분류 단위로만 common 베이스 추가* — common 폭증 통제 규칙. wiki `learnings/` 후보.
- *`@RestControllerAdvice` 매칭 순서와 `@Order`* — Spring 일반 지식. `knowledge/` 후보.
- *N=1 도메인-specific → N≥2 common 추출* 의 과도기 패턴 — wiki `learnings/` 후보.
