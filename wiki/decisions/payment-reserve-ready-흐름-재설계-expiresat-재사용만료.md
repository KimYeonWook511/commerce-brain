---
type: decision
status: superseded
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, reservation, idempotency, expiration, redirect, api-naming]
created: 2026-06-04
updated: 2026-08-17
superseded_by: "[[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]"
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]"
  - "[[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]]"
---

# reserve(ready) 흐름 재설계 — RESERVED 예약 행 생성·재사용·만료

> [!warning] 뒤집힘 (2026-08) — 예약 행이 없어지고 활성 슬롯 빼앗기로 대체됐다
> 예약 테이블이 폐지되면서([[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]) 아래의 재사용·만료·lazy 회수 흐름 전체가 **"같은 주문으로 다시 요청이 오면 직전 결제 행을 종결하고 슬롯을 넘겨받는다"**로 바뀌었다. 넘겨받을 수 있는 대상은 **아직 승인을 부르지 않은 행뿐**이며 그 경계의 근거는 [[배타점유-슬롯-미리잡기-vs-성공시-감지·되돌리기]].
> 살아남은 것은 두 가지다 — **박제된 점유를 시간·상태로 자동 복구해야 한다**는 문제 인식, 그리고 **"남의 것이면 없음으로 답한다"는 신원 확인**이 결제 행으로 이관됐다는 사실.

결제창 준비 단계(ready = reserve)를 재설계한 결정이다. 재설계 동기와 도메인 분리는 허브 [[payment-order-도메인분리와-pg격리]] 참조.

## 기존 ready의 3가지 문제

재설계 전 `PaymentReadyService.readyPayment()`(확인함)에는 세 문제가 있었다.

1. merchantPayKey를 **Order에 저장**(값 없을 때만 발급)한다 → 키 고정·이력 소실·provider 미반영.
2. ready가 **RESERVED 행을 만들지 않고 키만 반환**한다 → redirect 역조회·따닥 재사용·만료를 할 수가 없다.
3. expiresAt/provider 기반 **재사용·만료 로직이 없다** → 한번 박힌 키가 영원히 재사용된다.

## 결정 — RESERVED 행 생성/재사용/반환

ready를 "키 발급 + 반환"에서 **"RESERVED 예약 행 생성/재사용 + 결제창 정보 반환"**으로 바꾼다.

- 서버가 merchantPayKey를 발급(`"PAY-" + ULID`)해 RESERVED 예약 행에 둔다. 발급 주체가 서버인 이유: 유일성을 서버가 통제(DB unique)해야 하고, 시도를 추적하는 신분증·재무 식별자라 클라이언트를 신뢰할 수 없으며, reserve 때 네이버에 보내는 값이기 때문이다.
- 반환은 **결제창을 여는 최소 정보**만(clientId/chainId/merchantPayKey/금액/상품명/returnUrl 등, 반환 타입 `PaymentReadyResult`). PG 시크릿·내부 id는 금지.
- redirect로 돌아온 값은 신뢰하지 않고 승인 시 금액·키를 재검증한다(`verifyApprovedResponse`). 금액 대조·검증 계층은 [[payment-amount-mismatch-이중검증-409-vs-400-분리]].

동시 따닥의 두 번째 INSERT는 예약 테이블의 `reserved_key` unique가 차단한다 — [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]. reserve 따닥에 한해 클라이언트 idempotencyKey(Redis+DB)보다 order 단위 unique가 더 단순해(키 발급·전달 없이 DB가 "주문당 진행중 reserve 1개" 강제) 이쪽을 택했다.

## 재사용 조건(expiresAt·provider)과 박제 자동복구

재사용 조건은 셋을 모두 본다.

```text
유효한 RESERVED = status=RESERVED ∧ expiresAt>now ∧ provider = 요청 provider
```

- **expiresAt(시간)을 판단에 넣는 이유 — 박제(stale RESERVED) 자동복구.** 진행 중 reserve를 status만으로 재사용하면, SUCCESS 전환 중 DB 장애 등으로 죽은 RESERVED가 남고 "유효한 RESERVED 있음"으로 판단해 죽은 키를 계속 반환한다(답이 없음). 판단에 시간을 넣으면 박제된 행은 만료되는 순간 재사용 대상에서 빠지고 다음 요청이 새로 발급한다 → 자동 복구. 재사용은 반드시 expiresAt과 함께.
- **provider까지 보는 이유.** RESERVED엔 provider가 들어있다. "유효하면 무조건 반환"하면 네이버로 띄웠다 카카오를 누른 사용자에게 **네이버 결제창**이 뜨는 사고가 난다. 다른 수단 선택 → 새 provider로 새 RESERVED 발급(기존은 만료).

승인 호출 후 DB 반영 실패는 "결과 모름"이라 아무 흔적 없이 RESERVED만 남기지 않고 UNKNOWN 흔적을 남긴다 — [[payment-unknown-결과불명-처리와-예외분류]].

## redirect 역조회 진입점 변경(Order 안 거침)

실전 막힌 지점: 네이버가 redirect로 `merchantPayKey`, `pgPaymentId`만 준다(order_id 없음). reserve 시점에 merchantPayKey ↔ order_id 매핑을 저장해야 주문을 되찾는다 — 이것이 reserve가 RESERVED 행을 생성해야 하는 이유다.

- 기존: Order에 박힌 merchantPayKey로 역조회(`OrderRepository.findByMerchantPayKey`).
- 재설계 후: **예약(Payment)을 merchantPayKey로 조회**해 order_id를 확보(Order를 안 거침). 역조회의 주인이 결제창 발급 entry(예약)로 일원화된다.

## 결함 — 만료 필터만으로 재발급 막힘 → EXPIRED lazy 회수

> [!warning] 자동 구현에서 드러난 설계 맹점
> 초기 결정은 "만료 예약은 EXPIRED로 마킹하지 않고 재사용 판단 시 만료 시각 필터로만 걸러낸다"였다(마킹을 두면 "누가 언제 마킹하느냐"는 또 다른 박제 위험이 생긴다고 봤기 때문). 그런데 만료 예약 행은 상태가 여전히 RESERVED라 `reserved_key`를 계속 점유한다. 만료 필터는 *재사용*만 막을 뿐 *새 발급*은 같은 키로 unique 위반을 일으켜 **같은 주문이 영원히 재예약 불가**가 된다. "필터로 충분"이 재사용 한쪽만 본 결론이었다.

- **해결**: EXPIRED 상태를 재도입하되, reserve 진입 시점에 만료/무효 예약을 그 자리에서 **lazy 회수**(status=EXPIRED + reserved_key=NULL)한 뒤 새로 발급한다. reserve를 호출하는 요청이 *자기가 쓸 자리를 직접 정리*하므로 별도 배치/스케줄러의 박제 위험이 안 생긴다. 금액 변경 같은 무효화도 같은 경로로 회수된다.
- **교훈**: 상태/필드를 단순화로 제거할 때 영향을 *모든 경로(재사용 + 재발급)에서* 따져야 한다.

## /payments/ready → /reserve rename

의미가 흐려진 외부 API 이름을 정정했다. "ready"보다 "reserve"가 이 단계의 실제 의미(예약 생성/재사용)에 맞다. frontend가 아직 미개발이라 하위 호환을 깨도 무비용인 구간이었고, 틀린 이름은 운영 트래픽이 붙을수록 정정 비용이 커지므로 지금 바로잡았다. (예약을 별도 테이블로 분리한 결정은 [[payment-reserve-예약테이블-분리-a안-b안]].)

## 미해결 — 네이버 예약 API 호출 여부·batch 삭제

| 항목 | 현재 | 결정 조건 |
|---|---|---|
| reserve의 네이버 예약 API 호출 여부 | 미확정 | (a) 서버가 예약 API 호출해 reserveId 수신 → 만료 RESERVED는 신규 행(EXPIRED 전이 후) / (b) 서버가 키만 발급 → in-place 갱신 가능. 만료 재발급 방식·반환 필드를 확정 |
| RESERVED batch 삭제 | 보류 | expiresAt 조회 필터로 기능은 전부 동작. 테이블 비대 시 청소 batch |

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 기존 ready 3문제, RESERVED 행 생성/재사용, expiresAt·provider 재사용 조건, 박제 자동복구, 역조회 진입점 변경.
- [[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]] — 만료 필터만으로 재발급 막힘 결함과 EXPIRED lazy 회수, `/reserve` rename.
- [[raw/sessions/backend/2026-06-04-payment-reserve-table-split-b-option]] — 예약 행 대상 재사용/만료 흐름의 테이블 반영.
