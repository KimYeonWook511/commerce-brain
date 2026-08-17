---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, order, domain-separation, pg-gateway, naverpay, order-number, package-structure]
created: 2026-06-04
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]]"
  - "[[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]]"
---

# 결제·주문 도메인 분리와 PG 격리 — 재설계 동기와 식별자 재배치

> [!note] 진화 (2026-08) — 이 허브가 가리키는 클러스터가 2차 재설계로 갈렸다
> 이 노트는 1차 재설계(2026-06) 클러스터의 허브다. 동기·도메인 경계·식별자 재배치라는 뼈대는 유효하지만, 이 허브가 "각자 별도 노트에 있다"고 가리킨 개별 결정 중 여럿이 2026-08에 뒤집혔다.
> - **reserve 흐름 → 폐지.** 예약이 결제 행의 활성 슬롯으로 흡수됐다([[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]).
> - **부분취소 → 열렸다.** 그리고 환불이 독립 aggregate가 됐다([[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]).
> - **PG 격리의 형태가 바뀌었다.** 이 노트가 세운 격리 의도는 유효하지만, 실제 구조는 "결제사를 최상위 묶음으로 두고 그 안에 네 계층"이 되어 규칙이 경고한 형태가 되어 있었다 — 나가는 인터페이스 하나 + 결제사마다 어댑터 하나로 재편했다([[결제사-연동타입-인프라-격리와-나가는-호출-읽기제한시간]]).
> - **트랜잭션 경계는 확장됐다** — 경계를 넘는 자리 셋을 허용하고 대신 파일 단위로 세게 만들었다([[돈정합성-우선-경계넘는-트랜잭션-셋-허용과-셀-수-있게]]).
>
> 결제 어휘(결제대행사 vs 간편결제)와 세 층의 역할 정리는 [[결제사-간편결제-구분과-세-층-역할-결과불명-재시도-모델]]에 따로 있다.

결제/주문 재설계 클러스터의 허브 노트다. 이 재설계에서 갈라져 나온 개별 결정(멱등·완료판단·reserve 흐름·UNKNOWN·트랜잭션 경계·부분취소)은 각자 별도 노트에 있고, 여기서는 그 전부를 관통하는 **동기·도메인 경계·패키지 구조**만 담는다.

## 재설계 동기 — merchantPayKey 위치·상태 혼재

기존 NaverPay 연동은 "동작은 하나 구조가 이상하다"는 감이 있었고, 파고들어 확인한 실제 원인은 둘이었다.

1. **가변 값을 불변 엔티티에 박음.** 시도마다 바뀌는 가맹점 결제 키(merchantPayKey)를 불변인 `Order`에 저장(`Order.merchantPayKey` + `assignMerchantPayKey`, null일 때만 채우는 멱등 setter)해서, 주문 하나에 키 하나만 남고 **결제 시도 이력이 소실**됐다. 설계 문서엔 "금액 변경 시 새 키"라 적혀 있었으나 코드엔 새 키 발급 경로 자체가 없어 문서와 코드가 어긋나 있었다.
2. **현재 상태와 개별 시도 결과를 한 곳에 뭉갬.** "결제의 현재 유효 상태"와 "마지막 시도의 결과"를 status 한 컬럼에 섞어 표현하려 해, `FAILED`가 "승인 실패"인지 "취소 실패"인지 구분되지 않았다.

이 두 문제를 푸는 것이 재설계 전체의 출발점이다. 상태와 이력을 가르는 결정은 [[payment-append-only-원장과-exists-완료판단]]에서 상술한다.

## Order와 Payment는 다른 도메인

먼저 합의한 핵심 원칙은 **주문과 결제는 책임이 다른 도메인**이라는 것이다.

- **Order**: "무엇을 얼마에 산다." 불변 식별자와 주문 시점 스냅샷만 갖고, 결제 수단을 모른다.
- **Payment**: "그 돈을 어떻게 받는다." 결제 시도를 append-only로 쌓는다.

이 분리에 따라 Order가 들고 있던 결제 식별자 발급·저장 책임을 결제 쪽으로 전량 이관했다(`assignMerchantPayKey`, `findByMerchantPayKey*` 등 제거). 그 결과 같은 주문의 다중 PG 재시도나 금액 변경이 *새 예약 발급*으로 자연스럽게 표현된다. 결합이 남는 승인 확정·보상 판단은 한쪽 도메인이 아니라 조율자(facade) 한 점으로 모은다 — [[payment-order-facade-결합끊기-tell-dont-ask]].

## PG 추상화와 naver 패키지 격리

NaverPay는 결제수단 구현체 중 하나일 뿐이다. `PaymentGateway` 인터페이스(reserve/approve/cancel/provider)로 추상화하고, 네이버 구현을 `naver` 패키지로 격리해 **네이버 전용 DTO가 그 패키지 밖으로 새어나가지 않게** 했다.

- `NaverPayGateway` — 네이버 응답을 도메인 결과로 번역하는 경계.
- `NaverPayApiClient` — 순수 HTTP 통신.
- `NaverPayProperties` / `dto` — 설정과 네이버 전용 DTO(패키지 밖 노출 금지).

이 격리 덕에 provider가 늘어도 게이트웨이 구현만 추가하면 되고, 결제 응용/도메인은 provider 중립을 유지한다.

## id vs orderNumber(내부/외부 식별자)

Order에서 merchantPayKey를 떼어내면서, 외부 노출용 식별자를 내부 PK와 분리해 도입했다.

| 값 | 위치 | 성격 |
|---|---|---|
| `id` (PK) | Order | 내부 식별자(조인·내부 조회용, 추측 가능·DB 세부) |
| `orderNumber` | Order | 주문당 1개·불변, 외부(사용자/PG/CS/송장) 노출용(추측 어려운 비즈니스 키) |
| `merchantPayKey` | Payment(예약) | 서버 발급, 시도마다, PG 식별 + redirect 역조회 키 |
| `pgPaymentId` | Payment | 네이버가 그 시도에 발급한 결제 ID(redirect로 수신) |

외부로 나가는 모든 자리엔 orderNumber를, 내부 조회는 order_id를 쓴다. MSA 전환 시 서비스 경계를 넘는 식별자도 내부 PK가 아니라 비즈니스 키가 선호된다. 재설계 착수 시점 Order 엔티티엔 아직 orderNumber가 없어, 그 도입 자체가 이 재설계의 목표에 포함됐다. Payment가 Order를 참조하는 방식(order_id 값, FK 물리 제약 없음)의 컨벤션은 [[cross-aggregate-fk-to-id-참조-컨벤션]]을 따른다.

## 목표 패키지 구조

```text
com.commerce
├─ order      (domain / application / presentation / repository)
└─ payment
   ├─ domain          (Payment, PaymentStatus, PaymentType, PaymentFailCode, PaymentProvider)
   ├─ application     (ready/approve/cancel 오케스트레이션)
   ├─ presentation    (redirect/callback 진입점)
   ├─ repository
   └─ provider/gateway
      ├─ PaymentGateway   (인터페이스)
      └─ naver            (NaverPayGateway / ApiClient / Properties / dto — 밖으로 안 나감)
```

결제 데이터는 최종적으로 예약(`PaymentReservation`)과 사건(`Payment`) 두 테이블로 갈렸다 — [[payment-reserve-예약테이블-분리-a안-b안]]. 외부 API도 이 재설계에서 `/payments/ready` → `/payments/reserve`로 정정했으나, frontend 미개발 구간이라 하위 호환 비용은 없었다(상세 [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]]). 엔티티 통합 후 남은 `attempt` 네이밍 잔재 정리는 [[payment-attempt-네이밍-정리와-refactor-경계]]에서 다뤘다.

## 근거

- [[raw/sessions/backend/2026-06-04-payment-order-redesign-decisions]] — 재설계 동기, 도메인 분리 원칙, id/orderNumber, PG 격리 패키지 구조.
- [[raw/sessions/backend/2026-06-05-pr-205-payment-redesign-review-fixes]] — merchantPayKey 책임 이관(Order→예약), 두 테이블 분리, `/reserve` rename의 실제 반영.
