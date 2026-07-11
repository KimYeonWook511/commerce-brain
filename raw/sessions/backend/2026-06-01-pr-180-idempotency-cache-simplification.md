---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 180 }
---

# 주문 생성 멱등 캐시를 in-flight 차단 전용으로 좁힘 — 결과 캐싱 제거, 동시 요청 500→409

주문 생성 멱등성은 그동안 Redis 가 `PROCESSING`·`COMPLETED`·`FAILED` 세 상태를 관리하며
"1차 방어선 + 결과 캐시" 역할을 하고, RDB unique 제약이 최종 보장을 맡는 이중 구조였다.
이 세션은 그 구조를 다시 뜯어본 것이다. 출발점은 두 이슈였다 — 같은 멱등 키로 동시에 들어온
요청이 안전망 500 을 받아 클라이언트가 *처리 중인지 진짜 장애인지* 구분 못 하는 문제(`#171`),
그리고 `FAILED` enum 값이 어디서도 set 되지 않는 placeholder 로만 남아 있던 문제(`#172`).
두 문제를 따로 패치하는 대신 "Redis 가 세 상태를 관리한다"는 가정 자체를 재검토했고,
결과 캐싱의 효용이 DB 조회 대비 ms 미만이라는 사실이 드러나면서 캐시 책임을 한 가지로 좁히는
전면 재설계로 이어졌다. 결과물은 PR #180 (`#171`/`#172`/`#173` 동시 close).

## 결정한 것

핵심 통찰은 **"이 캐시의 본질은 중복 차단이고, 결과 캐싱은 부수 효과였다"**. 이걸 잡으면
나머지 결정이 줄줄이 따라온다.

### 캐시 책임을 in-flight 차단 전용으로 좁힘

주문 멱등 저장소(Redis 마커를 다루는 port)의 책임을 *처리 진행 중 표시* 한 가지로 좁히고,
`COMPLETED`/`FAILED` 결과 캐싱을 제거했다. 결과의 진실은 DB 하나만 갖는다.

- **결과 캐싱 효용이 실측상 무의미했다.** `COMPLETED` 캐시가 hit 해도 결국 주문 조회(`findById`)로
  DB 를 한 번 더 친다. unique 인덱스 조회(`findByMemberIdAndIdempotencyKey`)와의 latency 차이가
  ms 미만이라 캐시가 *DB 조회 자체* 를 줄여주지 못한다. 반면 결과 캐싱은 캐시-DB 정합성 위험
  (마커 정리 실패 시 불일치)만 얹는다.
- **`FAILED` 캐싱도 불필요.** 재시도는 같은 비즈니스 검증을 DB 에서 다시 거쳐 같은 실패를 얻고,
  일시 실패(네트워크 등)는 retry 로 회복된다. 실패를 캐싱하면 오히려 재시도 의미가 사라진다.
- **결과:** 멱등 저장소 인터페이스가 4개 메서드에서 2개(`reserve`, `clear`)로 줄고, 마커 상태를
  표현하던 enum(`OrderIdempotencyStatus`)은 파일째 제거됐다 — 마커 의미가 *처리 중* 한 종류뿐이라
  더 표현할 게 없다. 정상 완료된 키가 즉시 사라져 Redis 메모리도 TTL 만료까지 붙들지 않는다.

### sealed interface 거부

멱등 상태를 타입 안전하게 표현하려 sealed interface 도입을 검토했으나 기각했다.
enum + nullable field 로 충분한 도메인에 sealed 를 얹는 건 과한 추상화다 — YAGNI 는 인터페이스
설계에도 그대로 적용된다. 게다가 마커가 *처리 중* 한 의미만 갖게 되면서 타입 분기 자체가
불필요해졌고, 결국 enum 도 제거하는 방향으로 정리됐다. 이 판단("단순한 도메인에 sealed 도입
금지")은 재사용 가능한 설계 피드백으로 따로 기록해뒀다.

### listener 제거 + Service finally 에서 clear 직접 호출

기존엔 주문 생성 흐름이 이벤트를 발행하고(`applicationEventPublisher`) Redis 저장소가
`@TransactionalEventListener(AFTER_COMMIT)` 으로 받아 마커를 정리했다. 이 구조를 걷어내고
주문 생성 응용 서비스(`OrderCreateService`)가 `try-finally` 안에서 `clear()` 를 직접 호출하도록
바꿨다.

- **결정 분기점:** `@TransactionalEventListener(AFTER_COMMIT)` 은 **기본이 동기 실행**이다.
  "이벤트로 분리했으니 비동기로 격리됐다"는 건 착각이고 실제 *latency 격리 효과는 0* 이다.
  그러면 이벤트 클래스·publish 호출·MDC 전파 같은 부가 비용만 떠안고 얻는 게 없다.
- **commit 이후 보장은 이미 공짜였다.** 이 서비스가 트랜잭션을 열지 않는(`NOT_SUPPORTED`) 구성이라
  finally 의 `clear` 가 자동으로 커밋 이후에 호출된다. "Redis 호출은 RDB 커밋 이후" 원칙이
  listener 없이도 자연히 지켜진다.
- **검토한 대안(마커 정리 위치):** (A) catch 블록에서 clear — 비즈니스 예외(4xx)만 정리되고
  RuntimeException(5xx)에서 누락. (B) finally + success flag — 성공 시에만 정리, 실패 시 TTL 만료
  대기, 불필요한 복잡도. (C, 채택) finally 무조건 clear — 성공/실패 무관 즉시 정리, 재시도 가능,
  가장 단순·일관. 비정상 잔존(서버 crash)은 TTL 만료로 자가 회복.
- publisher 패턴이 값을 갖는 건 *진짜 비동기 분리* 나 *다중 후처리* 가 필요할 때인데, 이번 책임은
  "마커 정리 한 줄"이라 둘 다 아니다. listener 자체가 사라지면서 `#173`(AFTER_COMMIT listener
  비동기 전환)은 자동으로 close 됐다.

### 사전 DB find 를 reserve 뒤에 배치

정상 멱등 흡수를 담당하는 사전 DB find 를 Redis `reserve()` **뒤**에 뒀다.
`reserve` 가 false(다른 요청이 이미 처리 중)면 DB find 자체가 발생하지 않는다 — 캐시의
*DB 도달 전 차단* 가치가 이 배치에서 나온다. 앞에 두면 동시 요청이 모두 DB find 를 한 번씩
통과한 뒤 race window 에 진입해, 매 요청 DB 조회가 발생하고 캐시의 명분이 약해진다.

### 동시 요청 응답을 500 → 409 `ORDER_IDEMPOTENCY_IN_PROGRESS` 로

같은 키 동시 요청의 응답을 안전망 500 에서 명시적 409 `ORDER_IDEMPOTENCY_IN_PROGRESS` 로 바꿨다.
사용자 입장에서 *처리 중* 임을 인지하고 backoff 재시도할 수 있게 된다. 기존 정책(정상 흐름은 사전
find 로 흡수하고 실제 동시 충돌만 안전망 500 에 위임하는 find-first 패턴)과의 정합은 유지된다 —
안전망 500 위임은 사라지지 않고, *Redis fallback 후 도달하는 진짜 race* 한 곳으로만 좁아진다.

### PROCESSING TTL 을 600초 → 60초로

마커 TTL 을 60초로 낮췄다. 600초는 마커가 결과 캐시(COMPLETED)까지 겸하던 시절의 값이라,
의미가 *처리 진행 중* 한 가지로 좁아진 지금은 과하다.

- **산정 기준:** 주문 생성 p99 latency 는 ~100ms 추정이라 60초는 600배 마진이고, MySQL 의
  락 대기 만료값(`innodb_lock_wait_timeout` 기본 50초)보다 살짝 길어 락 대기가 만료되면
  마커 TTL 도 곧 만료돼 자가 회복한다. 이 기준은 이 결정의 ADR 본문에 표로 남겨뒀다.
- **얻는 것:** 서버 crash 같은 비정상 잔존 시 사용자 봉인 시간이 짧아진다(클라이언트 backoff
  한두 사이클). 정상 처리는 ms 단위라 TTL 만료로 인한 race 는 사실상 발생하지 않는다.

### 받아들인 한계(의도된 트레이드오프)

- 같은 멱등 키를 재시도하는 시점에 DB 상태가 바뀌었으면 *다른 응답* 이 나올 수 있다
  (예: 첫 시도 `PRODUCT_NOT_FOUND` → 그 사이 상품 등록 → 재시도 200). *완벽한 응답 일관성*
  대신 *DB 상태 기준 멱등성* 을 택했다.
- Redis timeout(수 ms ~ 수 초) 시 응답 latency 에 그대로 영향이 간다. 동기 finally 가 Redis
  latency 를 응답에 직결시키는 구조이기 때문. 비동기 listener 로 분리 가능하지만 이번 범위 밖 —
  timeout 빈도가 실제 문제로 드러나면 재검토.
- `#173`(listener 비동기 전환)은 자동 close — 전환할 listener 자체가 사라졌다.

## 막힌 점·해결

### 무관한 NaverPay 도메인의 동시성 결함이 진행을 막음 — 분리 등록으로 우회

작업 첫 단계의 통합 검증(`./gradlew dockerTest`)이 이 task 와 무관한 NaverPay 도메인의 기존
동시성 결함까지 잡으면서 본 단계가 blocked 됐다. 원인은 결제 시도 테이블(`tbl_payment_attempt`)의
multi-column unique 제약이 실제 DB 에는 걸리지 않던 것 — 대상 컬럼들이 `VARCHAR(255)` 기본값으로
생성돼 utf8mb4 환경에서 InnoDB unique key 바이트 한도를 넘겼고, MySQL 이 unique key 생성을 거부한
걸 Hibernate 가 silent 로 넘겨 제약 없이 운영되고 있었다. 그래서 동시성 테스트가 중복 결제 시도
race 를 드러냈다. 이 결함을 별도 이슈(`#176`)로 분리 등록하고 별도 PR #179 로 먼저 해결(대상 컬럼에
`@Column(length=...)` 을 명시해 unique 제약 복구)한 뒤, 본 task 를 rebase 해 재개했다.

- **교훈으로 이어진 지점:** 이런 상황에서는 "develop HEAD 에서 같은 실패가 재현되는가"를 먼저
  확인해 *진짜 이 작업의 책임인지* 를 빠르게 분리해야 한다. AC 하나가 다른 도메인의 기존 결함까지
  끌어들여 작업이 막힐 수 있다.

## 배운 것

- **`@TransactionalEventListener(AFTER_COMMIT)` 은 기본 동기 실행이다.** "이벤트로 분리하면
  비동기로 격리된다"는 착각을 경계해야 한다. 진짜 비동기로 가려면 `@Async` 나 별도 `TaskExecutor`
  를 명시해야 한다. publisher 패턴의 가치는 *비동기 분리* 또는 *다중 후처리* 에 있지, 단순 코드
  분리에 있지 않다 — *단일 책임 후처리* 에는 finally 직접 호출이 더 단순하고 명확하다.
- **캐시의 책임을 명확히 정의하면 인터페이스가 단순해진다.** 부수 효과(결과 캐싱)까지 끌어안으면
  정합성 책임이 늘어난다. 기능을 얹기 전에 *"이 캐시가 진짜 주는 가치가 뭐냐 / 이것까지 책임지는
  게 맞나"* 를 한 번 더 묻는 것의 효과.
- **AC 자동 검증에 negative assertion 을 넣는 건 위험하다.** `grep` 의 exit code 의미
  (매칭 있음=0 / 없음=1)와 실행기의 *exit 0 = 통과* 가정이 충돌한다. "이 문자열이 없어야 통과"
  같은 부정 단언은 `!` 로 반전하거나 *금지사항* 으로 빼는 게 안전하다.
- **AC 가 다른 도메인의 기존 결함까지 끌어들일 수 있다.** 통합 테스트 성격의 AC 는 이 작업 범위
  밖의 결함으로도 실패한다. develop HEAD 동일 실패 재현 여부로 책임을 빠르게 가려내야 한다.
- **영향 범위 grep 을 미리 해두면 작업 분해가 명확해진다.** 이번 문서 동기화 단계는 처음엔 좁아
  보였지만 실제로는 16곳이었다(루트 ADR 문서 여러 군데 + api-spec / architecture / logging /
  testing-conventions + 기존 task ADR 3개: order-idempotency / event-outbox-trace-propagation /
  unique-find-first-policy 상단 cross-reference). 코드 1곳 변경의 실제 파급을 grep 으로 *현재
  사용처* 부터 확인하고 분해해야 규모 예측이 된다.

## 미해결·열린 질문

이 task 자체의 후속 작업은 없다. 다만 이번에 의도적으로 미룬 결정들의 재검토 트리거를 회고의
"미래 결정 시점" 으로 남겨뒀다:

- **Redis timeout 이 실제 문제가 될 만큼 잦아지면** → 비동기 listener 재도입 검토. 지금은 동기
  finally 가 Redis latency 를 응답 latency 에 직결시키고 있다.
- **외부 시스템 후처리(알림, 정산 등)가 필요해지면** → outbox 패턴 적용(기존 Kafka outbox 사례와
  같은 방향).
- **도메인 이벤트(PaymentCompleted 등) 다중 후처리 패턴이 확산되면** → Spring Event 부활 검토.
  *단일 책임 후처리* 가 아니라 *다중 후처리* 가 되는 시점에 publisher 패턴이 값을 갖는다.
- **결제 PG 통합으로 트랜잭션 안에 외부 I/O 가 들어오면** → PROCESSING TTL 재검토. 지금 60초는
  p99 latency ms 단위 + MySQL 락 대기 만료 50초 마진을 전제로 산정된 값이다.
