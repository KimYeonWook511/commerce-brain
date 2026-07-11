---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 224 }
---

# 결제 실시간 보상·예외 처리의 두 부채 발견 — 정상 결제 환불 버그와 갈라진 이중결제 보상

같은 PR(#224)에서 함께 다룬 결제 후처리 정책을 리뷰하던 중, 결제 *실시간 보상/예외 처리*에 더 깊은
부채가 있음을 발견했다. 무대는 `NaverPayApprovalService.completeVerifiedApproval` — NaverPay PG가
승인 SUCCESS(= 돈 캡처 완료)를 준 뒤, 우리 DB에 승인기록을 SUCCEEDED로 확정하는 전이
(`succeedApproval`)와 주문 PAID 반영을 처리하는 메서드다. 그 안의 catch 흐름에서 두 가지 문제
(정상 결제를 환불하는 버그, 그리고 예외 전략 위반 + dead code + 갈라진 이중결제 보상)를 확인하고,
해결 방향을 잡아 후속 이슈 #225로 넘겨 ADR로 확정·구현하기로 했다. 이 메모는 그 발견과 결정이다.

## 막힌 점·발견

### 발견 1 (가장 심각) — 정상 결제를 환불하고 있었다

- **증상:** 승인기록 확정 전이(`succeedApproval`) 반영이 일시적 실패(DB 데드락 등)로 예외를 던지면,
  `completeVerifiedApproval`의 catch가 `compensateUnexpected`로 보낸다. 이 보상 경로는 "예상 못 한
  예외를 한 묶음으로 받아 승인기록을 `FAILED(APPROVE_PROCESS_FAILED)`로 마킹하고 PG 환불까지 하는"
  경로다.
- **왜 버그인가:** 이 지점은 이미 키·금액 검증을 통과한 *맞는 결제*다. 즉 "틀린 결제"가 아니라
  "맞는 결제인데 우리 DB 기록만 실패"한 상태다. 이걸 환불하면 transient 실패에도 정당한 매출을
  취소해버린다. `compensateUnexpected`는 "완료가 맞음 / FAILED가 맞음 / 진짜 버그"를 한 status로
  싸잡아 처리하고 있었다.
- **올바른 모델 — 완료 재시도:** 승인기록을 실패로 마킹하지 말고 REQUESTED로 남긴다. 그러면 결제
  후처리 대사(PG 재확인 → APPROVED 확인 → 완료)가 그 흔적을 self-heal한다. 예상 못 한 예외를 한
  status로 뭉뚱그리지 말고 전파(500)해서 진짜 버그는 드러낸다. (실시간 경로는 "완료 또는 REQUESTED
  흔적 남김"까지만 책임지고, 결과 확정·복구는 배치 대사에 맡긴다.)

### 발견 2 — 예외 전략 위반 + dead code + 갈라진 이중결제 보상

- **예외 전략 위반:** 이 도메인의 예외 전략은 "DB 무결성 위반(unique 등)을 application/adapter가
  직접 catch하지 말고, 정상 중복은 사전조회(find-first)로 흡수하고 race 위반은 안전망 500에
  위임한다"이다(예외 처리 전략 문서 + find-first 통일 결정에서 확립). 그런데
  `completeVerifiedApproval`은 이중결제 환불을 위해 그 전략 확립 *이후* 다시
  `catch (DataIntegrityViolationException)`(DB 무결성 위반 예외를 application에서 직접 catch)을
  들여왔다 — 전략과 정면 충돌.
- **이중결제 보상이 두 갈래로 갈라져 있었다:**
  - 실제로 쓰이는 쪽 `compensateDuplicateApproval`(cancel-first): cancel 기록을 *먼저* 만들고
    승인기록 실패 마킹은 *나중에* 한다. 이 순서 때문에 크래시 시 "승인기록은 REQUESTED인데 cancel
    기록이 존재"라는, 어느 정책도 가정하지 않은 잔여 상태가 남는다.
  - 호출되지 않는 dead 쪽 `case PAYMENT_DUPLICATE → compensateDuplicatePayment`(fail-first):
    승인기록 실패를 *먼저* 마킹한다(다른 보상들과 같은 순서). `DataIntegrityViolationException`
    catch가 `PAYMENT_DUPLICATE`를 곧장 던져 밖으로 전파해버리므로, `PaymentException` catch 안의
    `case PAYMENT_DUPLICATE` 분기에는 도달할 경로가 없어 죽어 있었다.
  - **객관적으로 dead 쪽(fail-first)이 낫다** — 실패-먼저라 위 "승인기록 REQUESTED인데 cancel 존재"
    잔여 상태를 애초에 만들지 않는다.

## 결정한 것

발견 1의 완료 재시도 모델과 발견 2의 보상 정리를 후속 이슈 #225로 넘겨 확정한다. 핵심은 이중결제
탐지를 **application의 raw DB 예외 catch에서 adapter의 도메인 예외 매핑으로** 옮기는 것이다.

### 1. 이중결제 탐지를 adapter가 도메인 예외로 번역하도록 이관

- **왜 unique 위반 처리가 어딘가엔 반드시 필요한가:** `uk_payment_approved_order_key`(한 주문에
  SUCCEEDED 승인 1건만 허용하는 제약. `approved_order_key` 컬럼이 APPROVE+SUCCEEDED 전이 때 한 번만
  채워지는 NULL-트릭 부분 unique — NULL 행끼리는 충돌하지 않고 값이 채워진 행만 유일성을 강제)는
  동시 race를 사전조회(find-first)로 막을 수 없다. 사전 `find`와 insert 사이에 상대가 끼어들 수
  있어서다. 그래서 unique 위반을 실제로 처리하는 지점이 어딘가엔 있어야 한다.
- **현행(위반):** application이 raw `DataIntegrityViolationException`을 직접 catch한다.
- **결정:** adapter(`PaymentRepositoryAdapter`, 결제 저장 담당 JPA infra adapter)가
  `saveAndFlush`(저장 즉시 flush)에서 나는 unique 위반을 catch해 도메인 예외
  `PaymentException(PAYMENT_DUPLICATE)`로 번역한다. application은 raw DB 예외 대신 도메인 예외로
  반응한다(죽어 있던 `case PAYMENT_DUPLICATE`가 살아난다). 이러면 application의 raw DB 예외 의존이
  사라지고, 예외 전략의 예외조항("try-save-catch를 쓰더라도 인프라 예외 타입에 직접 의존하지 말고
  adapter에서 도메인 예외로 번역한다")에 부합한다.
- **부수 효과 — 잔여 상태 자동 해소:** 이 전환으로 fail-first 보상(`compensateDuplicatePayment`)이
  live가 되고 cancel-first 보상(`compensateDuplicateApproval`)이 제거된다. 그 결과 위 "승인기록
  REQUESTED인데 cancel 존재" 잔여까지 자동으로 사라진다.
- **매핑 범위 한정:** 매핑은 *승인-성공 저장 전용 경로*로만 건다. generic save에 걸면 FK/NOT
  NULL/다른 unique 위반까지 `PAYMENT_DUPLICATE`로 오매핑된다. 제약명이 이
  한 제약일 때만 이중결제로 가려낸다.

**핵심 근거 — InnoDB 동작(DB 엔진 동작 추론, 리포트에서 미검증 표시):** 같은 주문에 동시 두 승인이
들어오면 즉시 예외가 아니라, unique 충돌 레코드에 공유락(S-lock)을 걸고 *상대 트랜잭션 commit까지
대기*한다. 그 후 위반이 확정된다(상대가 롤백하면 이쪽은 성공). 즉시든 대기 후든 **위반이 adapter의
`saveAndFlush` 호출 안에서 드러나므로** 거기서 번역할 수 있다. 단순 INSERT라 gap lock으로 다른
주문(다른 키)까지 막지는 않는다 — 락 대신 unique 제약을 택한 이유와 일관된다.

**검토한 대안·트레이드오프:**
- **(i) adapter 매핑 (채택):** race 중복을 즉시 환불한다. 단 adapter에 도메인 지식이 약간 새고,
  `saveAndFlush`의 flush 타이밍에 의존한다.
- **(ii) 전파 + 배치 처리(#208):** 단순하고 flush 타이밍에 무관하다. 단 race 환불이 배치 주기만큼
  지연된다.
- → (i)을 택했다. 보상(환불)이 필요한 case라 즉시성이 가치 있기 때문이다.

- **주의(fragility) — `saveAndFlush`가 load-bearing:** 이 설계는 `saveAndFlush`의 즉시 flush에
  기댄다. plain `save`면 위반이 서비스 트랜잭션 경계의 commit 시점(adapter가 이미 return한 뒤)에
  터져 adapter가 못 잡는다. payment-naming-cleanup 회고도 이미 `saveAndFlush`를 이중결제 보상의
  load-bearing 요소로 경고했었다 — "명시적 save는 중복이다"라며 손대면 깨진다.

## 배운 것

- **정상 결제(맞는 결제)의 transient 기록 실패와 진짜 실패를 한 보상 status로 싸잡으면 안 된다.**
  검증을 통과한 결제는 환불이 아니라 REQUESTED로 두어 대사가 완료시키게 하고, 예상 못 한 예외는
  전파(500)로 드러낸다. "완료가 맞음/FAILED가 맞음/버그"는 서로 다른 처리다.
- **보상 순서(cancel-first vs fail-first)가 크래시 잔여 상태를 결정한다.** cancel을 먼저 만들면
  중간 크래시 시 "승인기록 REQUESTED + cancel 존재"라는 어느 정책도 안 가정한 잔여가 생긴다.
  실패-먼저면 그 잔여가 원천적으로 안 생긴다.
- **raw 인프라 예외(`DataIntegrityViolationException`) catch가 필요해 보여도, adapter에서 도메인
  예외로 번역하면 application의 계층 의존을 깨지 않고 예외 전략을 지킬 수 있다.** find-first가 못
  막는 race(NULL-트릭 부분 unique)는 이 carve-out으로 처리한다.

## 미해결·열린 질문

- 이중결제는 죽은 방어가 아니라 설계가 전제로 막는 실제 시나리오다: 첫 승인이 "진행 중"으로 매달린
  사이 사용자가 재시도하면 같은 주문에 서로 다른 두 캡처가 생길 수 있다. 그래서 unique 제약 +
  즉시 보상이 필요하다.
- (i)/환불→완료 모델/dead code 통일은 이슈 #225에서 ADR로 확정하고 구현한다. 그 전까지는 코드
  레벨 self-heal 안전망이 없다 — REQUESTED 잔여 회수는 대사 구현(#221/#208)에 의존한다.
