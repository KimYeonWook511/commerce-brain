---
platform: backend
author: KimYeonWook511
created: 2026-06-04
---

> 결제 도메인 동시성/멱등을 lock이 아닌 DB unique 제약으로 보장하기로 한 이유를 정리한 근거 문서.
> 같은 날짜의 [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] 결정 6·13·15·16을 뒷받침한다.

# 결제 동시성 — 왜 lock이 아니라 unique 제약인가

> 결제 도메인의 멱등/동시성을 lock이 아니라 DB unique 제약으로 보장하기로 한 이유를 정리한 문서.
> 메인 설계 문서(`payment-order-design-decisions.md`)의 결정 6·13·15·16을 뒷받침하는 근거 문서다.
> DB는 **MySQL(InnoDB)**, 기본 격리수준 **REPEATABLE READ** 전제.

---

## 결론 먼저

- **흐름을 가로질러 지켜야 하는 불변식**(주문당 진행 중 reserve 1개, 주문당 성공 승인 1개)은 **DB unique 제약**으로 보장한다.
- **lock(`SELECT ... FOR UPDATE`)** 은 **한 트랜잭션 안에서 끝나는 짧은 계산**(부분취소 과다취소 검증)에만 쓴다.
- 이유는 두 가지: (1) **lock은 트랜잭션 수명에 묶여 긴 결제 흐름을 가로지를 수 없다.** (2) "없는 행 조회 후 INSERT" 패턴의 lock은 **gap lock으로 남의 결제까지 막는다.**

---

## 1. 결제 흐름은 하나의 트랜잭션이 아니다

결제는 여러 트랜잭션과 외부 대기로 쪼개진다.

```
[TX1] reserve → RESERVED 행 생성, 커밋          ← 여기서 lock은 풀린다
   ⏸ 사용자가 결제창에서 결제 (수 초 ~ 수 분, 우리 트랜잭션 밖)
[TX 아님] 서버가 PG 승인 API 호출                ← DB 트랜잭션 아님, 네트워크 대기
[TX2] 승인 결과 수신 → SUCCEEDED + 주문 PAID, 커밋  ← TX1과 완전히 다른 요청/스레드/트랜잭션
```

여기서 redirect로 돌아오는 TX2는 **다른 HTTP 요청, 다른 스레드, 다른 트랜잭션**이다.

---

## 2. lock은 트랜잭션이 끝나면 소멸한다

`SELECT ... FOR UPDATE`로 잡은 lock은 **트랜잭션이 커밋/롤백되는 순간 즉시 풀린다.** 따라서 위 흐름을 lock으로 묶으려 하면 전부 막힌다.

- **TX1의 lock은 reserve 커밋과 함께 사라진다.** 사용자가 결제하는 동안 유지되지 않는다.
- **사용자 입력을 기다리며 트랜잭션을 열어둔 채 lock을 쥐는 것**은 커넥션을 수 분간 점유하는 안티패턴이다. 동시 사용자가 많으면 커넥션 풀이 즉시 고갈된다. (절대 금지)
- **TX1에서 잡은 lock이 TX2로 이어질 방법이 원천적으로 없다.** 다른 트랜잭션이기 때문이다.

즉 lock은 "한 트랜잭션 안에서 짧게 직렬화"하는 도구이지, "사용자와 PG를 기다리는 긴 흐름을 가로질러 상태를 지키는" 도구가 아니다. 결제 흐름은 후자라서 lock이 맞지 않는다.

---

## 3. unique 제약은 트랜잭션 수명에 묶이지 않는다

unique 제약은 lock과 달리 **데이터가 존재하는 한 영구히** 불변식을 지킨다. "쥐고 기다리는" 것이 아니라, 각 트랜잭션이 자기 시점에 INSERT/UPDATE를 시도하고 충돌하면 DB가 거부한다.

```
[TX1] RESERVED INSERT → reserved_key unique가 "그 시점에" 따닥 차단
       커밋 후에도 reserved_key 값은 행에 남는다
   ⏸ 사용자 결제 (몇 분)
[TX2] APPROVE SUCCEEDED + approved_order_key set
       → approved_order_key unique가 "그 시점에" 이중결제 차단
```

각 분기점에서 unique가 **독립적으로** 불변식을 지키므로, 흐름이 여러 트랜잭션·여러 요청·외부 대기로 쪼개져 있어도 문제가 없다.

### lock vs unique 비교

| | Lock (`FOR UPDATE`) | Unique 제약 |
|---|---|---|
| 수명 | 트랜잭션이 끝나면 소멸 | 데이터가 사는 한 영구 |
| 긴 흐름 가로지르기 | 불가 (커밋하면 풀림) | 가능 (행에 값이 남음) |
| 외부 대기 중 유지 | 불가 (커넥션 점유 안티패턴) | 해당 없음 (쥐는 게 아님) |
| 다른 요청/스레드로 이어짐 | 불가 | 가능 (같은 행/값을 보면 됨) |

---

## 4. gap lock — lock의 또 다른 함정

lock이 흐름을 못 묶는 것과 별개로, InnoDB의 lock에는 동시성 부작용도 있다.

- **Row lock**: 특정 행만 잠근다. 서로 다른 주문이면 충돌 없음 → 남의 결제 안 막음.
- **Gap lock(next-key lock)**: REPEATABLE READ에서 **존재하지 않는 행을 조건으로 잠그거나 범위로 잠그면**, 그 사이 빈 구간까지 잠가 **다른 트랜잭션의 INSERT를 막는다.** 인덱스를 따라 걸리므로 적절한 인덱스가 없으면 더 넓게 잠긴다.
- 특히 **"없는 행 조회 `FOR UPDATE` → 없으면 INSERT" 패턴이 gap lock을 유발**한다. (reserve 재사용에서 쓰려던 바로 그 패턴)

→ "없으면 INSERT, 있으면 막기"를 lock으로 하면 gap lock으로 옆 범위의 INSERT, 즉 남의 결제까지 막을 수 있다. unique 제약은 그냥 INSERT를 시도하고 충돌 시 거부하므로 gap을 미리 잠그지 않는다 → 남의 결제에 영향이 없다.

---

## 5. 그래서 어디에 무엇을 쓰는가

| 상황 | 방법 | 이유 / 남의 결제 영향 |
|---|---|---|
| reserve 따닥 (없으면 INSERT) | **unique 제약** | 흐름을 가로지름 + gap lock 회피. 영향 없음 |
| 이중결제 (주문당 성공 승인 1개) | **unique 제약** | 흐름을 가로지름(reserve→승인이 다른 트랜잭션). 영향 없음 |
| 부분취소 과다취소 검증 (계산 후 판단) | order 행 **`FOR UPDATE`**(PK 단일 행) | 한 트랜잭션 안에서 끝남 + PK 단일 행이라 gap 없음. 영향 없음 |
| 피해야 할 것 | 없는 행 조회 `FOR UPDATE` → INSERT | gap lock으로 남의 INSERT 차단 ⚠️ |

### unique 제약은 MySQL에서 NULL 트릭으로 구현

MySQL(InnoDB)은 조건부(partial) unique index를 지원하지 않으므로, "조건 만족 시에만 값, 아니면 NULL" 컬럼 + 일반 unique로 대체한다(InnoDB는 NULL을 중복으로 치지 않음). 상세는 메인 문서 결정 6·13 참고.

```sql
-- 이중결제: 성공 APPROVE일 때만 order_id, 그 외 NULL
CREATE UNIQUE INDEX uk_approved ON payment (approved_order_key);

-- reserve 따닥: RESERVED일 때만 "orderId:provider", 그 외 NULL
CREATE UNIQUE INDEX uk_reserved ON payment (reserved_key);
```

조건 컬럼의 set/NULL은 status 전이와 **같은 트랜잭션·같은 UPDATE**에서 처리한다(엔티티 상태 전이 메서드에 캡슐화).

---

## 6. 부분취소 검증에는 왜 lock이 유효한가

부분취소 과다취소 검증은 시작부터 끝까지 **한 트랜잭션 안에서, 외부 대기 없이** 끝난다. 흐름을 가로지르지 않으므로 lock이 유효하다.

```
[TX] order 행 FOR UPDATE          -- PK 단일 행 잠금 (gap 없음)
     → 잔액 = SUM(APPROVE) - SUM(CANCEL)  (성공 행만)
     → 취소액 ≤ 잔액 검증
     → CANCEL INSERT
     → 커밋 (lock 해제)
```

"여러 행을 읽어 합산한 결과로 판단"하는 작업이라 단일 행 unique로는 표현할 수 없고, 한 트랜잭션 안에서 끝나므로 lock이 적합하다. PK 단일 행을 잠그므로 gap lock도 생기지 않고, 그 주문에만 직렬화되어 남의 결제에 영향이 없다.

---

## 한 줄 요약

> 결제 흐름은 reserve / 사용자 결제 / PG 호출 / 승인 반영이 서로 다른 트랜잭션이고 중간에 외부를 기다리므로, 트랜잭션과 함께 소멸하는 lock으로는 흐름 전체를 가로질러 불변식을 지킬 수 없다. 그래서 "흐름을 가로지르는 불변식"(reserve 1개·성공 승인 1개)은 트랜잭션 수명에 묶이지 않는 **unique 제약**으로 보장하고(추가로 gap lock도 피함), lock은 "한 트랜잭션 안에서 끝나는 계산"(부분취소 과다취소 검증)에만 **PK 단일 행**으로 좁게 쓴다.
