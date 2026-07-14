---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, idempotency, redis, cache, concurrency, transaction-boundary]
created: 2026-06-01
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-01-pr-180-idempotency-cache-simplification]]"
---

# 주문 생성 멱등 캐시를 in-flight 차단 전용으로 좁힘 — 결과 캐싱 제거

## 컨텍스트 — 세 상태 캐시 재검토

주문 생성 멱등성은 그동안 Redis가 `PROCESSING`·`COMPLETED`·`FAILED` 세 상태를 관리하며 "1차 방어선 + 결과 캐시" 역할을 하고, RDB unique 제약이 최종 보장을 맡는 이중 구조였다. 출발점은 두 이슈였다 — 같은 멱등 키로 동시에 들어온 요청이 안전망 500을 받아 클라이언트가 *처리 중인지 진짜 장애인지* 구분 못 하는 문제(#171), 그리고 `FAILED` enum이 어디서도 set되지 않는 placeholder로만 남은 문제(#172). 두 문제를 따로 패치하는 대신 "Redis가 세 상태를 관리한다"는 가정 자체를 재검토했다(PR #180, #171/#172/#173 동시 close).

## 핵심 통찰 — 캐시 본질은 중복 차단, 결과 캐싱은 부수효과

**"이 캐시의 본질은 중복 차단이고, 결과 캐싱은 부수 효과였다."** 이걸 잡으면 나머지 결정이 줄줄이 따라온다.

## 결과 캐싱 제거 — 효용 ms 미만, 인터페이스 4→2, enum 삭제

주문 멱등 저장소(Redis 마커 port)의 책임을 *처리 진행 중 표시* 한 가지로 좁히고 `COMPLETED`/`FAILED` 결과 캐싱을 제거했다. 결과의 진실은 DB 하나만 갖는다.

- **결과 캐싱 효용이 실측상 무의미.** `COMPLETED` 캐시가 hit해도 결국 주문 조회(`findById`)로 DB를 한 번 더 친다. unique 인덱스 조회(`findByMemberIdAndIdempotencyKey`)와의 latency 차이가 ms 미만이라 캐시가 *DB 조회 자체*를 줄여주지 못한다. 반면 결과 캐싱은 캐시-DB 정합성 위험(마커 정리 실패 시 불일치)만 얹는다.
- **`FAILED` 캐싱도 불필요.** 재시도는 같은 비즈니스 검증을 DB에서 다시 거쳐 같은 실패를 얻고, 일시 실패는 retry로 회복된다. 실패를 캐싱하면 오히려 재시도 의미가 사라진다.
- **결과:** 인터페이스가 4개 메서드에서 2개(`reserve`, `clear`)로 줄고, 상태 enum(`OrderIdempotencyStatus`)은 파일째 제거. 마커 의미가 *처리 중* 한 종류뿐이라 더 표현할 게 없다. 정상 완료된 키가 즉시 사라져 Redis 메모리도 TTL 만료까지 붙들지 않는다.

## sealed interface 거부 (YAGNI)

멱등 상태를 타입 안전하게 표현하려 sealed interface 도입을 검토했으나 기각. enum + nullable field로 충분한 도메인에 sealed를 얹는 건 과한 추상화다 — YAGNI는 인터페이스 설계에도 적용된다. 마커가 *처리 중* 한 의미만 갖게 되며 타입 분기 자체가 불필요해졌고 enum도 제거됐다. 이 판단("단순 도메인에 sealed 도입 금지")은 재사용 가능한 설계 피드백으로 따로 기록했다.

## listener 제거 + finally clear (AFTER_COMMIT은 기본 동기, latency 격리 0)

기존엔 주문 생성 흐름이 이벤트를 발행하고 Redis 저장소가 `@TransactionalEventListener(AFTER_COMMIT)`으로 받아 마커를 정리했다. 이 구조를 걷어내고 `OrderCreateService`가 `try-finally`에서 `clear()`를 직접 호출하도록 바꿨다.

- **결정 분기점:** `@TransactionalEventListener(AFTER_COMMIT)`은 **기본이 동기 실행**이다. "이벤트로 분리했으니 비동기로 격리됐다"는 착각이고 *latency 격리 효과는 0*이다. 이벤트 클래스·publish 호출·MDC 전파 부가 비용만 떠안고 얻는 게 없다.
- **commit 이후 보장은 이미 공짜.** 이 서비스가 트랜잭션을 열지 않는(`NOT_SUPPORTED`) 구성이라 finally의 `clear`가 자동으로 커밋 이후에 호출된다. "Redis 호출은 RDB 커밋 이후" 원칙이 listener 없이 자연히 지켜진다.
- **검토한 대안(마커 정리 위치):** (A) catch 블록 clear — 비즈니스 예외(4xx)만 정리되고 RuntimeException(5xx)에서 누락. (B) finally + success flag — 성공 시에만 정리, 불필요한 복잡도. (C, 채택) finally 무조건 clear — 성공/실패 무관 즉시 정리, 재시도 가능, 가장 단순·일관. 비정상 잔존(서버 crash)은 TTL 만료로 자가 회복.
- publisher 패턴은 *진짜 비동기 분리*나 *다중 후처리*가 필요할 때 값을 갖는데 이번 책임은 "마커 정리 한 줄"이라 둘 다 아니다. listener가 사라지며 #173(AFTER_COMMIT listener 비동기 전환)은 자동 close.

## 사전 DB find를 reserve 뒤 배치 (DB 도달 전 차단)

정상 멱등 흡수를 담당하는 사전 DB find를 Redis `reserve()` **뒤**에 뒀다. `reserve`가 false(다른 요청이 이미 처리 중)면 DB find 자체가 발생하지 않는다 — 캐시의 *DB 도달 전 차단* 가치가 이 배치에서 나온다. 앞에 두면 동시 요청이 모두 DB find를 한 번씩 통과한 뒤 race window에 진입해 매 요청 DB 조회가 발생하고 캐시의 명분이 약해진다.

## 동시 요청 500→409 IN_PROGRESS

같은 키 동시 요청의 응답을 안전망 500에서 명시적 409 `ORDER_IDEMPOTENCY_IN_PROGRESS`로 바꿨다. 사용자가 *처리 중*임을 인지하고 backoff 재시도할 수 있게 된다. 기존 find-first 패턴(정상 흐름은 사전 find로 흡수하고 실제 동시 충돌만 안전망 500에 위임)과의 정합은 유지된다 — 안전망 500 위임은 사라지지 않고 *Redis fallback 후 도달하는 진짜 race* 한 곳으로만 좁아진다. 그 좁아진 한 곳의 fallback 정책은 [[redis-장애-멱등캐시-fallback-boolean-예외분리]]가 이어받는다(이 노트가 좁힌 boolean `reserve`를 그 노트가 장애 시 예외로 분리). find-first + unique 최종보장은 [[find-first-write-not-check-db-unique-멱등]].

## PROCESSING TTL 600→60초 (산정 기준)

마커 TTL을 60초로 낮췄다. 600초는 마커가 결과 캐시(COMPLETED)까지 겸하던 시절의 값이라 의미가 *처리 진행 중* 한 가지로 좁아진 지금은 과하다.

- **산정 기준:** 주문 생성 p99 latency ~100ms 추정이라 60초는 600배 마진이고, MySQL 락 대기 만료값(`innodb_lock_wait_timeout` 기본 50초)보다 살짝 길어 락 대기가 만료되면 마커 TTL도 곧 만료돼 자가 회복한다.
- **얻는 것:** 서버 crash 같은 비정상 잔존 시 사용자 봉인 시간이 짧아진다(클라이언트 backoff 한두 사이클). 정상 처리는 ms 단위라 TTL 만료로 인한 race는 사실상 없다.

## 받아들인 한계 — DB 상태 기준 멱등성, Redis timeout 동기 직결

- 같은 멱등 키를 재시도하는 시점에 DB 상태가 바뀌었으면 *다른 응답*이 나올 수 있다(예: 첫 시도 `PRODUCT_NOT_FOUND` → 그 사이 상품 등록 → 재시도 200). *완벽한 응답 일관성* 대신 *DB 상태 기준 멱등성*을 택했다.
- Redis timeout(수 ms ~ 수 초) 시 응답 latency에 그대로 영향. 동기 finally가 Redis latency를 응답에 직결시키는 구조 — 비동기 listener로 분리 가능하나 이번 범위 밖.

## 곁가지 — NaverPay unique 누락이 진행을 막음(별도 이슈 우회)

작업 첫 단계의 통합 검증(`dockerTest`)이 무관한 NaverPay 도메인의 기존 동시성 결함까지 잡으며 blocked됐다. 원인은 `tbl_payment_attempt`의 multi-column unique 제약이 실제 DB에 안 걸리던 것 — 대상 컬럼이 `VARCHAR(255)` 기본으로 생성돼 utf8mb4에서 InnoDB unique key 바이트 한도를 넘겨 MySQL이 unique 생성을 거부했고, Hibernate가 silent로 넘겨 제약 없이 운영됐다([[silent-schema-drift-패턴]]). 별도 이슈(#176)로 분리해 PR #179로 먼저 해결(`@Column(length=...)` 명시로 unique 복구, [[multi-column-unique-length-명시-컨벤션]])한 뒤 rebase 재개. 교훈: "develop HEAD에서 같은 실패가 재현되는가"를 먼저 확인해 진짜 이 작업의 책임인지 빠르게 분리해야 한다.

## 미해결 — 재검토 트리거들

이 task 자체의 후속은 없다. 의도적으로 미룬 결정의 재검토 트리거만 남겼다.

- **Redis timeout이 잦아지면** → 비동기 listener 재도입 검토(지금은 동기 finally가 Redis latency를 응답에 직결).
- **외부 시스템 후처리(알림·정산)가 필요해지면** → outbox 패턴 적용.
- **도메인 이벤트 다중 후처리가 확산되면** → Spring Event 부활 검토(*단일 책임 후처리*가 아니라 *다중 후처리*가 되는 시점에 publisher 패턴이 값을 갖는다).
- **결제 PG 통합으로 트랜잭션 안에 외부 I/O가 들어오면** → PROCESSING TTL 재검토.

## 배운 것

- **`@TransactionalEventListener(AFTER_COMMIT)`은 기본 동기 실행이다.** 진짜 비동기는 `@Async`나 별도 `TaskExecutor`를 명시해야 한다. publisher 패턴의 가치는 *비동기 분리* 또는 *다중 후처리*에 있지 단순 코드 분리에 있지 않다.
- **캐시 책임을 명확히 정의하면 인터페이스가 단순해진다.** 부수효과(결과 캐싱)까지 끌어안으면 정합성 책임이 는다. 기능을 얹기 전에 "이 캐시가 진짜 주는 가치가 뭐냐"를 한 번 더 물어야 한다.
- **AC 자동 검증에 negative assertion을 넣는 건 위험하다.** `grep`의 exit code(매칭 있음=0/없음=1)와 실행기의 *exit 0=통과* 가정이 충돌한다 — 이 계열 교훈은 [[동시성-테스트-작성-규칙과-단언-전략]]과도 이어진다.

## 근거

- [[raw/sessions/backend/2026-06-01-pr-180-idempotency-cache-simplification]] — 멱등 캐시 재설계 전체(결과 캐싱 제거, sealed 거부, listener 제거, TTL, NaverPay 우회, 회고 트리거)(PR #180).
