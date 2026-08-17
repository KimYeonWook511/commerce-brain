---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, concurrency, unique-constraint, gap-lock, pessimistic-lock, innodb, mysql]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]]"
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]"
---

# 결제 동시성 — 왜 lock이 아니라 unique 제약인가 (gap lock 회피)

결제 도메인 재설계 전반이 기대는 한 가지 판단 — "흐름을 가로지르는 불변식은 lock이 아니라 unique로 지킨다" — 을 따로 떼어 정리한 결정이다. DB는 **MySQL(InnoDB)**, 기본 격리수준 **REPEATABLE READ** 전제.

## 배경 — 결제에 lock을 걸까?

결제는 돈이 걸린 예민한 흐름이라 "동시성을 lock으로 직렬화하자"는 직감이 먼저 온다. 그런데 두 가지 걸림돌이 있었다. (1) lock을 걸면 남의 결제까지 막지 않을까? (2) 애초에 결제 흐름을 lock으로 묶을 수 있기는 한가? 파고들어보니 후자가 근본 문제였다 — 결제 흐름은 하나의 트랜잭션이 아니다.

## 왜 lock으로 안 되나 — 흐름은 한 트랜잭션이 아니다

결제는 여러 트랜잭션과 외부 대기로 쪼개진다.

```
[TX1] reserve → RESERVED 행 생성, 커밋          ← 여기서 lock은 풀린다
   ⏸ 사용자가 결제창에서 결제 (수 초 ~ 수 분, 우리 트랜잭션 밖)
[TX 아님] 서버가 PG 승인 API 호출                ← DB 트랜잭션 아님, 네트워크 대기
[TX2] 승인 결과 수신 → SUCCEEDED + 주문 PAID, 커밋  ← TX1과 다른 요청/스레드/트랜잭션
```

redirect로 돌아오는 TX2는 다른 HTTP 요청·스레드·트랜잭션이다. `SELECT ... FOR UPDATE`로 잡은 lock은 트랜잭션이 커밋/롤백되는 순간 즉시 풀리므로, 이 흐름을 lock으로 묶으려 하면 전부 막힌다.

- TX1의 lock은 reserve 커밋과 함께 사라진다 — 사용자가 결제하는 동안 유지되지 않는다.
- 사용자 입력을 기다리며 트랜잭션을 열어둔 채 lock을 쥐는 것은 커넥션을 수 분간 점유하는 안티패턴이다(동시 사용자 많으면 커넥션 풀 즉시 고갈, 절대 금지).
- TX1에서 잡은 lock이 TX2로 이어질 방법이 원천적으로 없다(다른 트랜잭션이므로).

즉 lock은 "한 트랜잭션 안에서 짧게 직렬화"하는 도구이지, "사용자와 PG를 기다리는 긴 흐름을 가로질러 상태를 지키는" 도구가 아니다. 외부 호출을 트랜잭션 밖에 두고 DB 쓰기만 한 트랜잭션에 묶는 경계는 [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]] 참조.

## unique는 트랜잭션 수명에 안 묶인다

unique 제약은 데이터가 존재하는 한 영구히 불변식을 지킨다. "쥐고 기다리는" 게 아니라, 각 트랜잭션이 자기 시점에 INSERT/UPDATE를 시도하고 충돌하면 DB가 거부한다. 각 분기점에서 unique가 독립적으로 불변식을 지키므로, 흐름이 여러 트랜잭션·요청·외부 대기로 쪼개져 있어도 문제가 없다.

| | Lock (`FOR UPDATE`) | Unique 제약 |
|---|---|---|
| 수명 | 트랜잭션 끝나면 소멸 | 데이터가 사는 한 영구 |
| 긴 흐름 가로지르기 | 불가(커밋하면 풀림) | 가능(행에 값 남음) |
| 외부 대기 중 유지 | 불가(커넥션 점유 안티패턴) | 해당 없음(쥐는 게 아님) |
| 다른 요청/스레드로 이어짐 | 불가 | 가능(같은 행/값을 보면 됨) |

이 판단 위에서 "주문당 진행 중 reserve 1개"와 "주문당 성공 승인 1개"를 unique로 보장한다. MySQL엔 partial unique index가 없어 NULL 트릭으로 구현하며(조건 만족 시에만 값, 아니면 NULL), 실제 구현·캡슐화 규칙은 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]에 있다. 이 문서와 그 노트는 같은 재설계에서 저자가 "이유"와 "구현"을 나눠 남긴 것이라 강하게 상호참조한다.

## gap lock 함정과 회피

lock이 흐름을 못 묶는 것과 별개로, InnoDB lock에는 동시성 부작용도 있다.

- **Row lock**: 특정 행만 잠근다. 서로 다른 주문이면 충돌 없음 → 남의 결제 안 막음.
- **Gap lock(next-key lock)**: REPEATABLE READ에서 존재하지 않는 행을 조건으로 잠그거나 범위로 잠그면 빈 구간까지 잠가 다른 트랜잭션의 INSERT를 막는다. 인덱스를 따라 걸려 적절한 인덱스가 없으면 더 넓게 잠긴다.
- 특히 **"없는 행 조회 `FOR UPDATE` → 없으면 INSERT" 패턴이 gap lock을 유발**한다 — reserve 재사용에서 쓰려던 바로 그 패턴이 여기 해당한다.

→ "없으면 INSERT, 있으면 막기"를 lock으로 하면 gap lock으로 옆 범위의 INSERT, 즉 남의 결제까지 막을 수 있다. unique는 그냥 INSERT를 시도하고 충돌 시 거부하므로 gap을 미리 잠그지 않는다 → 남의 결제에 영향이 없다.

## 어디에 무엇을 쓰는가

| 상황 | 방법 | 이유 / 남의 결제 영향 |
|---|---|---|
| reserve 따닥 (없으면 INSERT) | **unique 제약** | 흐름 가로지름 + gap lock 회피. 영향 없음 |
| 이중결제 (주문당 성공 승인 1개) | **unique 제약** | 흐름 가로지름(reserve→승인이 다른 트랜잭션). 영향 없음 |
| 부분취소 과다취소 검증 (계산 후 판단) | order 행 **`FOR UPDATE`**(PK 단일 행) | 한 트랜잭션 안에서 끝남 + PK 단일 행이라 gap 없음. 영향 없음 |
| 피해야 할 것 | 없는 행 조회 `FOR UPDATE` → INSERT | gap lock으로 남의 INSERT 차단 ⚠️ |

결론: "lock을 안 걸고 싶다"는 직감은 옳았다. 단 그 대신 unique 제약이 거는 짧은 잠금에 맡기는 것이고, lock이 꼭 필요한 곳(계산 판단)은 PK 단일 행으로 좁게 건다.

## 부분취소 계산 판단엔 PK 단일 행 lock이 유효

부분취소 과다취소 검증은 시작부터 끝까지 한 트랜잭션 안에서 외부 대기 없이 끝난다. 흐름을 가로지르지 않으므로 lock이 유효하다.

```
[TX] order 행 FOR UPDATE          -- PK 단일 행 잠금 (gap 없음)
     → 잔액 = SUM(APPROVE) - SUM(CANCEL)  (성공 행만)
     → 취소액 ≤ 잔액 검증
     → CANCEL INSERT
     → 커밋 (lock 해제)
```

"여러 행을 읽어 합산한 결과로 판단"하는 작업이라 단일 행 unique로는 표현할 수 없고, 한 트랜잭션 안에서 끝나므로 lock이 적합하다. PK 단일 행이라 gap lock도 없고 그 주문에만 직렬화된다.

이 원칙은 실제 PAID 취소 경로에 그대로 나타난다 — order 행을 `PESSIMISTIC_WRITE`(`where o.id=:orderId and o.memberId=:memberId`, id가 PK라 사실상 단일 행 락)로 잠근 뒤 PAID 검증 → 환불 대상 조회 → CANCEL 영속화 → `order.cancel()` → 재고 복구를 한 트랜잭션에서 처리한다. 자식(order_item)까지 락이 번지지 않도록 join fetch 대신 단일 행 락 + 아이템 별도 로드로 분리한 근거는 [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]에 상술돼 있다. 지금은 전액취소만 운영하고 과다취소 검증(SUM 계산)은 부분취소를 켤 때 이 락 위에 얹는다.

## 미해결 — 부분취소 과다취소 검증

부분취소(과다취소 검증)는 아직 구현하지 않았다. 모델(금액을 가진 CANCEL 행 + 잔액 SUM 도출)만 열어 뒀고, 켤 때 위 order 행 PK lock 위에 "취소액 ≤ 잔액" 검증을 얹으면 된다(지금은 전액취소만 대상). 부분취소 모델을 열어둔 결정은 [[payment-부분취소-모델만-열고-구현-보류]] 참조.

> [!note] 후속 (2026-08) — 이 노트가 예약해둔 "합산 검증" 자리가 정리됐다
> 부분취소가 실제로 들어오면서 잠금 범위가 확정됐다 — **주문 행 하나만 잠그고 품목 행은 잠그지 않는다**([[부분취소-동시성-주문행-단일잠금과-캐시겹-미도입]]). 다만 결제 쪽 한도 검증은 합산 조회가 아니라 **누적액 컬럼 하나 읽기**로 바뀌어 낙관 락만으로 성립한다([[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]).
> 한편 이 노트를 인용한 게이트 항목 하나가 **실재하지 않는 문장을 근거로 댔다** — 이 문서에 "합산"이라는 단어가 없는데 "합산 계산 판단은 비관 락으로 이미 규정했다"고 적혀 통과했다. 결론은 타당했고 근거만 허구였던 사례다([[명세-자기검토의-한계와-인용한-규약의-실재성-검증]]).

## 근거

- [[raw/sessions/backend/2026-06-04-payment-concurrency-unique-vs-lock]]
- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]
- [[raw/sessions/backend/2026-06-18-pr-258-paid-order-cancel-refund-design]]
