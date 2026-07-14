---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, naming, refactor, jpa, enum, save-flush, code-review]
created: 2026-06-05
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]]"
---

# 결제 도메인 attempt 네이밍 잔재 정리 — 옛 엔티티 잔재만, 진짜 '시도'는 보존

## 제약 — 순수 refactor(동작·API·DB 불변)

결제 도메인 재설계(PR #205)에서 엔티티 `PaymentAttempt`를 `Payment`로 통합했는데, 변수·repo 메서드·서비스 클래스·에러코드 식별자엔 옛 `attempt` 네이밍이 그대로 남아 있었다. 그 잔재를 걷어내고 상태 반영 메서드 이름 패턴을 정리한 세션(#209 → PR #210)이다. 핵심 제약은 "순수 refactor — 외부 동작·API·DB 스키마는 일절 안 건드린다"였고, 그래서 어디까지가 정리 대상이고 어디부터가 보존인지 경계를 긋는 게 작업의 대부분이었다. PaymentAttempt→Payment 통합 자체는 [[payment-reserve-예약테이블-분리-a안-b안]]에서 일어났다.

## mark 계열 동사화(멀쩡한 동사는 보존)

예약 엔티티 `PaymentReservation`의 `markUsed`→`use`, `markExpired`→`expire`로 축약했다. `succeed`/`fail`은 이미 동사라 그대로, `markUnknown`은 "결과 불명"에 마땅한 동사가 없어 `mark` 접두사를 정직한 표현으로 유지했다. 반대로 `succeed`/`fail`까지 `markSucceeded`/`markFailed`로 통일하는 안은 기각 — 멀쩡한 동사를 mark로 끌어내리면 어색하고 churn만 커진다. 어색한 쪽만 동사로 끌어올리는 게 churn도 적고 도메인 행위 중심 네이밍 원칙과 일관된다. 이 `use`/`expire` 메서드는 예약 재사용/만료 흐름([[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]])의 도메인 동작이다.

## attempt 식별자 제거 + 서비스 rename(이름 충돌 회피 Record)

`Payment` 타입(=옛 `PaymentAttempt`)을 가리키는 변수·파라미터·필드·repo 메서드·처리 메서드에서 `attempt`를 걷어냈다(예: `approveAttempt`→`approvePayment`, `findApproveAttempt`→`findApprovePayment`, `processApproveAttempt`→`processApprovePayment`).

- **서비스 rename:** `PaymentApprovalAttemptService`→`PaymentApprovalRecordService`, `PaymentCancellationAttemptService`→`PaymentCancellationService`. 취소 쪽은 이름 충돌이 없어 깔끔하게 attempt만 뗐다. 승인 쪽을 그냥 `PaymentApprovalService`로 줄이면 **이미 존재하는** 오케스트레이션 서비스(승인 성공 + 주문 완료를 한 트랜잭션으로 묶음)와 충돌해, "승인 시도 기록"을 담당한다는 뜻으로 `Record`를 넣어 구분했다.
- **에러코드는 식별자만 rename:** enum 상수명(`PAYMENT_ATTEMPT_NOT_FOUND`→`PAYMENT_RECORD_NOT_FOUND` 등)만 바꾸고, code 문자열(`PAYMENT-404-2` 등)과 한국어 메시지는 **외부 계약**이라 손대지 않았다.

## 정리 경계 — 옛 엔티티 잔재 vs 진짜 '시도'

**판단 기준:** 식별자가 `Payment` 엔티티(=옛 `PaymentAttempt`)를 가리키면 정리 대상, 행위로서의 "시도(try)"를 뜻하면 보존.

- **보존한 것:** 보안 테스트 공격 시도 `attackerAttempt`, 동시성/재시도 맥락 concurrent·retry attempt, 한국어 "시도"(에러 메시지·`@DisplayName`), test 전용 `postprocess` 패키지, 그리고 역사 기록(레거시 단일 ADR 서술, 이미 머지된 task 폴더, flyway migration 파일, outbox 테이블의 `attempt_count` 컬럼). 역사 기록은 결정 당시 상태(`PaymentAttempt`)를 기록한 것이라 소급 수정하지 않는다([[backend-완료된-task-문서-불변-원칙]]).
- `s/attempt/payment/g` 전역 치환이었다면 의미를 훼손했을 것이라, 각 식별자를 case-by-case로 판단했다. "의미가 두 갈래인 단어는 어원 인접 식별자를 먼저 목록화한 뒤 하나씩 판단"이 교훈.

## status enum 값 제외(@Enumerated STRING = DB값)

`PaymentStatus`(`REQUESTED`/`SUCCEEDED`/`FAILED`/`UNKNOWN`)를 `SUCCESS`/`REQUEST` 형태로 바꿀지 논의했으나 **안 바꾸기로** 했다.

- **제약 위반:** 이 enum은 `@Enumerated(EnumType.STRING)`이라 상수 이름이 곧 DB 저장 문자열이다. rename하면 저장값과 불일치가 생겨 데이터 마이그레이션이 반드시 따라붙어 "DB 변화 없음" 제약을 깬다.
- **의미:** status enum은 "지금 어떤 상태인가"라 과거분사/형용사형이 정석이다. `REQUESTED`/`SUCCEEDED`는 이미 균일하고 옳다. 동사형(`REQUEST`)은 행위라 부적절, 명사형(`SUCCESS`/`FAILURE`)은 `UNKNOWN`에서 균질성이 깨진다. PaymentStatus enum 값 자체는 [[payment-append-only-원장과-exists-완료판단]] 맥락. rename은 DB 마이그레이션과 짝지어 별도 task로만 가능.

## dead code 제거 + 명시 save 통일

- `Payment.succeed()` 안의 `failCode=null; failDetail=null;` 두 줄을 제거했다. `succeed()`는 진입 가드로 `REQUESTED`에서만 호출되고 그 상태 결제는 `failCode`가 설정될 경로가 애초에 없어 증명 가능한 no-op dead code다. `PaymentTest`의 "succeed 후 failCode가 null" 단언은 리셋 동작이 아니라 "성공 결제엔 failCode 없다"는 불변식 검증이라 리셋 코드를 지워도 통과 → 단언은 보존했다.
- **명시 save() 4곳 추가(리뷰 수용):** AI 코드 리뷰(Gemini)가 상태 변경 후 `save()`를 명시 호출하지 않던 4곳(승인 기록 서비스 `fail`·`failIfRequested`, 취소 서비스 `succeed`·`fail`)에 추가를 제안해 각각 별도 커밋으로 수용했다. 근거 컨벤션은 "응용 계층은 dirty checking에 묵시적으로 기대지 말고 `repository.save(entity)`를 명시 호출". 엄밀히는 그 컨벤션이 "신설 서비스에만 적용"이라 이 기존 서비스들은 범위 밖이었으나, 같은 서비스 안 불일치(`create`는 명시 save, `fail`류는 안 함)가 실재해 일관성으로 수용했다. 이 4곳은 flush 타이밍이 load-bearing이 아니라(NULL 트릭 unique에 무관) 안전하다. AI 리뷰 제안 수용 맥락은 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]].

## save/saveAndFlush load-bearing 보존

승인 오케스트레이션에서 결제 행 성공 전이 후의 명시적 `save()`는 단순 관성 코드가 아니라 **load-bearing**이라 제거 금지다. `PaymentRepositoryAdapter.save()`는 내부적으로 `saveAndFlush`(즉시 flush)를 쓴다. "한 주문에 성공 승인 1건"을 강제하는 unique `uk_payment_approved_order_key`의 대상 컬럼(`approvedOrderKey`)은 APPROVE+SUCCEEDED 전이 때만 orderId로 채워지고 그 외 NULL이라(NULL 중복 허용), 이 즉시 flush 덕에 제약 위반이 **트랜잭션 커밋 전, 승인 흐름의 try-catch 안에서** 확정된다. 그 `DataIntegrityViolationException`을 같은 tx catch에서 잡아 이중결제 보상을 트리거한다. flush를 커밋 시점으로 미루면 위반이 try-catch 밖에서 터져 보상 catch를 빠져나가므로 이 명시 save는 제거 금지. 제약 위반 catch → 보상의 전체 메커니즘은 [[payment-이중결제-reserve따닥-mysql-null트릭-unique]].

## 미해결 — verb/noun 컨벤션·enum rename 후속

- **서비스 클래스명 verb/noun 컨벤션 전면 정리** — 별도 후속 이슈로 분리 예정(미생성). 이번에 만든 `PaymentApprovalRecordService` 등도 확정 후 재검토 대상. 코드베이스는 단일 유스케이스=동사형·개념 묶음=명사형이 혼재하고 강제 컨벤션은 없다.
- **`PaymentStatus`·`PaymentReservationStatus` enum 값 rename** — DB 마이그레이션과 짝을 이뤄 별도 task로만.
- test 전용 `postprocess` 패키지 attempt 잔재 정비 — 배치 도입 시 일괄.

## 근거

- [[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]]
