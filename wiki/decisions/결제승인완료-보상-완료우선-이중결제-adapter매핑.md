---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, payment, compensation, double-payment, reconciliation, adapter, unique-constraint, flush, idempotency, dead-code, transient-failure]
created: 2026-06-08
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-08-pr-224-payment-compensation-exception-findings]]"
  - "[[raw/sessions/backend/2026-06-08-pr-226-approval-compensation-exception-mapping]]"
---

# 결제 승인 완료 경로 보상·예외 처리 정리 — 완료 우선 + 이중결제 adapter 매핑

> [!note] 진화 (2026-08) — 이중결제 식별의 갈림 지점이 제약 이름 기반으로 정밀해졌다
> 아래의 "이중결제를 adapter에서 매핑한다"는 배치는 유효하고, 그 위에 둘이 더해졌다.
> - **무결성 위반을 타입이 아니라 제약 이름으로 가른다.** 한 트랜잭션이 네 테이블을 저장하게 되면서 유일 제약 위반과 NOT NULL·외래 키 위반이 같은 타입으로 섞여 들어오게 됐고, "이 제약에 부딪히면 뜻이 하나"라는 전제가 깨졌다: [[무결성위반-도메인예외-번역을-제약이름으로-가른다]].
> - **실측으로 확인된 것 하나** — 이 구성에서는 중복 키 전용 예외(`DuplicateKeyException`)가 **아예 생성되지 않는다.** 하위 타입으로 좁히면 "좁힌다"가 아니라 "아무것도 안 잡는다"가 된다: [[예외타입-실측과-테스트가-명세를-넘어-단언하는-것]].
> - 아래가 기대는 즉시 flush 계약은 그대로이고, 그 계약을 메서드 이름으로 드러내는 방식이 [[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]에서 셋으로 갈렸다.

무대는 `NaverPayApprovalService.completeVerifiedApproval` — NaverPay PG가 승인 SUCCESS(=캡처 완료)를 준 뒤, 응답의 키·금액이 우리 결제와 일치하는지 검증을 통과하면 결제행을 SUCCEEDED로, 주문을 PAID로 반영하는 메서드. 리뷰 중 이 경로의 두 부채를 **발견(pr-224, #224)**하고 후속 이슈 #225로 넘겨 **확정·구현(pr-226)**했다.

## 컨텍스트·발견 — 정상 결제 환불 버그 + 갈라진 이중결제 보상(dead code)

### 발견 1 (가장 심각) — 정상 결제를 환불하고 있었다

승인기록 확정 전이(`succeedApproval`)가 일시적 실패(DB 데드락 등)로 예외를 던지면, catch가 `compensateUnexpected`로 보내 승인기록을 `FAILED(APPROVE_PROCESS_FAILED)`로 마킹하고 PG 환불까지 했다. 그런데 이 지점은 이미 키·금액 검증을 통과한 *맞는 결제*다 — "틀린 결제"가 아니라 "맞는 결제인데 DB 기록만 실패"한 상태다. 이걸 환불하면 transient 실패에 정당한 매출을 취소해버린다. `compensateUnexpected`는 "완료가 맞음 / FAILED가 맞음 / 진짜 버그"를 한 status로 싸잡고 있었다.

### 발견 2 — 예외 전략 위반 + dead code + 갈라진 이중결제 보상

- **예외 전략 위반:** 이 도메인의 예외 전략은 "정상 중복은 사전조회([[find-first-write-not-check-db-unique-멱등]])로 흡수하고 race 위반은 안전망 500에 위임, DB 무결성 위반을 application이 직접 catch하지 않는다"였다. 그런데 `completeVerifiedApproval`은 그 전략 확립 *이후* 이중결제 환불을 위해 다시 `catch (DataIntegrityViolationException)`을 들여왔다 — 정면 충돌.
- **이중결제 보상이 두 갈래로 갈라져 있었다:** 실제로 타는 `compensateDuplicateApproval`(cancel-first: cancel 기록을 먼저, 승인기록 실패 마킹은 나중)은 크래시 시 "승인기록 REQUESTED인데 cancel 존재"라는 어느 정책도 안 가정한 잔여를 남긴다. dead 쪽 `case PAYMENT_DUPLICATE → compensateDuplicatePayment`(fail-first)는 `DataIntegrityViolationException` catch가 `PAYMENT_DUPLICATE`를 곧장 밖으로 던져 `PaymentException` catch의 분기에 도달할 경로가 없어 죽어 있었다. **객관적으로 dead 쪽(fail-first)이 낫다** — 위 잔여를 애초에 안 만든다.

## 결정 1 — 완료 우선: transient 기록 실패는 REQUESTED, 환불 안 함

정상 승인(PG 캡처 성공 + 키·금액 검증 통과) 뒤 DB 기록만 transient 실패하면, 보상 없이 예외를 그대로 전파(500)하고 결제행을 REQUESTED로 남긴다.

- **상태 모델:** SUCCEEDED(완료 확정) / FAILED(실패 확정) / REQUESTED(아직 미확정 흔적) 셋. 예전엔 이 셋을 FAILED 하나로 싸잡았다. 완료 우선은 실시간 경로 책임을 "완료시키거나, 아니면 REQUESTED 흔적 남기기"까지로 좁히고, 결과 확정·복구는 배치 대사(reconcile)가 PG 재확인으로 완료시키게 미룬다.
- **왜:** 금전 정합이 우선. PG가 이미 캡처 성공하고 키·금액 검증까지 통과한 *맞는 결제*를 DB 기록이 잠깐 미끄러졌다는 이유로 환불해 정상 매출을 취소하고 싶지 않다. 경합 확률이 낮아도 돈 문제는 안전 우선.
- **구현:** `completeVerifiedApproval`에서 unmapped 예외에 대한 `compensateUnexpected` 호출을 제거하고, `PaymentException` default 분기·`CustomException`·일반 `Exception` catch에서 모두 보상 없이 rethrow. 쓰이지 않게 된 `compensateUnexpected` 메서드 자체도 삭제.

## 명시적 비정상(키/금액 불일치)은 환불 유지, 진짜 버그는 500 전파

응답 키 불일치(`MERCHANT_KEY_MISMATCH`)·금액 불일치(`AMOUNT_MISMATCH`)는 *틀린 결제*라 판별이 확실하므로 기존대로 환불(보상)을 유지한다([[payment-amount-mismatch-이중검증-409-vs-400-분리]]). 완료 우선은 "정상인데 기록만 미끄러진" transient 실패에만 적용된다. 진짜 예상 못 한 버그도 500으로 가시화(전파)한다 — "완료가 맞음/FAILED가 맞음/버그"는 서로 다른 처리다.

## 결정 2 — 이중결제 탐지를 adapter 도메인 예외 매핑으로 이관 (예외 전략 carve-out)

이중결제(같은 주문에 성공 결제 둘 이상) 탐지를, application이 `DataIntegrityViolationException`을 직접 catch하던 것에서 repository adapter가 도메인 예외로 번역해 올려주는 것으로 바꿨다.

- **바뀐 구조:** adapter(`PaymentRepositoryAdapter`)에 승인 완료 저장 전용 메서드 `saveApproved`를 두고, "한 주문에 성공 결제 하나"를 강제하는 unique 제약 `uk_payment_approved_order_key` 위반을 만나면 `PaymentException(PAYMENT_DUPLICATE)`로 번역, 그 외 무결성 위반은 원 예외 전파. application(`NaverPayApprovalService`)에서는 raw `catch(DataIntegrityViolationException)`를 없애고(import도 제거), 살아난 `case PAYMENT_DUPLICATE` 분기가 fail-first 보상(`compensateDuplicatePayment`)을 수행. 저장 호출부(`succeedApproval`)도 일반 `save`→`saveApproved`로 교체.
- **왜 adapter가 맞나:** unique 제약은 NULL-트릭 부분 unique(`approved_order_key`가 APPROVE+SUCCEEDED 전이 때만 채워짐)라 사전 `find`와 insert 사이 race를 find-first로 못 막는다([[payment-이중결제-reserve따닥-mysql-null트릭-unique]]). 그래서 위반을 실제로 처리하는 지점이 어딘가엔 있어야 하는데, 예외 전략은 "도메인 의미가 분명한 제약 위반을 adapter에서 도메인 예외로 번역하는" try-save-catch를 좁은 carve-out으로 허용한다. 이중결제는 보상(환불)이 필요한 case라 find-first+500만으로는 부족하고 이 carve-out에 정확히 들어맞는다. 예전 application의 raw DAO catch는 이 정책을 위반하는 부채였다.

## saveAndFlush가 load-bearing + fail-first 통일 + 매핑 범위 한정(제약명)

- **`saveAndFlush`가 load-bearing:** `saveApproved`는 내부에서 `saveAndFlush`(즉시 flush)를 쓴다. 이 조기 flush가 unique 위반을 그 메서드 호출 *안에서* 확정하는 게 매핑 성립의 핵심 의존성. 일반 `save`로 바꾸거나 flush를 커밋까지 미루면 위반이 adapter catch 밖(서비스 tx 경계 commit 시점)에서 터져 매핑이 깨진다. `payment-naming-cleanup` 회고도 "명시적 save는 중복이다"라며 손대면 깨진다고 이미 경고했었다.
- **fail-first 단일 경로로 통일:** cancel-first(`compensateDuplicateApproval`) 경로를 제거하고 fail-first(`compensateDuplicatePayment`)로 통일해, "승인기록 REQUESTED인데 cancel 존재" 잔여를 원천 제거.
- **오매핑 방지:** 매핑을 `saveApproved` 전용 경로 + `uk_payment_approved_order_key` 제약명 일치로 한정해, FK·NOT NULL·다른 unique 위반이 `PAYMENT_DUPLICATE`로 오매핑되지 않게 했다. 범용 `save`는 매핑 안 함. (이 adapter 경계·flush 타이밍의 일반화는 [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]].)

## 검토한 대안 — adapter 매핑(즉시성) vs 배치 전파(#208)

- **(i) adapter 매핑 (채택):** race 중복을 즉시 환불한다. 단 adapter에 도메인 지식이 약간 새고 `saveAndFlush` flush 타이밍에 의존.
- **(ii) 전파 + 배치 처리(#208):** 단순하고 flush 타이밍 무관하나 race 환불이 배치 주기만큼 지연.
- → 보상(환불)이 필요한 case라 즉시성이 가치 있어 (i) 채택.

**InnoDB 동작 근거(DB 엔진 동작 추론, 미검증 표시):** 같은 주문에 동시 두 승인이 들어오면 즉시 예외가 아니라 충돌 레코드에 공유락(S-lock)을 걸고 상대 tx commit까지 대기한 뒤 위반 확정(상대 롤백 시 이쪽 성공). 즉시든 대기 후든 위반이 `saveAndFlush` 안에서 드러나므로 거기서 번역 가능. 단순 INSERT라 gap lock으로 다른 주문까지 막지 않는다 — 락 대신 unique 제약을 택한 이유와 일관.

## 한계·부채 — reconcile 미구현으로 REQUESTED 회수 주체 없음

완료 우선 모델은 REQUESTED 잔여를 회수해줄 배치 대사에 전적으로 의존한다. 대사는 `APPROVE_RECONCILE` 대상을 골라 PG에서 승인 확정(`PG_APPROVED`)을 재확인한 뒤 완료시키는 흐름인데 그 구현(이슈 #221 / Epic #208)이 아직 없다. **그 전까지는 코드 레벨 self-heal 안전망이 없다** — 지금 REQUESTED로 남는 결제는 회수 주체가 없는 상태로, 감수하고 진행한 부채다. (대사 설계는 [[결제-후처리-대상식별-status중심-재설계]].)

이중결제는 죽은 방어가 아니라 설계가 전제로 막는 실제 시나리오다: 첫 승인이 "진행 중"으로 매달린 사이 사용자가 재시도하면 같은 주문에 서로 다른 두 캡처가 생길 수 있다.

## 후속

`getConstraintName()` 폴백이 이 프로젝트에서 dead 경로임이 통합 테스트에서 드러났고(전역 `SQLErrorCodeSQLExceptionTranslator` 빈이 cause 체인을 바꿔서), 실제 매핑은 SQLException 메시지 정규식이 담당하고 있었다. 그 translator 빈 제거는 영향 범위가 다른 결정이라 별도 이슈 #227로 분리했다([[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]]) → [[sql-translator-빈-제거-제약명-이중결제-식별]]. 같은 결제 UNKNOWN/보상/reconcile 부채는 [[pg-승인-예외-경계-요청전송시점]]. 이 세션의 AI 코드리뷰·작업 범위 운영 교훈은 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]으로 분리했다.

## 근거

- [[raw/sessions/backend/2026-06-08-pr-224-payment-compensation-exception-findings]]
- [[raw/sessions/backend/2026-06-08-pr-226-approval-compensation-exception-mapping]]
