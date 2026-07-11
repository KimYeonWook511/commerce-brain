---
platform: backend
author: KimYeonWook511
created: 2026-06-18
origin:
  - { type: pr, repo: commerce-backend, ref: 262 }
---

# 결제-주문 결합 제거 — 승인 확정 조율을 provider 중립 facade로 모으고 거부 사유를 errorCode 단방향으로 전달

결제 대사·승인 코드가 주문 상태(`order.getStatus()`)를 직접 읽고 분기하던 결합을 제거한 세션이다(이슈 #240, PR #262). 주문 상태머신을 결제 쪽에서 대신 돌리다 보니, 주문 상태가 늘 때마다 결제 분기가 폭발하는 구조였다. 여러 도메인을 엮는 승인 확정 흐름(승인 사실 확정 + 거부 시 보상)을 provider 중립 조율 facade(`ConfirmApprovalUseCase` — tx를 열지 않고 흐름만 조립하는 payment.application 위치의 UseCase)로 모으고, 실시간 승인 진입점(`ApproveNaverPayUseCase`)과 대사 진입점(`ReconcilePaymentUseCase`)이 이 facade를 공유하게 했다. API·DB 스키마 변경은 없는 순수 내부 구조 리팩터다.

## 결정한 것

### 1. 거부 사유를 errorCode로 전달 — 결제→주문 단방향, 주문 상태 해석은 주문 메서드 안에 가둔다

주문 완료(`Order.completePayment()`)가 거부될 때, 기존에는 상태 불문 단일 예외(`ORDER_PAID_NOT_ALLOWED`)를 던지고 대사 쪽(`handleOrderNotCompletable`)이 주문을 재조회해 `order.getStatus()`로 4분기했다. 이걸 거부 사유별 errorCode로 세분화해, 주문이 INIT이 아닐 때 상태별로 다른 코드를 던지도록 바꿨다 — 이미 결제 완료면 `ORDER_ALREADY_PAID`, 취소된 주문이면 `ORDER_CANCELED_FOR_PAYMENT`, 그 외 비-INIT은 `ORDER_INVALID_STATE_FOR_PAYMENT`. facade는 주문을 재조회하지 않고 이 errorCode만으로 분기한다.

- **핵심 원칙 — Tell-Don't-Ask:** 주문 상태 해석을 주문 메서드 안에 가두고, 결제는 거부 결과(errorCode)만 받는다. 결제가 주문 상태머신을 알 필요가 없어진다. 실시간 승인이 이미 errorCode로 보상을 고르던 방식과도 통일됐다.
- **"이 결제가 중복인가"는 주문이 아니라 결제에게 묻는다:** 중복 여부 판별을 주문 상태가 아니라 payment 질문(`existsApprovedByOrderId` — 해당 주문에 이미 성공한 승인 결제가 있는지)으로 옮겼다.
- **주문 자체가 없는 경우:** `completePayment` 이전 단계의 주문 조회(락 조회)에서 `ORDER_NOT_FOUND`로 먼저 나온다. facade가 이 코드를 직접 받아 환불 없이 정합성 통지 + FAILED 종착으로 둔다(주문 미존재 시 통지하던 기존 대사 동작 보존).
- **검토한 대안 — enum 반환 + 식별자 컬럼:** 주문 완료가 예외 대신 결과 enum을 반환하고, 주문 테이블에 "어느 결제가 이 주문을 결제했는지" 식별자 컬럼을 추가하는 방식도 검토했다. DB 스키마 변경이 따라와 기각.
- **트레이드오프:** OrderErrorCode가 늘어난다. 그러나 주문 상태가 늘어도 facade 분기는 errorCode 단위로만 늘고, 결제가 주문을 재조회하는 결합 자체는 사라진다.

### 2. dead 코드 제거 + 금전 안전망 — PAID 성공-주체 확정 분기를 통지+fail로 대체

대사가 주문 완료 거부를 받았을 때 주문이 이미 PAID면, "이 결제가 성공 주체라면 결제 기록만 SUCCEEDED로 맞춘다"는 경로(`succeedApprovalRecordOnly` / `SucceedPaymentApprovalRecordService`)가 있었다. 이 경로는 이전 결정이 명시적으로 박아둔 것이었다 — 대사가 비-INIT 주문을 만나면 그냥 건너뛰지 않고 종착 상태로 전이시키되, 주문이 PAID인데 성공한 승인 결제가 없으면 "이 건이 성공 주체"로 보고 SUCCEEDED로 맞추라는 결정. 이번에 그 경로가 **도달 불가**임을 확인하고 제거했다(그 이전 결정을 supersede).

- **도달 불가 근거(코드 흐름 + DB 제약):** 주문이 PAID가 되는 유일한 경로는 승인 성공과 한 트랜잭션(`payment.succeed` + `saveApproved` + `order.completePayment`)이다. 그리고 "주문당 성공한 승인 결제 1개"를 강제하는 unique 제약(`uk_payment_approved_order_key`)이 두 번째 결제를 `completePayment` 도달 전에 `PAYMENT_DUPLICATE`로 막는다. 따라서 "주문은 PAID인데 성공한 승인 결제가 없음"이라는 성공-주체 분기의 진입 조건은 모순 — 절대 성립할 수 없다.
- **그래도 안전망은 남긴다:** 돈이 걸린 경로라 "증명됐으니 삭제"로 끝내지 않았다. 만에 하나 그 상태에 도달하더라도 성공 주체를 잘못 환불하는 사고를 막기 위해, 환불 대신 운영 통지(정합성 오류) + FAILED 종착으로 둔다. 금전 정합성은 희박한 경합에서도 안전하게 가는 쪽을 택했다.
- **PAID 거부의 실제 분기:** facade의 PAID 처리는 `existsApprovedByOrderId`로 실제 중복 여부를 판별해, 중복이면(다른 성공 결제 존재) 보상 환불, 비중복이면(위의 dead 조건) 환불 없이 통지+fail. 취소된 주문에 대한 보상 환불 등 기존 대사 동작은 facade가 그대로 보존한다.

### 3. provider 중립 facade는 "미래 구조의 절반"만 깐다 — gateway resolver는 후속

provider별 PG 게이트웨이 resolver, 공통 승인 진입 UseCase, PG 결과 정규화 레이어는 **이번에 만들지 않기로** 별도 결정으로 명시했다.

- **왜 안 만들었나:** 결제 provider가 네이버페이 하나뿐이다. 네이버페이 승인은 `ready→approve(redirect)`·`ALREADY_COMPLETE` 같은 특유의 상태머신을 갖고, 카카오/토스는 또 다르다. 정규화 경계는 두 번째 provider의 실제 모양을 봐야 제대로 그어진다 — 가상의 provider를 상상해 추상화하면 네이버페이 한 곳에만 맞는 틀린 경계가 나온다(YAGNI, "맥락이 달라지는 시점에 분리").
- **대신 절반은 미리 깐다:** provider 특화 진입점(`ApproveNaverPayUseCase`)은 PG 프로토콜 흐름을 담는 진입점으로 그대로 두되, 그 안의 provider 공통 "confirm" 로직만 facade로 추출했다. facade를 provider 중립 위치(payment.application)에 둠으로써, 두 번째 provider가 들어오면 같은 승인 확정 로직을 재사용할 토대만 마련한 셈이다.

### 4. facade 반환 결과(`Outcome`)를 sealed interface 대신 enum + nullable record로 표현

승인 확정 결과 타입 `Outcome`을 성공/거부/예외전파 세 갈래로 표현하는데, sealed interface가 아니라 `Decision` enum(`SUCCEEDED | REJECTED | PROPAGATE`) + nullable 필드(errorCode, cause)를 가진 record로 만들었다.

- **왜:** 코드베이스에 sealed interface 선례가 0건이고, "enum + nullable로 충분하면 그쪽을 쓴다(과한 추상화 지양)"는 컨벤션을 따랐다. 자동 도입 초안은 sealed interface였는데, 도구/리뷰어의 제안 방향과 이 레포의 표현 방식을 분리해 후자로 맞췄다.
- 정적 팩토리(`Outcome.succeeded()` / `rejected(code)` / `propagate(cause)`)로 세 결과를 만든다.

## 막힌 점·해결

- **record 전환으로 정적 팩토리가 package-private이 됐다.** interface였을 땐 메서드가 묵시적 public이었는데, record로 바꾸니 정적 팩토리가 package-private가 되어 다른 패키지의 테스트가 접근하지 못해 컴파일이 깨졌다. 팩토리에 `public`을 명시해 해결.
- **불필요한 errorCode 캐스팅 제거가 연쇄 타입 불일치를 냈다.** 예외에서 errorCode를 꺼낼 때 하던 강제 캐스팅(`(PaymentErrorCode) ex.getErrorCode()`)을 제거하니, 꺼낸 값의 타입이 상위 `ErrorCode`가 되어, 그 값을 다시 결과 생성(`Outcome.rejected(code)` — `PaymentErrorCode` 인자)에 넘기던 곳이 타입 불일치가 됐다. if 조건이 이미 그 값이 특정 상수임을 보장하므로, 꺼낸 변수 대신 `PaymentErrorCode` 상수를 직접 넘기도록 동반 수정했다(동작은 동일).

## 배운 것

- **"dead 코드"는 도달 불가를 코드 흐름 + DB 제약으로 증명한 뒤 제거한다.** 그리고 돈이 걸린 경로라면 증명이 됐어도 안전망(환불 대신 통지+실패)을 남긴다. 증명의 전제(unique 제약, 단일 tx 진입)가 나중에 깨질 수 있기 때문이다.
- **결합 제거는 분기를 facade로 "옮기는" 게 아니라 "제거"하는 것이다.** `order.getStatus()` 분기를 facade로 그대로 이동하면 결합은 그대로다. 상태 해석을 소유 도메인(주문) 안에 가두고 호출 측(결제)은 결과만 받게 만들어야 결합이 실제로 사라진다.

## 미해결·열린 질문

- **gateway resolver·공통 승인 진입 UseCase·PG 결과 정규화는 의도적으로 범위 밖.** 두 번째 결제 provider(카카오/토스 등) 도입 시 후속 작업으로 남겨뒀다. 그때 진입 UseCase 신설·gateway 추상화·결과 정규화가 함께 붙는다.
- **통합 경로는 이번에 확인되지 않았다.** Docker 기반 통합 테스트·배치 테스트는 내부 리팩터라 이번에 미실행으로 미뤘다. 순수 구조 리팩터라 단위 테스트로 커버했지만, 통합 경로가 실제로 도는지는 검증되지 않은 상태다.
