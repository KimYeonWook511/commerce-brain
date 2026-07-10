---
platform: backend
author: KimYeonWook511
created: 2026-06-18
origin:
  - { type: pr, repo: commerce-backend, ref: 262 }
---

## 한 일
- 이슈 #240 결제-주문 결합 제거. 결제 대사·승인 코드가 주문 상태(`order.getStatus()`)를 직접 읽고
  분기하던 걸, provider 중립 조율 facade(`ConfirmApprovalUseCase`)로 모았다. PR #262.
  정본: docs/tasks/payment-order-decouple/adr.md, architecture.md, retrospective.md

## 결정한 것
- **거부 사유를 errorCode로 전달(결제→주문 단방향)**: 주문 완료(`Order.completePayment`) 거부를 단일
  예외 대신 사유별 errorCode(이미 PAID / 취소됨 / 그 외 비-INIT)로 세분화. `ConfirmApprovalUseCase`가
  주문을 재조회하지 않고 errorCode로 분기한다. "주문 상태 해석은 주문 메서드 안에 가둔다"(Tell-Don't-Ask).
  검토한 대안은 주문 완료가 결과 enum을 반환하고 주문에 "어느 결제가 결제했는지" 식별자 컬럼을 추가하는
  방식이었는데, DB 스키마 변경이 따라와 안 택함.
- **dead 코드 제거 + 안전망**: 대사가 주문 완료 거부를 받았을 때 주문이 PAID면 "이 결제가 성공 주체면
  결제 기록만 SUCCEEDED로 맞춤" 경로가 있었는데, 도달 불가임을 확인했다(이전 결정 ADR-048을 supersede —
  그 ADR은 이 경로를 "성공 주체면 SUCCEEDED로 맞춤"으로 박아뒀었다). 근거 — 주문이 PAID가 되는 유일한
  경로는 결제 성공과 한 트랜잭션이고, "주문당 성공한 승인 결제 1개"를 강제하는 unique 제약
  (`uk_payment_approved_order_key`)이 두 번째 결제를 그 전에 막는다. 따라서 "주문은 PAID인데 성공한 승인
  결제가 없음"은 모순. 이 dead 경로를 제거하되, 만에 하나 도달하면 성공 주체를 잘못 환불하지 않도록 환불
  대신 운영 통지 + 실패 종착으로 둔다(금전 안전).
- **provider 중립 facade를 미래 구조의 절반으로**: provider별 PG 게이트웨이 resolver·공통 승인 진입
  UseCase는 안 만들었다. 결제 provider가 네이버페이 하나뿐이라 가상의 provider를 상상해 추상화하면 틀린
  경계가 나온다. `ConfirmApprovalUseCase`를 provider 중립 위치(payment.application)에 둬서, 2번째
  provider가 들어오면 같은 승인 확정(confirm) 로직을 재사용할 토대만 깔았다.
- **Outcome 표현**: facade 반환 결과 타입(`Outcome`)을 sealed interface 대신 enum + nullable 필드를 가진
  record로 표현. 코드베이스에 sealed interface 선례가 0건이고 "과한 추상화 지양" 컨벤션이라 그쪽을 따랐다.

## 막힌 점
- 결과 타입(`Outcome`)을 record로 바꾸니 static 팩토리 메서드가 package-private이 됐다(interface일 땐
  묵시적 public이었음). 다른 패키지의 테스트가 못 써서 컴파일이 깨졌고 `public` 명시로 해결.
- 예외에서 errorCode를 꺼낼 때 강제 캐스팅(`(PaymentErrorCode) ex.getErrorCode()`)을 제거하니, 그
  errorCode를 다시 결과 생성(`Outcome.rejected(code)`)에 넘기던 곳이 타입 불일치가 됐다. if 조건이 이미
  그 값을 보장하므로 PaymentErrorCode 상수를 직접 넘기도록 동반 수정(동작 동일).

## 배운 것
- "dead 코드"는 도달 불가를 코드 흐름 + DB 제약으로 증명한 뒤 제거한다. 돈이 걸린 경로면 증명됐어도
  안전망(환불 대신 통지+실패)을 남긴다.
- 결합 제거는 분기를 facade로 "옮기는" 게 아니라 "제거"하는 것. 상태 해석을 소유 도메인 안에 가두고
  호출 측은 결과만 받는다.

## 다음 단계
- provider별 게이트웨이 resolver·공통 승인 진입 UseCase·PG 결과 정규화: 두 번째 결제 provider
  (카카오/토스 등) 도입 시(루트 ADR-065로 후속 명시). 정본 docs/tasks/payment-order-decouple/adr.md
- Docker 기반 통합 테스트(integrationTest)·배치 테스트는 이번에 미실행(내부 리팩터라 미뤘으나 통합 경로
  확인은 안 된 상태)
