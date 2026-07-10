---
platform: backend
author: KimYeonWook511
created: 2026-06-08
---

## 출발 질문
멱등 논의에서 "unique 제약조건 위반을 어디서 catch하고, 어느 계층에서 도메인 예외로 번역하나"로 이어졌다. application service냐 infra adapter냐, globalExceptionHandler 공통 처리냐가 고민이었다.

## 핵심 결론
인프라 예외(`DataIntegrityViolationException`)는 **"그것을 알아도 되는 가장 낮은 계층(infra adapter)에서 잡아 도메인 예외로 번역"** 하고, 위로는 도메인 예외만 흘려보낸다. 단 JPA는 변경이 **flush 시점(기본=커밋)** 에 SQL로 나가므로, adapter에서 위반을 잡으려면 `saveAndFlush`로 flush를 앞당겨야 한다. 그런데 dirty-checking UPDATE는 `save`를 안 타서 adapter가 flush를 통제 못 하는 한계가 있고, 위반난 트랜잭션은 rollback-only로 오염되므로 보상 후처리는 반드시 **별도 트랜잭션(REQUIRES_NEW)** 에서 한다.

## 정리한 개념

### 1. unique 제약조건 위반은 두 종류 — "어디서"보다 "잡아서 뭘 하나"가 먼저
- (가) 같은 결제의 중복 콜백 → 멱등 성공 응답
- (나) 진짜 이중결제(다른 PG로 둘 다 승인) → 보상 취소(환불)
- 둘 다 같은 `DataIntegrityViolationException`으로 올라온다. 처리가 정반대라, 능동 후처리가 필요한지로 catch 전략이 갈린다.

### 2. globalExceptionHandler 공통 처리가 안 되는 이유
- globalHandler는 "예상 못 한 예외를 마지막에 받아 500"하는 안전망.
- 그런데 이중결제 위반은 **예상된 비즈니스 상황 + 능동 후처리(보상)** 가 필요. globalHandler엔 "어느 결제였는지" 컨텍스트가 없어 환불을 못 한다.
- → 능동 처리가 필요한 위반은 **그 지점에서 명시적 catch**. 그 외 모든 제약 위반은 globalHandler로 위임(500).

### 3. "전부 catch하고 검사"는 하지 않는다
- 모든 곳에서 `DataIntegrityViolationException` 잡고 제약 이름 검사 = 별로. 안 한다.
- catch는 **능동 후처리가 필요한 단 한 곳**에만(이중결제 보상). reserve 따닥은 find-first 흡수 + race window는 globalHandler 500. 나머지는 globalHandler.

### 4. 번역은 경계에서 — 원칙은 infra adapter
- 원칙: 하위 계층 기술 예외는 그 계층을 빠져나가는 경계에서 상위가 이해하는 언어(도메인 예외)로 번역.
- `DataIntegrityViolationException`은 인프라 예외 → infra adapter까지만 알아도 되는 타입 → adapter에서 잡아 `DuplicateApprovalException` 등으로 번역 → application은 도메인 예외만 안다.
- 정하는 기준: **"이 예외를 받는 계층이 이 타입을 알아도 되는가?"**

### 5. adapter에서 잡으려면 flush를 앞당겨야 (saveAndFlush) — 트랜잭션은 service가 쥐어도 됨
- JPA는 `save()` 호출 시 INSERT를 바로 안 날리고 flush(기본=커밋) 때 모아 보낸다. 그래서 위반이 **service 트랜잭션 커밋 시점**에 터지면 adapter try-catch는 못 잡는다.
- 해결: adapter 안에서 `saveAndFlush`로 INSERT를 즉시 실행 → 위반이 adapter 안에서 드러남 → 거기서 번역.
- **adapter가 트랜잭션을 시작/끝낼 필요 없다.** service 트랜잭션을 전파받되 flush 타이밍만 앞당기는 것.

### 6. saveAndFlush가 "즉시 예외"는 아니다 — S-lock 대기 (정정)
- 처음엔 "saveAndFlush하면 그 자리에서 터진다"고 단순화했지만, 동시 케이스는 다르다.
  - 이미 커밋된 중복 → 즉시 `Duplicate entry`.
  - 동시 진행 중 → InnoDB가 충돌 인덱스 레코드에 **S-lock** 걸고 **상대 커밋까지 대기** → 상대 커밋 시 그제서야 위반, 상대 롤백 시 INSERT 성공.
- 이 대기는 멱등을 깨지 않는다 — 동시 두 승인을 직렬화해 "하나만 성공"을 정확히 보장. 대기를 거쳐 위반이 확정되어 잡힌다.
- 같은 키끼리만 대기하지 남의 결제(다른 order_id)는 안 막는다. 단순 INSERT는 자기 키 자리만 잠그지 범위를 안 잠금 → gap lock과 다름. (gap lock은 "없는 행 조회 FOR UPDATE → INSERT" 패턴에서 생김, 그래서 그 패턴 금지)

### 7. 위반난 트랜잭션은 오염된다 → 보상은 REQUIRES_NEW
- adapter에서 도메인 예외로 바꿔 던져도 그 트랜잭션은 이미 **rollback-only**. 같은 트랜잭션에서 보상 기록을 DB에 쓰면 커밋 안 됨.
- 보상 취소는 **새 트랜잭션(REQUIRES_NEW)** 에서. 위반난 트랜잭션은 롤백되게 두고, 보상은 깨끗한 새 트랜잭션에서 PG 환불 + cancel 행 기록.
- 우리 `PaymentApprovalCompensationService`가 "클래스 레벨 @Transactional 없음 / 단계별 독립 커밋"인 이유가 이것.

### 8. adapter 번역이 힘든 경우 (현실적 한계)
- **dirty-checking UPDATE**: `succeed()`처럼 엔티티 필드만 바꾸면 `save`를 안 타고 커밋 때 자동 flush → adapter가 flush를 통제 못 함. (우리 uq_approved 위반이 정확히 이 경우)
- commit-time에만 드러나는 제약 / 제약 이름 판별 불가 / 위반이 맥락 의존 의미(중복 콜백 vs 이중결제, adapter는 중립 예외까지만).
- 폴백: (a) `succeed()` 후 service가 명시적 `saveAndFlush` 호출해 타이밍을 adapter로 끌어오기, 또는 (b) service 경계에서 잡되 "예외적 허용"으로 한 지점에 격리.
- **명시적 save 자체는 어색하지 않다**(오히려 명확). 미묘한 건 "save가 아니라 saveAndFlush를 고른 동기가 flush 타이밍이라는 인프라 사정"뿐 — 작은 타협이고, application이 인프라 예외를 직접 catch하는 것보다 계층상 더 깨끗.

### 9. adapter는 "계약"을 보고 구현, "호출자 흐름"을 가정하면 안 됨 (중요)
- repository 인터페이스 메서드명은 기술 중립으로 동일하게 두고, JPA adapter 안에서만 flush를 하든 말든 구현.
- 단 adapter가 "이 service에선 뒤에 flush 유발 코드가 있으니 나는 flush 안 해도 돼" 식으로 **호출 맥락을 가정하면**, 흐름이 바뀔 때 깨진다.
- 해결: 메서드 **계약을 명확히** 한다. "이 메서드는 즉시 제약 검증을 보장한다"를 계약으로 박으면(예: `saveApprovalResult`), JPA adapter는 그 계약을 지키려 saveAndFlush를 쓰고(흐름 무관), MyBatis adapter는 그냥 호출해도 계약 충족.

### 10. JDBC / MyBatis는 이 고민이 없다 (대조)
- JDBC/MyBatis는 "호출 = 즉시 SQL = 즉시 제약 검사". flush 개념 자체가 JPA(영속성 컨텍스트)에만 있음.
- 그래서 위반이 호출 자리에서 터져 adapter에서 자연스럽게 잡힌다. "update 했으면 명시적으로 반영"이 거기선 당연.
- 어느 기술이든 adapter의 책임(기술 예외 → 도메인 예외 번역)은 동일. 차이는 "flush를 앞당기느냐" 한 줄뿐.

## 결정/원칙 (commerce-backend 프로젝트)
- 인프라 예외는 그것을 알아도 되는 가장 낮은 계층(adapter)에서 도메인 예외로 번역. 위로는 도메인 예외만.
- adapter 번역을 위해 `saveAndFlush`로 flush 앞당김(트랜잭션은 service가 쥠). 단 그 save만 국소 적용(전체 saveAndFlush는 batch 이점 손실).
- dirty-checking UPDATE라 adapter가 못 잡는 곳(uq_approved)은 명시적 saveAndFlush로 끌어오거나 service 경계 catch를 "예외적 허용"으로 격리.
- 위반난 트랜잭션은 오염 → 보상은 REQUIRES_NEW.
- adapter는 메서드 계약만 보고 구현, 호출자 흐름 가정 금지.

## 열린 질문 / 주의
- uq_approved 위반이 save() 시점이냐 commit 시점이냐 확인 필요 → adapter 번역(이상) vs service 격리(차선) 결정.
- lock wait timeout(InnoDB 기본 50초) — 동시성 극단 시 대기 길어질 수 있으나 결제는 같은 order_id 동시 요청이 소수라 현실적 문제 적음.

## 관련
- [[raw/sessions/backend/2026-06-08-payment-idempotency-write-not-check]] — 멱등 패턴(이 예외 처리가 거기서 쓰임)
- [[raw/sessions/backend/2026-06-07-pr-218-pg-approve-exception-boundary]] — 전송 시점 경계, 같은 광범위 catch 두 계층 문제(같은 결)
