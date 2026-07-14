---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, jwt, jjwt, authentication, dead-code, footgun, token-validation]
created: 2026-07-11
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-07-11-servlet-filter-exception-and-jwt-catch]]"
---

# JWT 예외 catch footgun — bare SecurityException이 java.lang으로 조용히 바인딩

## 정의 — bare SecurityException이 java.lang으로 바인딩되는 함정

JWT 파싱 실패를 잡는 catch 절이 `catch (UnsupportedJwtException | MalformedJwtException | SecurityException e)`였는데, `SecurityException`을 import하지 않아 그 이름이 **JDK 기본 `java.lang.SecurityException`**으로 조용히 바인딩됐다. 같은 단순 이름이 JDK와 라이브러리 양쪽에 존재할 때 전형적으로 나는 함정으로, import 누락이 컴파일 에러가 아니라 "무관한 타입을 잡는 죽은 catch"로 귀결된다.

## jjwt 예외 계층(실측)과 위조 예외가 상속하는 것

JWT 라이브러리(jjwt 0.11.5)는 서명 위조 시 `io.jsonwebtoken.security.SignatureException`을 던진다. 실제 jar로 확인한 계층은:

```
io.jsonwebtoken.security.SignatureException
  → io.jsonwebtoken.SignatureException (레거시, deprecated)
  → io.jsonwebtoken.security.SecurityException
  → io.jsonwebtoken.JwtException (RuntimeException)
```

즉 위조 예외는 **라이브러리 자신의** `io.jsonwebtoken.security.SecurityException`을 상속할 뿐 **JDK `java.lang.SecurityException`을 전혀 상속하지 않는다.** 따라서 JDK `SecurityException`으로 잡던 그 catch 절은 위조 예외를 못 잡고 통과시켰다(사실상 죽은 catch). 개발자가 라이브러리의 `SecurityException`을 import해서 잡았다면 위조가 잡혔을 것이다 — 진짜 함정은 import 누락으로 bare 이름이 무관한 JDK 타입으로 바인딩된 것이다.

## 영향 — 위조 refresh 500 / 위조 access 401

- 위조 서명된 **refresh 토큰**: 미포착으로 올라가 전역 예외 핸들러의 최후 `catch(Exception)`에 걸려 **500**이 됐다. 인증 실패인데 서버 오류로 위장 + 에러 로그 스팸.
- 위조 **access 토큰**: 필터의 제네릭 catch에 걸려 제네릭 **401**(만료/무효 세부 코드가 아님).

두 경로 모두 [[security-common-leaf-재편과-토큰-포트-네이밍]]에서 "만료/무효/위조를 구분해 유지"하기로 한 토큰 실패 코드 체계를 무력화한다 — 위조가 제대로 위조 코드로 분기되지 못했다.

## 실증 — 다른 키 서명 토큰으로 SignatureException 확인

정상 서명키가 아닌 다른 키로 서명한 토큰을 만들어 검증기에 넣는 임시 테스트로 확인 → 실제로 `io.jsonwebtoken.security.SignatureException`이 던져짐을 관측했다(계층 가정을 코드로 검증).

## 해결 — JwtException 베이스로 catch + 만료 우선 분기 + 회귀 테스트

catch를 JWT 라이브러리 베이스 예외 `io.jsonwebtoken.JwtException` 하나로 바꿨다(라이브러리의 `security.SecurityException`보다 더 위 베이스라, 서명 위조뿐 아니라 형식오류·미지원 등 모든 JWT 오류를 함께 덮어 더 안전). 만료 예외 `ExpiredJwtException`은 그 위에서 먼저 잡아 만료 코드로 분기, 그 아래에서 나머지 JWT 오류를 무효 코드로. 위조 토큰 회귀 테스트(access·refresh 둘 다 → 무효 코드)를 추가했다. 이 수정 뒤에는 위조 토큰조차 구체 인증 예외가 되므로, [[서블릿-필터-예외-처리와-에러-디스패치]]에서 제거한 필터의 제네릭 `catch(Exception)`이 완전히 도달 불가가 된다.

## 교훈·관련 링크

- **JDK에 같은 단순 이름이 있는 예외(`SecurityException` 등)는 import를 빠뜨리면 조용히 `java.lang` 쪽으로 바인딩되는 footgun이다.** 라이브러리 예외를 잡을 땐 그 라이브러리의 예외 계층(베이스 타입)을 실제로 확인하고 잡아라.
- 이 결함은 **자동 코드 리뷰어 둘이 모두 놓쳤다** — "bare 이름이 java.lang으로 바인딩된다 + JWT 라이브러리 예외 계층" 두 사실을 동시에 알아야 보이는 부류라, 사람+에이전트가 적대적으로 뒤져 찾았다. 이 앙상블 교훈은 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]과 결이 같다.
- 최초 구현부터 있던 pre-existing 결함이었다. "옮겨오는(이미 있던) 코드도 별도의 적대적 버그 헌팅 대상"이라는 원칙([[서블릿-필터-예외-처리와-에러-디스패치]]의 리팩터 교훈)으로 드러났다.
- 관련 catch절 함정의 다른 형태: [[persistence-exception-노출-경계-추상수준]]의 "ArchUnit이 맨 catch절 예외 타입을 못 잡는다".

## 근거

- [[raw/sessions/backend/2026-07-11-servlet-filter-exception-and-jwt-catch]]
