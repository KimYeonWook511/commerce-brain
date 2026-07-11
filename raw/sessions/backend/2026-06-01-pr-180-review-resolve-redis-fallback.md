---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 180 }
---

# Redis 장애 시 주문 멱등성 캐시를 DB unique 안전망으로 fallback — boolean 반환을 도메인 예외로 분리

주문 생성 멱등 저장소(Redis)를 in-flight 차단 전용으로 좁힌 1차 작업에서, 저장소 접근 실패
(`DataAccessException`)를 catch해 `reserve()`가 `false`를 반환하도록 짰다. 그런데 `false`는
*이미 다른 요청이 처리 중*(정상 409 차단)과 *저장소 자체가 사용 불가*(시스템 장애)를 한 신호로
합친다 — 그래서 Redis가 죽으면 동시 요청이 하나도 없는 단독 주문까지 409로 막혀 주문 생성 전체가
마비된다. 자동 코드 리뷰 두 곳이 이 결함을 독립적으로 지적했고, 그들이 제안한 "한 줄 `return true`"
패치를 받지 않고 *반환 시맨틱 자체를 도메인 예외로 분리*하는 방향으로 다시 잡은 세션이다.

## 막힌 점·해결

### Redis 장애가 정상 차단으로 위장돼 주문 생성이 마비된다

- **결함:** `RedisOrderIdempotencyStore.reserve()`(멱등 마커를 `setIfAbsent`으로 심어 in-flight
  여부를 boolean으로 돌려주는 infra adapter)의 `DataAccessException` catch가 `return false`였다.
  호출부 `OrderCreateService`는 `reserved == false`를 "다른 요청이 처리 중"으로 읽어 409
  `ORDER_IDEMPOTENCY_IN_PROGRESS`를 던진다. 결국 Redis 장애 시 *경합이 전혀 없는 단독 주문도* 409로
  차단돼, 저장소가 내려간 동안 주문 생성이 통째로 멎는다.
- **두 자동 리뷰어가 같은 결함을 독립적으로 지적**했고, 둘 다 제안한 수정은 "`return true`" 한 줄이었다.
  이건 거부했다. `return true`는 장애 중 마커 없이 통과시켜 마비는 피하지만, *예약 성공*으로 위장할 뿐
  여전히 정상/장애를 boolean 한 신호로 뭉갠다 — 근본(시맨틱 합쳐짐)은 그대로다.
- **해결:** infra adapter가 `DataAccessException`을 도메인 예외
  `OrderIdempotencyStoreUnavailableException`으로 변환해 던지고(이때 `log.error` + stack),
  `OrderCreateService`가 그 예외를 catch해 DB unique 안전망 경로(`findOrExecute` — 사전 `find`로 멱등
  흡수 후 없으면 insert, 충돌은 unique 제약이 최종 방어)로 fallback 진행한다(`log.warn`). fallback은
  마커를 심지 않은 경로이므로 `clear`를 호출하지 않는다. 이로써 `reserve`의 boolean은 *진짜 예약
  여부* 한 가지만 표현하게 된다. (`clear()` 쪽의 `DataAccessException`은 `log.warn`만 하고 삼킨다 —
  마커 잔존은 TTL 만료로 자가 회복.)
- **추가 커밋 세 갈래의 성격:** (1) fix — 도메인 예외 신설 + service의 catch·fallback + 사전
  find/insert를 `findOrExecute` private 헬퍼로 추출, (2) docs — 예외 처리 전략 문서에 *Redis 캐시
  장애 처리* 섹션을 신설하는 등 관련 문서 동기화, (3) test — fallback 경로에서 `clear`가 **절대**
  호출되지 않음을 `verify(..., never()).clear(...)`로 결박.

## 결정한 것

### 1. 한 줄 `return true` 대신 반환 시맨틱을 도메인 예외로 분리한다

- **boolean 시맨틱 정직성이 핵심 통찰.** *예약됨*과 *저장소 사용 불가*를 boolean `false` 한 신호로
  합치면 application이 *정상 차단(409)*과 *시스템 사정(fallback)*을 구별할 수 없다. 의미가 두 가지면
  *예외로 분리*가 정직하다. return type 자체가 하나의 추상화 선택이다.

### 2. infra 매핑 vs application 직접 catch — 분기점은 fallback 가능성

- infra adapter가 기술적 예외를 도메인 예외로 *매핑*하는 단계는 어느 도메인이든 공통이다. 갈리는 건
  변환 이후 *정책 결정 위치*이고, 그 갈림의 기준은 **fallback 가능 여부**다.
- **Order는 DB unique 제약이라는 안전망이 있어 fallback 가능** → application이 도메인 예외를 catch해
  안전망 경로로 진입. infra는 "어떤 예외냐"만 알고, "어떻게 대응하냐"는 application이 쥐어 port 추상화가
  강화된다.
- **Auth(인증 토큰 저장소)는 fallback 불가** — refresh token이 없으면 인증 자체가 불가능해 대체 경로가
  없다. (이 프로젝트 Auth는 당시 application이 저장소 예외를 직접 catch하는 패턴이라 방식이 갈려 일관성
  부담이 있었다. 후속 별도 작업으로 남김 — 미해결 참조.)
- **이 차이를 *현재 코드 패턴 차이*가 아니라 *fallback 가능성 차이*로 설명할 수 있어야 정합이 선다.**
  같은 매핑 구조를 쓰면서 정책 위치만 fallback 가능성에 따라 갈라지는 게 규약의 본체다.

### 3. 도메인 예외는 `RuntimeException` 직접 상속 — catch 흐름을 강제하려는 의도

- `OrderIdempotencyStoreUnavailableException`을 프로젝트 공용 `CustomException` 대신 `RuntimeException`을
  직접 상속시켰다. `CustomException`을 상속하면 `GlobalExceptionHandler.handleCustomException`가 이
  예외를 자동으로 응답에 매핑해버려, "application이 catch해 fallback한다"는 의도가 우회된다. 자동 매핑을
  피해야 catch 흐름이 강제된다.
- Spring `@ExceptionHandler` 자동 매핑은 *편의*와 *catch 우회 위험*의 양면이다. Redis 장애 예외는
  자동 매핑을 **회피**하는 쪽을 택한 것 — (참고로 낙관 락 충돌 예외는 정반대로, 409 자동 매핑을 *원해서*
  `CustomException` 계열로 둔다. "도메인 예외는 RuntimeException 직접 상속"은 일반 규칙이 아니라 이런
  fallback/우회 회피가 필요한 경우 한정이다.)

### 4. 예외 클래스 위치는 기존 컨벤션을 따른다 — 단일 예외 하나로 새 패턴 도입 안 함

- 예외를 `com.commerce.<domain>.exception/`에 둔다 — 이 프로젝트는 *도메인별 cross-cutting 예외
  카탈로그* 컨벤션을 쓰고, `application/exception/` 같은 layer별 분리는 채택하지 않는다. 예외 하나
  추가하려고 새 배치 패턴을 들이지 않는다.

**지금 다시 본다면**

- 1차 작업(캐시 책임을 in-flight 차단으로 단순화)에 집중하느라 *실패 표현 시맨틱*을 따로 챙겨야 한다는
  점을 놓쳤다. *단일 boolean 시맨틱이 OK*라고 본 게 가장 큰 누락이었다.
- 자동 코드 리뷰가 *증상*만 지적하고 *근본 원인(시맨틱이 합쳐진 것)*은 안 짚었지만, 사용자와 대화하며
  *더 정직한 방향*으로 다시 잡을 수 있었던 흐름이 가치 있었다. *제안 patch 그대로 받기 vs 근본까지
  파고들기*의 판단이 리뷰 수용의 본질이다.

## 배운 것

- **return type의 시맨틱이 *정상*과 *시스템 사정*을 동시에 표현하면 거짓말이 된다.** 의미가 둘이면
  *예외로 분리* — 시그니처가 정직해지고 caller가 분기 의도를 *코드 흐름으로* 표현할 수 있다.
- **port/adapter 패턴에서 fallback 정책의 결정 주체는 application이다.** infra는 *기술적 사실*(어떤
  예외인지)만 알면 되고, *어떻게 대응할지*는 application이 정한다. 이걸 *port 시그니처에 Spring 예외
  노출 없음*으로 강제하면 추상화가 보존된다.
- **TTL race + finally clear의 메커니즘.** `clear`가 key 기반 *무조건 delete*라면, 자기 트랜잭션이
  마커 TTL(60초)을 넘겨 늦게 끝날 경우 그 사이 만료·재생성된 *다른 요청의 marker*를 삭제할 수 있다.
  근본 해결책은 owner-token + compare-delete(Lua로 원자적 "내 토큰일 때만 삭제")다. 다만 *트리거
  조건(트랜잭션이 TTL을 넘길 만한 latency)*이 없으면 *메커니즘 명시 + 해결책 사전 파악*까지만 해두고
  현 구조를 받아들인다 — *낮은 빈도 race*의 대응 패턴이다.
- **자동 코드 리뷰의 한계와 가치.** *증상은 정확*하지만 *해결책은 단순*(`return true` 한 줄)했다.
  *근본 원인까지 파고들어 더 나은 방향*으로 가는 판단은 사람 몫이다. 리뷰 채택은 *제안 patch 그대로
  받기*가 아니라 *지적의 본질에 정직한 해결*이다.
- **`@MockitoSpyBean`은 테스트 종료 시 mock을 자동 reset한다**(Spring Boot 3.4+에서 `@SpyBean`을
  대체, `MockReset.AFTER` 기본). 그래서 테스트 간 stub 누수가 없어 mock을 되돌리는 별도 `@AfterEach`가
  필요 없다(현 테스트의 `@AfterEach`는 DB 정리만 한다). 다른 agent가 보수적으로 *명시 안 됨*이라
  경고했지만, 프레임워크 공식 기본 동작을 확인하는 쪽이 정확했다.

## 미해결·열린 질문

- **Auth 패턴 통일** — Auth 저장소 실패는 여전히 application이 직접 catch하는 패턴이라, 이번에 정리한
  Order의 "infra 매핑 + 정책은 application/presentation" 규약과 어긋나 일관성 부담이 남는다. 후속 별도
  이슈(#181)로 분리했다: Auth도 infra 매핑으로 옮겨 *동작은 보존하고 port 추상화만 강화*한다. (Auth는
  fallback 불가라 최종 정책 위치는 Order와 다를 수 있다.)
- **TTL 초과 시 finally clear가 남의 marker를 지우는 race** — 자동 리뷰가 이 지점을 추가로 지적했으나
  이번엔 무수정으로 뒀다. 트리거 조건(결제 PG 호출 같은 외부 I/O가 트랜잭션 안으로 들어와 60초를 넘길
  latency)이 현재 코드엔 없고, TTL을 60초로 정한 결정이 이미 이 trade-off를 인지·수용한 상태이기
  때문이다. 확인 결과 결제 PG 통합은 지금 계획이 없어 owner-token(compare-delete) 도입도 불필요하다.
  미래에 그런 외부 I/O가 트랜잭션에 진입하면 TTL과 함께 이 race를 재검토하는 게 트리거다.
