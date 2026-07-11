---
platform: backend
author: KimYeonWook511
created: 2026-06-08
---

# unique 위반을 어느 계층에서 도메인 예외로 번역하나 — infra adapter 경계 + JPA flush 타이밍

멱등 처리를 논의하다 "DB unique 제약 위반을 어디서 catch하고 어느 계층에서 도메인 예외로 번역하나"로 넘어온 설계 논의 세션이다. application service냐 infra adapter냐, 아니면 예상 못 한 예외를 마지막에 받는 공통 예외 핸들러(`GlobalExceptionHandler`)에 맡기냐가 쟁점이었다. 이 예외 처리는 결제 멱등(사전 조회로 흡수하고 실제 충돌만 안전망에 위임하는) 패턴이 실제로 걸리는 자리이자, 같은 시기 PG 승인 전송 시점 경계 논의에서 나온 "광범위 catch가 두 계층에 걸치는 같은 결의 문제"와 이어진다. 아직 코드로 확정하기 전 방향을 잡는 대화라 결론과 함께 몇 가지 미확인 가정을 열어둔 채 정리했다 — 그중 핵심 가정(아래)은 이후 코드로 그대로 굳었다.

## 결정한 것

핵심 결론부터: 인프라 예외(`DataIntegrityViolationException`)는 **그것을 알아도 되는 가장 낮은 계층인 infra adapter에서 잡아 도메인 예외로 번역**하고, 위로는 도메인 예외만 흘려보낸다. 단 JPA는 변경을 flush 시점(기본은 커밋)에 SQL로 내보내므로, adapter에서 위반을 잡으려면 `saveAndFlush`로 flush를 앞당겨야 한다. 그런데 엔티티 필드만 바꾸는 dirty-checking UPDATE는 `save`를 안 타서 adapter가 flush를 통제 못 하는 한계가 있고, 위반난 트랜잭션은 rollback-only로 오염되므로 보상 후처리는 반드시 별도 트랜잭션에서 한다.

### 번역은 경계에서 — 원칙은 infra adapter

- **기준:** 하위 계층의 기술 예외는 그 계층을 빠져나가는 경계에서, 상위가 이해하는 언어(도메인 예외)로 번역한다. `DataIntegrityViolationException`은 인프라 예외라 영속화 adapter까지만 알아도 되는 타입이다. adapter에서 잡아 도메인 예외(예: `PaymentException(PAYMENT_DUPLICATE)`)로 번역하면 application은 도메인 예외만 알면 된다.
- **판정 질문:** "이 예외를 받는 계층이 이 타입을 알아도 되는가?" application이 `org.springframework.dao.*` 패키지에 의존하지 않게 만드는 게 목적이다.
- 이 원칙은 "DB unique 위반은 정상 흐름을 사전 `find` 조회로 처리하고 실제 동시 충돌만 안전망 500에 위임한다(find-first)"는 기존 프로젝트 방침의 **carve-out**에 해당한다. find-first가 기본이되, 능동 후처리(보상)가 필요한 위반만 try-save-catch로 adapter에서 번역한다.

### unique 위반은 두 종류 — "어디서"보다 "잡아서 뭘 하나"가 먼저

같은 `DataIntegrityViolationException`으로 올라와도 처리가 정반대라, **능동 후처리가 필요한지**로 catch 전략이 갈린다.

- **같은 결제의 중복 콜백** → 멱등 성공 응답. 능동 후처리 불필요.
- **진짜 이중결제(다른 PG로 둘 다 승인)** → 보상 취소(환불). 능동 후처리 필요.

### 공통 예외 핸들러로 못 미루는 이유

- `GlobalExceptionHandler`는 예상 못 한 예외를 마지막에 받아 500으로 떨구는 안전망이다.
- 그런데 이중결제 위반은 **예상된 비즈니스 상황 + 능동 후처리(보상)**가 필요하다. 공통 핸들러엔 "어느 결제였는지" 컨텍스트가 없어 환불을 못 한다.
- → 능동 처리가 필요한 위반은 **그 지점에서 명시적 catch**, 그 외 모든 제약 위반은 공통 핸들러로 위임(500).

### "전부 catch하고 제약 이름 검사"는 안 한다

- 모든 곳에서 `DataIntegrityViolationException`을 잡고 제약 이름을 검사하는 방식은 택하지 않는다.
- catch는 **능동 후처리가 필요한 단 한 곳**(이중결제 보상)에만 둔다. reserve 중복 요청(따닥)은 사전 `find` 흡수 + race window는 공통 핸들러 500, 나머지도 전부 공통 핸들러로.

### adapter에서 잡으려면 flush를 앞당긴다 (트랜잭션은 service가 쥔 채)

- JPA는 `save()` 호출 시 INSERT/UPDATE를 바로 안 날리고 flush(기본=커밋) 때 모아 보낸다. 그래서 위반이 **service 트랜잭션 커밋 시점**에 터지면 adapter의 try-catch로는 못 잡는다.
- 해결: adapter 안에서 `saveAndFlush`로 즉시 flush → 위반이 adapter 안에서 드러남 → 거기서 번역. adapter가 트랜잭션을 시작/끝낼 필요는 없다. service 트랜잭션을 전파받되 **flush 타이밍만 앞당기는 것**이다.
- 이 `saveAndFlush`는 필요한 저장 지점에만 국소 적용한다. 전 구간을 `saveAndFlush`로 바꾸면 배치 flush 이점을 잃는다.

### 위반난 트랜잭션은 오염된다 → 보상은 별도 트랜잭션

- adapter에서 도메인 예외로 바꿔 던져도 그 트랜잭션은 이미 **rollback-only**다. 같은 트랜잭션에서 보상 기록을 DB에 쓰면 커밋되지 않는다.
- 보상 취소는 **새 트랜잭션(REQUIRES_NEW)**에서 한다. 위반난 트랜잭션은 롤백되게 두고, 깨끗한 새 트랜잭션에서 PG 환불 + cancel 행 기록을 수행한다.
- 결제 보상을 담당하는 서비스가 "클래스 레벨 `@Transactional` 없음 / 단계별로 독립 커밋되는 구조"인 이유가 이것이다.

### adapter는 "계약"을 보고 구현, "호출자 흐름"은 가정하지 않는다

- repository 인터페이스의 메서드명은 기술 중립으로 두고, JPA adapter 안에서만 flush를 하든 말든 구현한다.
- 단 adapter가 "이 service에선 뒤에 flush 유발 코드가 있으니 나는 flush 안 해도 돼" 식으로 **호출 맥락을 가정하면**, 흐름이 바뀔 때 조용히 깨진다.
- 해결: 메서드 **계약을 명확히** 한다. "이 메서드는 즉시 제약 검증을 보장한다"를 계약으로 박으면(승인 저장 전용 메서드처럼), JPA adapter는 그 계약을 지키려 `saveAndFlush`를 쓰고(흐름 무관), MyBatis adapter는 그냥 호출만으로 계약을 충족한다.

## 막힌 점·해결

### adapter가 flush를 통제 못 하는 경우 — dirty-checking UPDATE

- **한계:** `succeed()`처럼 관리 상태(managed) 엔티티의 필드만 바꾸면 `save`를 안 타고 커밋 때 자동 flush된다. 이러면 adapter가 flush 타이밍을 통제할 수 없다. 결제 승인 완료 저장이 정확히 이 경우다 — 승인 완료 전용 unique 제약(`uk_payment_approved_order_key`)이 걸리는 게 바로 이 dirty-checking UPDATE에서 나온다.
- 그 외에도 commit 시점에만 드러나는 제약, 제약 이름을 판별하기 어려운 경우, 위반의 의미가 맥락 의존(중복 콜백 vs 이중결제 — adapter는 중립 예외까지만 안다)인 경우가 adapter 번역이 힘든 유형이다.
- **폴백 두 가지:** (a) `succeed()` 후 service가 명시적으로 저장을 호출해 flush 타이밍을 adapter로 끌어오기, (b) service 경계에서 잡되 "예외적 허용"으로 한 지점에 격리.
- **명시적 save 자체는 어색하지 않다** — 오히려 "여기서 저장한다"는 의도가 코드 표면에 드러나 명확하다(응용 계층이 영속화 호출을 명시적으로 표현한다는 별도 프로젝트 방침과도 같은 결). 미묘한 건 "`save`가 아니라 `saveAndFlush`를 고른 동기가 flush 타이밍이라는 인프라 사정"뿐인데, 이건 작은 타협이고 application이 인프라 예외를 직접 catch하는 것보다 계층상 더 깨끗하다.
- **이후 실제 확정:** 승인 완료 저장은 폴백 (a)를 택했다. 승인 완료 저장 전용 경로가 dirty-checking으로 바뀐 엔티티를 명시적으로 `saveAndFlush`해 위반을 그 호출 안에서 확정하고, `uk_payment_approved_order_key` 위반일 때만 `PAYMENT_DUPLICATE` 도메인 예외로 번역한다. 아래 열린 질문에 있던 "save 시점이냐 commit 시점이냐"가 이 방향으로 정리된 셈이다.

### `saveAndFlush`가 "즉시 예외"는 아니다 — S-lock 대기 (정정)

- 논의 초반엔 "`saveAndFlush`하면 그 자리에서 바로 터진다"고 단순화했는데, 동시 케이스는 다르다는 걸 짚고 정정했다.
  - **이미 커밋된 중복** → 즉시 `Duplicate entry`.
  - **동시 진행 중** → InnoDB가 충돌 인덱스 레코드에 **S-lock**을 걸고 **상대 커밋까지 대기**한다. 상대가 커밋하면 그제서야 위반이 확정되고, 상대가 롤백하면 이쪽 INSERT가 성공한다.
- 이 대기는 멱등을 깨지 않는다 — 동시 두 승인을 직렬화해 "하나만 성공"을 정확히 보장한다. 대기를 거쳐 위반이 확정되어 잡힌다.
- 같은 키끼리만 대기하고 다른 주문(`order_id`)은 안 막는다. 단순 INSERT는 자기 키 자리만 잠그지 범위를 잠그지 않아 gap lock과 다르다(gap lock은 "없는 행을 `FOR UPDATE`로 조회 → INSERT" 패턴에서 생기므로, 그 패턴은 금지한다).

## 배운 것

- **JDBC / MyBatis는 이 고민이 없다.** "호출 = 즉시 SQL = 즉시 제약 검사"라 위반이 호출 자리에서 터져 adapter에서 자연스럽게 잡힌다. flush 개념 자체가 영속성 컨텍스트를 가진 JPA에만 있다. 어느 기술이든 adapter의 책임(기술 예외 → 도메인 예외 번역)은 동일하고, 차이는 "flush를 앞당기느냐" 한 줄뿐이다.
- **JPA에서 인프라 예외를 adapter에서 번역하려면 flush 타이밍이 load-bearing 의존이 된다.** `saveAndFlush`의 조기 flush가 위반을 그 호출 안에서 확정한다는 사실 위에 번역이 성립한다 — 그래서 이건 "그냥 저장"이 아니라 계약이다.
- **위반난 트랜잭션은 rollback-only로 오염되므로 보상은 반드시 트랜잭션 밖에서.** 능동 후처리(보상)와 트랜잭션 경계를 분리하지 않으면 보상 기록이 함께 롤백된다.

## 미해결·열린 질문

- 승인 완료 unique 위반이 `save()` 시점에 터지는가 commit 시점에 터지는가를 확인해야 adapter 번역(이상)과 service 격리(차선) 중 방향이 갈린다. (→ 이후 명시적 `saveAndFlush`를 승인 저장 경로에 넣어 adapter 번역 쪽으로 확정.)
- lock wait timeout(InnoDB 기본 50초) — 동시성 극단 상황에서 대기가 길어질 수 있으나, 결제는 같은 `order_id`에 동시 요청이 소수라 현실적 문제는 적다고 봤다.
