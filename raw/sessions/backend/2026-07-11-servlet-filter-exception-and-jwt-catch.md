---
platform: backend
author: KimYeonWook511
created: 2026-07-11
origin:
  - { type: pr, repo: commerce-backend, ref: 287 }
---

# 서블릿 필터의 예외 처리·에러 디스패치, 그리고 JWT 예외 catch footgun

배경: JWT 기반 인증을 쓰는 백엔드에서, 요청마다 토큰을 검증하는 서블릿 필터와 JWT 파싱 코드를 리팩터하다가 예외 처리 관련 결함·오해를 여러 개 발견하고 정리했다. Spring Security를 쓰지 않고 평범한 서블릿 필터(`OncePerRequestFilter`)로 자체 인증을 구현한 환경이다. 아래는 다른 백엔드에서도 그대로 써먹을 수 있는 일반 지식이다.

## 막힌 점·해결

### 1. `catch (SecurityException)`이 JWT 위조 예외를 못 잡던 footgun
- 증상: JWT 파싱 실패를 잡는 catch 절이 `catch (UnsupportedJwtException | MalformedJwtException | SecurityException e)`였는데, `SecurityException`을 import하지 않아서 그 이름이 **JDK 기본 `java.lang.SecurityException`**으로 바인딩돼 있었다. 그런데 JWT 라이브러리(jjwt 0.11.5)는 서명 위조 시 `io.jsonwebtoken.security.SignatureException`을 던진다. 실제 jar로 확인한 이 예외의 계층은 `io.jsonwebtoken.security.SignatureException → io.jsonwebtoken.SignatureException(레거시, deprecated) → io.jsonwebtoken.security.SecurityException → io.jsonwebtoken.JwtException(RuntimeException)`이라 **JDK `java.lang.SecurityException`을 전혀 상속하지 않는다.** 따라서 JDK `SecurityException`으로 잡던 그 catch 절은 위조 예외를 못 잡고 통과시켰다(사실상 죽은 catch).
- footgun의 핵심: **JWT 라이브러리에는 자기 자신의 `io.jsonwebtoken.security.SecurityException`이 있고, 위조 예외(`io.jsonwebtoken.security.SignatureException`)는 위 계층을 통해 그걸 상속한다.** 즉 개발자가 그 라이브러리의 `SecurityException`을 import해서 잡았다면 위조가 잡혔을 것이다. 진짜 함정은 import 누락으로 bare `SecurityException`이 라이브러리의 것이 아니라 **무관한 JDK `java.lang.SecurityException`으로 조용히 바인딩된 것**이다 — 같은 단순 이름이 JDK와 라이브러리 양쪽에 존재할 때 전형적으로 나는 함정.
- 영향: 위조 서명된 **refresh 토큰**은 미포착으로 올라가 전역 예외 핸들러의 최후 catch(Exception)에 걸려 **500**이 됐다(인증 실패인데 서버 오류로 위장 + 에러 로그 스팸). 위조 **access 토큰**은 필터의 제네릭 catch에 걸려 제네릭 401이 됐다(만료/무효 세부 코드가 아님).
- 실증: 정상 서명키가 아닌 다른 키로 서명한 토큰을 만들어 검증기에 넣는 임시 테스트로 확인 → 실제로 `io.jsonwebtoken.security.SignatureException`을 던짐.
- 해결: catch를 JWT 라이브러리 베이스 예외 `io.jsonwebtoken.JwtException` 하나로 바꿨다(라이브러리의 `security.SecurityException`보다 더 위 베이스라, 서명 위조뿐 아니라 형식오류·미지원 등 모든 JWT 오류를 함께 덮어 더 안전). 만료 예외 `ExpiredJwtException`은 그 위에서 먼저 잡아 만료 코드로 분기, 그 아래에서 나머지 JWT 오류를 무효 코드로. 위조 토큰 회귀 테스트(access·refresh 둘 다 → 무효 코드)를 추가.
- 교훈: `SecurityException`처럼 JDK에 같은 단순 이름이 있는 예외는 import를 빠뜨리면 조용히 `java.lang` 쪽으로 바인딩되는 footgun이다. 라이브러리 예외를 잡을 땐 그 라이브러리의 예외 계층(베이스 타입)을 실제로 확인하고 잡아라. 이 결함은 자동 코드 리뷰어 둘이 모두 놓쳤다 — 코드만 보면 자연스러워 보이고, "bare 이름이 java.lang으로 바인딩된다 + JWT 라이브러리 예외 계층" 두 사실을 동시에 알아야 보이는 부류라, 사람과 에이전트가 적대적으로 뒤져 찾았다.

### 2. 서블릿 필터에서 던진 예외는 전역 예외 핸들러(@RestControllerAdvice)에 닿지 않는다
- 원리: `@ControllerAdvice`/`@ExceptionHandler`는 요청을 컨트롤러로 라우팅하는 프론트 컨트롤러(DispatcherServlet) **안에서** 핸들러가 던진 예외만 처리한다. 서블릿 필터는 그 프론트 컨트롤러 **앞**에서 돈다. 그래서 필터 자체에서 던진 예외는 전역 핸들러가 절대 못 본다.
- 따라서: 필터 단계의 인증 실패 응답은 필터가 직접 써야 한다(전역 핸들러가 대신 못 해줌). 반대로 컨트롤러/서비스가 던진 예외는 프론트 컨트롤러 안이라 전역 핸들러가 정상 처리한다.
- 관련 결함: 처음 필터가 `filterChain.doFilter()`(=이후 필터 체인 + 컨트롤러 전체 실행) 호출을 인증 try-catch **안에** 두고 있었다. 그래서 하류 컨트롤러/서비스에서 난 예외까지 필터의 `catch`에 잡혀 401로 오처리될 수 있었다. 해결: 토큰 인증 호출만 try-catch로 감싸고(인증 실패만 401), `doFilter()`는 그 catch **밖**으로 빼서 하류 예외가 정상 전파되게 했다.

### 3. 필터마다 catch(Exception)을 두지 마라 — 예상 못한 예외는 프레임워크 에러 처리로 전파
- 필터에서 예상 못한 예외를 잡아 직접 응답하려는 유혹이 있지만, 그러면 필터마다 catch(Exception)을 달아야 하는 안티패턴이 된다. 실제로 이 프로젝트의 다른 필터들(요청 추적 ID 필터, 접근 로그 필터)은 catch 없이 `try { doFilter } finally { 정리 }`만 두고 예외를 그대로 전파한다.
- 필터에서 전파된 예외는 서블릿 컨테이너가 받아, Spring Boot가 자동 등록한 기본 에러 엔드포인트(`/error`, `BasicErrorController`)로 내부 forward한다 → 500 응답 + 로그를 프레임워크가 일괄 처리한다. 단 그 500 응답 포맷은 프레임워크 기본(`{timestamp,status,error,path}`)이라, 컨트롤러 예외가 전역 핸들러를 거쳐 나가는 우리 표준 응답 포맷(`{code,message,data}`)과는 다르다. 이 포맷 차이는 필터 계층 전체에 공통이지 특정 필터만의 문제가 아니다.
- 결론: 인증 필터의 제네릭 `catch (Exception)`(예상 못한 예외를 401로 삼키던 것)을 제거했다. 인증 실패(구체 예외)만 처리하고 나머지는 전파. 근거 — 토큰 검증기는 상태 없는(stateless) JWT 파싱이라 정상적으로는 구체 인증 예외만 던지므로, 그 제네릭 catch는 사실상 도달 불가한 죽은 코드였고 오히려 예상 못한 예외를 401로 위장했다. (앞의 §1에서 JWT 파싱 예외를 라이브러리 베이스로 제대로 잡게 고친 뒤에는, 위조 토큰조차 구체 인증 예외가 되므로 이 제네릭 catch가 완전히 도달 불가가 된다.) (정말로 필터 계층 500까지 표준 포맷으로 통일하고 싶으면, 필터마다 catch가 아니라 "최외곽 에러 처리 필터 하나"를 두는 게 올바른 방법 — 이번엔 도달 불가 경로라 불필요.)

### 4. 에러 엔드포인트(/error)가 인증을 다시 타지 않는 이유 — OncePerRequestFilter vs Spring Security
- 걱정했던 시나리오: 예외가 전파돼 `/error`로 forward되면, 그 요청이 인증 필터를 **다시** 타서 토큰을 검증하고, `/error`는 인증 예외 경로(화이트리스트)가 아니니 401로 뒤바뀌는 것 아닌가?
- 실제로는 안 그렇다. `/error`로 가는 건 일반 요청이 아니라 **ERROR 디스패치**다. 우리 필터가 상속한 `OncePerRequestFilter`에는 `shouldNotFilterErrorDispatch()`가 있고 **기본값이 true**라, ERROR 디스패치일 때는 필터의 본체 로직(토큰 검증)을 **건너뛴다**(그냥 통과). 그래서 `/error`에선 토큰 검증이 실행되지 않아 401 변환이 없다. (핵심은 필터가 어느 디스패치 타입에 등록되느냐가 아니라 이 스킵 메서드다 — 설령 필터가 ERROR 디스패치에서 실행되더라도 이 메서드가 본체 로직을 건너뛰게 하므로 안전하다.)
- 대조(중요한 실무 지식): **Spring Security**를 쓸 때는 이게 달랐다. Spring Security의 인가 필터(`AuthorizationFilter`, Security 5.7/6.0 기준)는 ERROR·FORWARD 디스패치에도 적용되기 때문에, 예외가 나 `/error`로 forward되면 그 요청이 인가를 다시 타서 막혔다. 그래서 예전엔 인가 설정에 `.requestMatchers("/error").permitAll()`을 넣어 `/error`를 열어줘야 했다. 우리는 Spring Security가 아니라 평범한 `OncePerRequestFilter`라, 그 스킵 메서드 덕분에 `/error`를 따로 허용할 필요가 없다.
- 추가로 MVC 계층도 안전하다: 기본 에러 엔드포인트를 처리하는 컨트롤러에는 우리 권한 애너테이션(`@RequireRole`)이 없으므로, 권한 인터셉터도 그냥 통과시킨다(no-op).

## 배운 것
- 리팩터할 때 "동작을 보존한다"는 프레임이 "이 동작이 애초에 옳은가?"라는 의심을 눌러버릴 수 있다. 옮겨오는(=이미 있던) 코드도 별도의 적대적 버그 헌팅 패스로 봐야 pre-existing 결함을 잡는다. 이번의 필터 예외 오처리와 JWT catch footgun 둘 다 그렇게 드러났다(둘 다 최초 구현부터 있던 결함이었다).
- 자동 코드 리뷰어는 서로 맹점이 달라 "여러 개를 겹쳐 쓰는 앙상블"이 가치다. 한 리뷰어가 "제일 똑똑"한 게 아니다 — 한 리뷰어는 필터의 doFilter 예외 오처리를 잡았지만 JWT catch footgun은 리뷰어 둘 다 놓쳐 사람+에이전트가 찾았다. LLM 리뷰는 확률적이라 같은 결함도 실행마다 잡히거나 놓칠 수 있다.
- 자체 인증을 서블릿 필터로 구현할 때 `OncePerRequestFilter`를 상속하면, 에러 디스패치 재실행(그로 인한 인증 재검증)이 기본으로 방어된다. Spring Security에서 `/error`를 permitAll 해야 했던 건 그쪽 인가 필터가 에러 디스패치에도 적용되기 때문 — 프레임워크가 다르면 이 부분 동작도 다르다.
