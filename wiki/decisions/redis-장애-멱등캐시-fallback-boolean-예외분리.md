---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, idempotency, redis, exception-handling, fallback, port-adapter]
created: 2026-06-01
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-01-pr-180-review-resolve-redis-fallback]]"
---

# Redis 장애 시 멱등 캐시를 DB unique 안전망으로 fallback — boolean을 도메인 예외로 분리

주문 생성 멱등 저장소(Redis)를 in-flight 차단 전용으로 좁힌 1차 작업([[주문-멱등-캐시-inflight-차단-전용]])의 후속으로, 저장소 접근 실패를 `reserve()`가 어떻게 표현할지를 다시 잡은 세션이다.

## 컨텍스트·결함 — boolean false가 정상 차단과 장애를 뭉개 주문이 마비된다

1차 작업에서 `RedisOrderIdempotencyStore.reserve()`(멱등 마커를 `setIfAbsent`으로 심어 in-flight 여부를 boolean으로 돌려주는 infra adapter)는 `DataAccessException`을 catch해 `return false`를 했다. 호출부 `OrderCreateService`는 `reserved == false`를 "다른 요청이 처리 중"으로 읽어 409 `ORDER_IDEMPOTENCY_IN_PROGRESS`를 던진다.

문제는 boolean `false`가 두 가지 전혀 다른 사실을 한 신호로 합친다는 것이다.

- *이미 다른 요청이 처리 중* — 정상적인 409 차단이어야 한다.
- *저장소 자체가 사용 불가* — Redis 장애로, 차단이 아니라 우회가 필요하다.

그래서 Redis가 죽으면 동시 요청이 하나도 없는 단독 주문까지 409로 막혀, 저장소가 내려간 동안 주문 생성 전체가 통째로 멎는다. 자동 코드 리뷰 두 곳이 이 결함을 독립적으로 지적했다.

## 'return true' 한 줄을 거부한 이유 — 증상 패치 vs 근본

두 자동 리뷰어가 제안한 수정은 모두 "`DataAccessException` catch에서 `return true`" 한 줄이었다. 이건 거부했다.

- `return true`는 장애 중 마커 없이 통과시켜 마비는 피한다. 하지만 이는 *예약 성공*으로 위장할 뿐, 여전히 정상/장애를 boolean 한 신호로 뭉갠다.
- 근본 원인(반환 시맨틱이 두 의미를 하나로 합친 것)은 그대로 남는다. **의미가 둘이면 예외로 분리하는 게 정직하다** — return type 자체가 하나의 추상화 선택이고, boolean이 *정상*과 *시스템 사정*을 동시에 표현하면 거짓말이 된다.

자동 리뷰가 *증상*은 정확히 짚었지만 *근본 원인*은 안 짚은 사례다. 리뷰 채택은 "제안 patch 그대로 받기"가 아니라 "지적의 본질에 정직한 해결"이라는 판단이 여기서 갈렸다. (같은 결의 AI 협업 판단은 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]] 참조.)

## 해결 — 반환 시맨틱을 도메인 예외로 분리, DB unique fallback

infra adapter가 `DataAccessException`을 도메인 예외 `OrderIdempotencyStoreUnavailableException`으로 변환해 던지고(이때 `log.error` + stack), `OrderCreateService`가 그 예외를 catch해 DB unique 안전망 경로로 fallback한다(`log.warn`).

- fallback 경로는 `findOrExecute`(사전 `find`로 멱등 흡수 후 없으면 insert, 충돌은 unique 제약이 최종 방어)로, 이는 [[find-first-write-not-check-db-unique-멱등]] 골격과 같다. Redis SETNX + RDB unique 이중 방어([[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]])에서 SETNX 층이 죽어도 RDB unique 층이 살아 있으니 fallback이 성립한다.
- fallback은 마커를 심지 않은 경로이므로 `clear`를 호출하지 않는다. 이로써 `reserve`의 boolean은 *진짜 예약 여부* 한 가지만 표현하게 된다.
- `clear()` 쪽의 `DataAccessException`은 `log.warn`만 하고 삼킨다 — 마커 잔존은 TTL(60초) 만료로 자가 회복한다.

커밋은 세 갈래로 나뉜다. (1) **fix** — 도메인 예외 신설 + service의 catch·fallback + 사전 find/insert를 `findOrExecute` private 헬퍼로 추출. (2) **docs** — 예외 처리 전략 문서에 *Redis 캐시 장애 처리* 섹션 신설 등 문서 동기화. (3) **test** — fallback 경로에서 `clear`가 절대 호출되지 않음을 `verify(..., never()).clear(...)`로 결박.

## infra 매핑 vs application catch — 분기점은 fallback 가능성

infra adapter가 기술적 예외를 도메인 예외로 *매핑*하는 단계는 어느 도메인이든 공통이다. 갈리는 건 변환 이후 *정책 결정 위치*이고, 그 갈림의 기준은 **fallback 가능 여부**다.

- **Order**는 DB unique 제약이라는 안전망이 있어 fallback 가능 → application이 도메인 예외를 catch해 안전망 경로로 진입한다. infra는 "어떤 예외냐"만 알고, "어떻게 대응하냐"는 application이 쥐어 port 추상화가 강화된다.
- **Auth**(인증 토큰 저장소)는 fallback 불가 — refresh token이 없으면 인증 자체가 불가능해 대체 경로가 없다.

이 차이를 *현재 코드 패턴 차이*가 아니라 *fallback 가능성 차이*로 설명할 수 있어야 정합이 선다. 같은 매핑 구조를 쓰면서 정책 위치만 fallback 가능성에 따라 갈라지는 게 규약의 본체다. (Auth 쪽 저장소 예외 매핑은 [[auth-redis-장애-도메인예외-매핑-도메인전용-어드바이스]]에서 별도로 정리된다.) Redis 장애에 대해 무조건 차단하는 strict 정책을 기각한 배경은 [[redis-장애-strict-정책-soft-fail-기각]]와 같은 결이다.

## RuntimeException 직접 상속 — 자동 매핑 회피로 catch 강제 (낙관 락 예외와 반대)

`OrderIdempotencyStoreUnavailableException`을 프로젝트 공용 `CustomException` 대신 `RuntimeException`을 직접 상속시켰다.

- `CustomException`을 상속하면 `GlobalExceptionHandler.handleCustomException`이 이 예외를 자동으로 응답에 매핑해버려, "application이 catch해 fallback한다"는 의도가 우회된다. 자동 매핑을 피해야 catch 흐름이 강제된다.
- Spring `@ExceptionHandler` 자동 매핑은 *편의*와 *catch 우회 위험*의 양면이다. Redis 장애 예외는 자동 매핑을 **회피**하는 쪽을 택했다.

> [!note] 낙관 락 충돌 예외와 정반대
> 낙관 락 충돌 예외는 오히려 409 자동 매핑을 *원해서* `CustomException` 계열에 둔다([[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]). "도메인 예외는 RuntimeException 직접 상속"은 일반 규칙이 아니라, 이번처럼 fallback/우회 회피가 필요한 경우 한정이다.

port 시그니처에 Spring 예외를 노출하지 않는 원칙(추상 수준 경계)은 [[persistence-exception-노출-경계-추상수준]], 도메인 예외에서 HttpStatus를 떼는 방향은 [[도메인-예외-httpstatus-제거-errorcategory]]와 같은 클러스터다.

## 예외 클래스 위치 — 도메인별 카탈로그 컨벤션 유지

예외를 `com.commerce.<domain>.exception/`에 둔다. 이 프로젝트는 *도메인별 cross-cutting 예외 카탈로그* 컨벤션을 쓰고, `application/exception/` 같은 layer별 분리는 채택하지 않는다. 예외 하나 추가하려고 새 배치 패턴을 들이지 않는다.

## 지금 다시 본다면

- 1차 작업(캐시 책임을 in-flight 차단으로 단순화)에 집중하느라 *실패 표현 시맨틱*을 따로 챙겨야 한다는 점을 놓쳤다. *단일 boolean 시맨틱이 OK*라고 본 게 가장 큰 누락이었다.
- **교훈**: return type의 시맨틱이 *정상*과 *시스템 사정*을 동시에 표현하면 거짓말이 된다. port/adapter에서 fallback 정책의 결정 주체는 application이고, infra는 *기술적 사실*만 알면 된다. 이걸 *port 시그니처에 Spring 예외 노출 없음*으로 강제하면 추상화가 보존된다.

## 미해결·후속

- **Auth 패턴 통일 (#181)** — Auth 저장소 실패는 여전히 application이 직접 catch하는 패턴이라, 이번 Order의 "infra 매핑 + 정책은 application" 규약과 어긋나 일관성 부담이 남는다. 후속 별도 이슈로 분리했다: Auth도 infra 매핑으로 옮겨 *동작은 보존하고 port 추상화만 강화*한다. (Auth는 fallback 불가라 최종 정책 위치는 Order와 다를 수 있다.)
- **TTL 초과 시 finally clear가 남의 marker를 지우는 race** — 자동 리뷰가 추가로 지적했으나 이번엔 무수정으로 뒀다. `clear`가 key 기반 *무조건 delete*라면, 자기 트랜잭션이 마커 TTL(60초)을 넘겨 늦게 끝날 때 그 사이 만료·재생성된 *다른 요청의 marker*를 삭제할 수 있다. 근본 해결은 owner-token + compare-delete(Lua로 원자적 "내 토큰일 때만 삭제")다. 다만 트리거 조건(결제 PG 호출 같은 외부 I/O가 트랜잭션 안으로 들어와 60초를 넘길 latency)이 현재 코드엔 없고, TTL 60초 결정이 이미 이 trade-off를 인지·수용한 상태라 현 구조를 받아들인다. 외부 I/O가 트랜잭션에 진입하면 재검토가 트리거다.
- **`@MockitoSpyBean`은 테스트 종료 시 mock을 자동 reset**한다(Spring Boot 3.4+에서 `@SpyBean`을 대체, `MockReset.AFTER` 기본). 그래서 테스트 간 stub 누수가 없어 mock을 되돌리는 별도 `@AfterEach`가 필요 없다(현 테스트의 `@AfterEach`는 DB 정리만 한다). 다른 agent가 보수적으로 경고했지만, 프레임워크 공식 기본 동작을 확인하는 쪽이 정확했다.

## 근거

- [[raw/sessions/backend/2026-06-01-pr-180-review-resolve-redis-fallback]]
