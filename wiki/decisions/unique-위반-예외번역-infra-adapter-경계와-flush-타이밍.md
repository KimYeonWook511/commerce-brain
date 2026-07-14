---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, infra-adapter, jpa, flush, transaction, unique-constraint, payment, innodb, concurrency]
created: 2026-06-08
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-08-jpa-flush-transaction-exception-boundary]]"
---

# DB unique 위반을 어느 계층에서 도메인 예외로 번역하나 — infra adapter 경계와 flush 타이밍

## 컨텍스트 — unique 위반을 application/adapter/공통핸들러 중 어디서 번역하나

멱등 처리를 논의하다 "DB unique 제약 위반을 어디서 catch하고 어느 계층에서 도메인 예외로 번역하나"로 넘어온 세션. application service냐 infra adapter냐, 아니면 예상 못 한 예외를 마지막에 받는 공통 핸들러(`GlobalExceptionHandler`)에 맡기냐가 쟁점이었다. 이 예외 처리는 결제 멱등 패턴이 실제로 걸리는 자리이고, 같은 시기 PG 승인 전송 시점 경계 논의([[pg-승인-예외-경계-요청전송시점]])의 "광범위 catch가 두 계층에 걸치는 같은 결의 문제"와 이어진다. 아직 코드 확정 전 방향을 잡는 대화였고, 핵심 가정은 이후 코드로 그대로 굳었다.

## 결정 — 경계에서 번역, 원칙은 infra adapter

인프라 예외(`DataIntegrityViolationException`)는 **그것을 알아도 되는 가장 낮은 계층인 infra adapter에서 잡아 도메인 예외로 번역**하고, 위로는 도메인 예외만 흘려보낸다.

- **기준:** 하위 계층의 기술 예외는 그 계층을 빠져나가는 경계에서 상위가 이해하는 언어(도메인 예외)로 번역한다. `DataIntegrityViolationException`은 인프라 예외라 영속화 adapter까지만 알아도 되는 타입이다. adapter에서 잡아 도메인 예외(예: `PaymentException(PAYMENT_DUPLICATE)`)로 번역하면 application은 도메인 예외만 알면 된다.
- **판정 질문:** "이 예외를 받는 계층이 이 타입을 알아도 되는가?" application이 `org.springframework.dao.*` 패키지에 의존하지 않게 만드는 게 목적이다([[persistence-exception-노출-경계-추상수준]]). 어느 기술이든 adapter의 책임(기술 예외 → 도메인 예외 번역)은 동일하고, JDBC/MyBatis는 "호출 = 즉시 SQL = 즉시 제약 검사"라 위반이 호출 자리에서 터져 자연스럽게 잡힌다. 차이는 JPA에서 "flush를 앞당기느냐" 한 줄뿐이다.

## find-first carve-out — 능동 후처리 필요한 위반만 명시적 catch

이 원칙은 "unique 위반은 정상 흐름을 사전 `find` 조회로 처리하고 실제 동시 충돌만 안전망 500에 위임한다"는 [[find-first-write-not-check-db-unique-멱등]] 방침의 **carve-out**이다. find-first가 기본이되, 능동 후처리(보상)가 필요한 위반만 try-save-catch로 adapter에서 번역한다.

같은 `DataIntegrityViolationException`으로 올라와도 **능동 후처리가 필요한지**로 catch 전략이 갈린다.

- **같은 결제의 중복 콜백** → 멱등 성공 응답. 능동 후처리 불필요 → 공통 핸들러 위임.
- **진짜 이중결제(다른 PG로 둘 다 승인)** → 보상 취소(환불). 능동 후처리 필요 → 그 지점에서 명시적 catch([[결제승인완료-보상-완료우선-이중결제-adapter매핑]]).

**공통 핸들러로 못 미루는 이유:** `GlobalExceptionHandler`는 예상 못 한 예외를 마지막에 받아 500으로 떨구는 안전망인데, 이중결제 위반은 예상된 비즈니스 상황 + 능동 후처리가 필요하다. 공통 핸들러엔 "어느 결제였는지" 컨텍스트가 없어 환불을 못 한다. **"전부 catch하고 제약 이름 검사"도 안 한다** — catch는 능동 후처리가 필요한 단 한 곳(이중결제 보상)에만 둔다. reserve 중복 요청(따닥)은 사전 `find` 흡수 + race window는 공통 핸들러 500, 나머지도 전부 공통 핸들러로.

## flush 타이밍 — saveAndFlush로 앞당김(계약), dirty-checking UPDATE 한계와 명시 save 폴백

JPA는 `save()` 호출 시 INSERT/UPDATE를 바로 안 날리고 flush(기본=커밋) 때 모아 보낸다. 그래서 위반이 **service 트랜잭션 커밋 시점**에 터지면 adapter의 try-catch로는 못 잡는다.

- **해결:** adapter 안에서 `saveAndFlush`로 즉시 flush → 위반이 adapter 안에서 드러남 → 거기서 번역. adapter가 트랜잭션을 시작/끝낼 필요는 없다. service 트랜잭션을 전파받되 **flush 타이밍만 앞당기는 것**이다. 이 `saveAndFlush`는 필요한 저장 지점에만 국소 적용한다(전 구간을 바꾸면 배치 flush 이점을 잃는다, [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]]).
- **계약으로 못 박음:** adapter가 "이 service엔 뒤에 flush 유발 코드가 있으니 나는 flush 안 해도 돼" 식으로 호출 맥락을 가정하면 흐름이 바뀔 때 조용히 깨진다. 그래서 메서드 계약을 "이 메서드는 즉시 제약 검증을 보장한다"로 명확히 한다. JPA adapter는 그 계약을 지키려 `saveAndFlush`를 쓰고(흐름 무관), MyBatis adapter는 호출만으로 충족한다. 그래서 `saveAndFlush`는 "그냥 저장"이 아니라 load-bearing한 계약이다.
- **dirty-checking UPDATE 한계:** `succeed()`처럼 managed 엔티티 필드만 바꾸면 `save`를 안 타고 커밋 때 자동 flush돼 adapter가 flush 타이밍을 통제 못 한다. 결제 승인 완료 저장이 정확히 이 경우다 — 승인 완료 전용 unique 제약(`uk_payment_approved_order_key`, [[payment-이중결제-reserve따닥-mysql-null트릭-unique]])이 걸리는 게 이 dirty-checking UPDATE에서 나온다.
- **택한 폴백 — 명시적 save:** 승인 완료 저장은 dirty-checking으로 바뀐 엔티티를 명시적으로 `saveAndFlush`해 위반을 그 호출 안에서 확정하고, `uk_payment_approved_order_key` 위반일 때만 `PAYMENT_DUPLICATE`로 번역한다. 명시적 save는 어색한 게 아니라 "여기서 저장한다"는 의도가 코드 표면에 드러나 명확하다 — 응용 계층이 영속화 호출을 명시하는 별도 프로젝트 방침([[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]])과 같은 결이다.

## 위반 트랜잭션 오염 → 보상은 REQUIRES_NEW 별도 트랜잭션

adapter에서 도메인 예외로 바꿔 던져도 그 트랜잭션은 이미 **rollback-only**다. 같은 트랜잭션에서 보상 기록을 DB에 쓰면 커밋되지 않는다. 그래서 보상 취소는 **새 트랜잭션(REQUIRES_NEW)**에서 한다 — 위반난 트랜잭션은 롤백되게 두고, 깨끗한 새 트랜잭션에서 PG 환불 + cancel 행 기록을 수행한다. 결제 보상 서비스가 "클래스 레벨 `@Transactional` 없음 / 단계별 독립 커밋" 구조인 이유가 이것이다([[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]).

> [!note] 후속 — 보상 트랜잭션 정책의 재검토
> 이 세션은 보상을 REQUIRES_NEW 별도 트랜잭션에 두는 것을 방향으로 잡았다. 이후 보상 트랜잭션 격리 정책은 [[requires-new-격리-제거-보상판단-트랜잭션정책]]에서 다시 다뤄졌다.

## 배운 것 — saveAndFlush는 즉시 예외가 아님(S-lock 대기), gap lock 회피

논의 초반엔 "`saveAndFlush`하면 그 자리에서 바로 터진다"고 단순화했으나 동시 케이스는 다르다는 걸 짚어 정정했다.

- **이미 커밋된 중복** → 즉시 `Duplicate entry`.
- **동시 진행 중** → InnoDB가 충돌 인덱스 레코드에 **S-lock**을 걸고 **상대 커밋까지 대기**한다. 상대가 커밋하면 그제서야 위반이 확정되고, 상대가 롤백하면 이쪽 INSERT가 성공한다.

이 대기는 멱등을 깨지 않는다 — 동시 두 승인을 직렬화해 "하나만 성공"을 정확히 보장한다. 같은 키끼리만 대기하고 다른 `order_id`는 안 막는다. 단순 INSERT는 자기 키 자리만 잠그지 범위를 안 잠가 gap lock과 다르다(gap lock은 "없는 행을 `FOR UPDATE`로 조회 → INSERT" 패턴에서 생기므로 그 패턴은 금지, [[payment-동시성-unique-vs-lock-gap-lock회피]]).

## 미해결

- **lock wait timeout(InnoDB 기본 50초):** 동시성 극단 상황에서 대기가 길어질 수 있으나, 결제는 같은 `order_id`에 동시 요청이 소수라 현실적 문제는 적다고 봤다.
- **save 시점 vs commit 시점:** 승인 완료 unique 위반이 어디서 터지는지가 adapter 번역(이상) vs service 격리(차선)를 갈랐다. 명시적 `saveAndFlush`를 승인 저장 경로에 넣어 **adapter 번역 쪽으로 확정**됐다.

## 근거

- [[raw/sessions/backend/2026-06-08-jpa-flush-transaction-exception-boundary]] — infra adapter 경계 번역, find-first carve-out, saveAndFlush 계약·dirty-checking 한계·명시 save 폴백, REQUIRES_NEW 보상, S-lock 대기 정정의 출처.
