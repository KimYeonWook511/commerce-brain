---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 224 }
---

## 한 일

- 위 postprocess 정책(#221) 코드 리뷰 중, 결제 *실시간 보상/예외 처리*의 더 깊은 부채를 발견해 후속 이슈 #225로 정리했다. 이 메모는 그 발견과 결정.
- 무대: `NaverPayApprovalService.completeVerifiedApproval` — PG가 승인 SUCCESS(= 돈 캡처 완료)를 준 뒤, 우리 DB에 `succeed`(승인기록을 SUCCEEDED 상태로 확정하는 전이) + 주문 PAID를 반영하는 메서드. 그 안의 catch 흐름이 문제였다.

## 발견 1 — 정상 결제를 환불하고 있었다

- `succeed` 반영이 일시적 실패(DB 데드락 등)로 던지면, catch가 `compensateUnexpected`(예상 못 한 예외를 한 묶음으로 받아 승인기록을 실패 마킹하고 PG 환불까지 하는 보상 경로)로 보낸다.
- 그런데 이 지점은 이미 키·금액 검증을 통과한 *맞는 결제*다. 즉 "틀린 결제"가 아니라 "맞는 결제인데 우리 DB 기록만 실패". 이걸 환불하면 transient 실패에도 정당한 매출을 취소한다.
- **올바른 모델: 완료 재시도.** 승인기록을 실패로 마킹하지 말고 REQUESTED로 두면, 위 postprocess 대사가 PG 재확인 → APPROVED 확인 → 완료로 self-heal한다. 모르는 예외는 한 status로 뭉뚱그리지 말고 전파(500)해서 드러낸다.

## 발견 2 — 예외 전략 위반 + dead code + 갈라진 이중결제 보상

- 이 도메인의 예외 전략은 "DB 무결성 위반(unique 등)을 app/adapter가 직접 catch하지 말고, 정상 중복은 사전조회(find-first)로 흡수하고 race 위반은 안전망 500에 위임"이다 (정본: `docs/adr.md`, `docs/exception-strategy.md`).
- 그런데 `completeVerifiedApproval`은 **`DataIntegrityViolationException`(DB 무결성 위반 예외)을 application에서 직접 catch**한다. 이중결제 환불을 위해 그 전략 확립 *이후* 다시 들여온 것이라 충돌.
- 게다가 이중결제 보상이 두 갈래로 갈라져 있었다:
  - 쓰이는 쪽 `compensateDuplicateApproval`: cancel 기록을 *먼저* 만들고 승인기록 실패는 *나중에*. 이 순서 때문에 크래시 시 "승인기록 REQUESTED인데 cancel 존재"라는, 정책이 가정 안 한 잔여가 남는다.
  - 안 쓰이는 dead 쪽 `case PAYMENT_DUPLICATE` → `compensateDuplicatePayment`: 승인기록 실패를 *먼저*(다른 보상들과 같은 순서). 호출되는 경로가 없어 죽어 있었다.
  - 객관적으로 dead 쪽이 낫다 — 실패-먼저라 위 잔여 상태("승인기록 REQUESTED인데 cancel 존재")를 만들지 않는다.

## 결정 — 이중결제 탐지를 adapter 도메인 예외 매핑으로 (→ #225)

- `uk_payment_approved_order_key`(한 주문에 SUCCEEDED 승인 1건만 허용하는 NULL-트릭 부분 unique 제약)는 동시 race를 사전조회로 못 막는다 → unique 위반 처리가 어딘가엔 반드시 필요.
- 현행: application이 raw `DataIntegrityViolationException`을 직접 catch (위 위반).
- **#225 결정: adapter(`PaymentRepositoryAdapter`)가 `saveAndFlush`(저장 즉시 flush)의 unique 위반을 catch해 도메인 예외(`PaymentException(PAYMENT_DUPLICATE)`)로 번역**, application은 도메인 예외로 반응(dead `case PAYMENT_DUPLICATE`가 live화). → application의 raw DB예외 의존이 사라지고, 예외 전략의 "try-save-catch를 쓰더라도 인프라 예외 타입에 직접 의존 말고 adapter에서 처리한다"는 예외조항에 부합.
  - InnoDB 동작 확인(핵심 근거): 동시 두 승인(같은 주문)이면 즉시 예외가 아니라 unique 충돌 레코드에 공유락(S-lock) 걸고 *상대 commit까지 대기* → 그 후 위반 확정(상대 롤백 시엔 성공). 즉시든 대기 후든 **위반이 adapter의 saveAndFlush 호출 안에서 드러나므로** 거기서 번역 가능. 단순 INSERT라 gap lock으로 다른 주문(다른 키)은 안 막는다(= lock 대신 unique를 택한 이유와 일관).
  - 매핑은 *승인-성공 저장 전용 경로*로 한정한다(generic save에 걸면 FK/NOT NULL/타 unique 위반까지 PAYMENT_DUPLICATE로 오매핑).
  - 이러면 실패-먼저 보상(`compensateDuplicatePayment`)이 live가 되고 cancel-먼저 보상(`compensateDuplicateApproval`)이 제거돼, 위 "승인기록 REQUESTED인데 cancel 존재" 잔여까지 자동 해소된다.
- 트레이드오프: (i) adapter 매핑 = race 중복을 즉시 환불, 단 adapter에 도메인 지식이 약간 새고 saveAndFlush에 의존. (ii) 전파+배치(#208) = 단순하고 flush 타이밍에 무관, 단 race 환불이 배치 주기만큼 지연. → (i) 채택(보상이 필요한 case라 즉시성이 가치 있음).
- **주의(fragility): 이 설계는 `saveAndFlush`(즉시 flush)가 load-bearing.** plain save면 위반이 commit(서비스 트랜잭션 경계, adapter가 이미 return한 뒤)에서 터져 adapter가 못 잡는다. payment-naming-cleanup 회고도 saveAndFlush를 이중결제 보상의 load-bearing 요소로 경고했었다 — "명시적 save는 중복"이라고 손대면 깨진다.

## 다음 단계

- #225에서 (i)/환불→완료/dead code 통일을 ADR로 확정·구현.
- 이중결제는 실제 발생 가능하다(첫 승인이 진행 중으로 매달린 사이 사용자가 재시도 → 같은 주문에 서로 다른 두 캡처). 죽은 방어가 아니라 설계가 전제로 막는 시나리오다.

같은 PR의 도메인 정책 메모: [[raw/sessions/backend/2026-06-08-pr-224-postprocess-policy-unknown-redesign]].
