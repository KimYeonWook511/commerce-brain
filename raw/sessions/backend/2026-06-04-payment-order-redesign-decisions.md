---
platform: backend
author: KimYeonWook511
created: 2026-06-04
---

> NaverPay 연동을 포함한 Payment/Order 도메인 재설계 전, 사용자가 정리한 17개 설계 결정 기록. ADR로 승격되기 전 단계의 사고 흔적.

# 결제 / 주문 도메인 재설계 — 설계 결정 기록 (NaverPay)

> 목적: NaverPay 연동을 포함한 Payment / Order 도메인을 재설계하기 위해, 어떤 고민을 했고
> 어떤 문제가 있었으며, 어떤 선택지에서 무엇을 왜 결정했는지를 정리한 문서.
> 이 문서를 기준으로 Claude Code와 함께 두 도메인을 재설계한다.
>
> **DB는 MySQL(InnoDB)** 이다. 이 전제가 멱등/동시성 설계(결정 6·13·14·15)에 직접 영향을 준다.

> **용어**
> - 이 문서에서 **Payment**는 "결제 시도 단위" 엔티티다(append-only, **주문당 N개**). 본문의 "시도"는 이 Payment 행 하나를 가리킨다. (1:1 결제 한 건이 아님에 주의)
>   - 현재 코드의 `PaymentAttempt`(`tbl_payment_attempt`)가 이 엔티티에 해당하며, 재설계 시 이름을 `Payment`로 통일한다.
> - **ready = reserve**: 결제창 호출 정보를 만들어 프론트에 주는 "결제 준비" 단계. 현재 코드의 `PaymentReadyService.readyPayment()`가 이 단계다. 본문에서 reserve / ready는 같은 것을 가리킨다.

---

## 0. 출발점

기존 NaverPay 연동은 동작은 했으나 "구조가 이상하다"는 느낌이 있었다. 확인된 실제 원인:

1. **가변 값(merchantPayKey)을 불변 엔티티(Order)에 박아두어**, 주문 하나에 키 하나만 남고 결제 시도 이력이 사라짐.
2. **결제의 "현재 상태"와 "개별 시도의 결과"를 한 곳에 섞어** 표현하려 함.

아래는 이 두 가지를 풀어가며 내린 결정들의 기록이다.

---

## 핵심 원칙 (먼저 합의한 것)

- **Order와 Payment는 다른 도메인이다.** 주문(무엇을 얼마에 산다)과 결제(그 돈을 어떻게 받는다)는 책임이 다르다.
- **PG는 추상화한다.** NaverPay는 결제수단 구현체 중 하나. `PaymentGateway` 인터페이스 + `naver` 패키지로 격리하고, 네이버 전용 DTO가 그 패키지 밖으로 새어나가지 않게 한다.
- **시도(Payment 행)는 불변 이벤트, 상태는 그 시도들을 종합한 결과다.** 시도는 append-only로 쌓고, "현재 상태"는 시도들로부터 도출한다.
- **방어는 항상 "1차 필터 + 최종 방어선" 이중 구조다.** 앱/Redis/UI 레벨의 빠른 1차 필터는 없어도 시스템이 돌아가야 하고, 정합성·중복의 최종 보장은 DB 제약에 둔다. (결정 12)
- **존재 보장은 unique 제약으로, 계산 기반 판단은 PK 단일 행 lock으로.** (결정 15 — gap lock 회피)

---

## 결정 1. 결제 시도는 상태 컬럼 하나로 전이시키지 않는다

- **고민**: `RESERVED → COMPLETED → (CANCELED / FAILED)`처럼 status 하나에 전이를 다 태우는 게 맞나?
- **문제**: 한 컬럼이 두 종류 정보를 떠안음 — (a) 결제의 현재 유효 상태, (b) 마지막 시도의 결과. 특히 `FAILED`가 "승인 실패"인지 "취소 실패"인지 뭉개진다.
- **결정**: 시도 이력(Payment, append-only)과 현재 상태를 분리. 승인요청→성공/실패, 취소요청→성공/실패를 각각 "일어난 사실"로 기록한다.
- **이유**: 시도는 한번 일어나면 바뀌지 않는 사실이다. 재시도로 성공해도 "실패했던 그 시도"가 성공으로 변하는 게 아니라 새 시도가 성공한 것. 외부 PG 연동에서는 "결과 모름(UNKNOWN)"도 남겨야 하므로 시도 단위 기록이 필수.

---

## 결정 2. merchantPayKey — 서버(가맹점)가 발급, Order가 아니라 시도(Payment)에 둔다

- **발급 주체**: merchantPayKey는 **서버(가맹점)** 가 발급한다. (이름 그대로 *merchant* PayKey) 클라이언트가 만들지 않는다.
  - 이유: 유일성을 서버가 통제(DB unique), 우리가 시도를 추적하는 신분증, 재무 식별자라 클라이언트 신뢰 불가, reserve 때 네이버에 보내는 값.
- **발급 시점**: **reserve(ready) 단계**. 재시도(새 결제 시도)면 새 키, 같은 시도의 재전송이면 같은 키.
- **클라이언트 idempotencyKey와의 관계**: 둘은 **다른 층**이다. (혼동 주의)
  | | merchantPayKey | (선택) 클라이언트 idempotencyKey |
  |---|---|---|
  | 발급 | 서버 | 클라이언트 |
  | 목적 | PG와의 결제 거래 식별 | "결제 시작 요청"의 중복 제출 방지 |
  | 막는 것 | PG 레벨 결제 중복 | 더블클릭/재시도로 요청이 두 번 호출되는 것 |
  | 필수 | 필수 | 선택 |
  - 클라이언트 키를 도입하더라도 merchantPayKey는 여전히 서버 발급.
- **위치(과거 문제의 핵심)**: 현재 코드는 merchantPayKey를 **Order에 저장**(`order.assignMerchantPayKey`, null일 때만 발급)한다. 이는 (a) 주문당 키 하나로 고정되어 이력이 안 쌓이고, (b) 가변 값을 불변 엔티티에 박으며, (c) provider가 키에 안 묶여 "수단 바꿔도 이전 결제창" 문제를 낳는다.
- **결정**: merchantPayKey를 Order에서 떼어 **Payment(RESERVED 행)** 에 둔다. Order에는 불변 식별자만 남긴다.
- **값별 거주지 정리**
  | 값 | 위치 | 성격 |
  |---|---|---|
  | `orderNumber` | Order | 주문당 1개, 불변. 외부(사용자/PG/CS/송장) 노출용 |
  | `id` (PK) | Order | DB 내부 식별자(조인용) |
  | `merchantPayKey` | Payment | 서버 발급, 시도마다, PG 식별 + redirect 역조회 키 |
  | `pgPaymentId` | Payment | 네이버가 그 시도에 발급한 결제 ID(redirect로 수신) |

> **id vs orderNumber**: `id`는 내부용(추측 가능, DB 세부사항), `orderNumber`는 외부 노출용(추측 어렵게, 구현 비종속). 외부로 나가는 모든 자리엔 orderNumber. MSA 전환 시 서비스 경계를 넘는 식별자도 내부 PK가 아니라 orderNumber 같은 비즈니스 키가 선호됨.

---

## 결정 3. Payment는 order_id로 소속을 표현한다 (FK 물리 제약은 걸지 않음)

- **전제(이 프로젝트)**: 별도 "완료 상태" 엔티티를 두지 않는다(결정 5). 시도는 승인 성공 이전부터 기록.
- **문제**: 승인요청·실패 시점엔 "완료 상태"가 없으니 그걸 참조할 대상 자체가 없다.
- **결정**: 소속 참조는 **order_id**(다른 도메인의 PK를 `Long` 값으로 보유). (merchantPayKey는 PG 식별용일 뿐 내부 소속 관계가 아니므로 대체 불가)
- **DB FK 제약은 걸지 않는다**: order_id는 "참조용 값"으로만 들고, `FOREIGN KEY` 물리 제약은 두지 않는다. FK 제약은 도메인 간 결합을 강하게 만들어 마이그레이션·삭제 순서·테스트 픽스처 등에서 불편이 크다. 존재 정합성은 애플리케이션 레벨(주문 조회 후 진행)에서 보장한다.
- **트레이드오프**: type별로 참조 대상을 달리하면 조회가 지저분 → 모든 Payment 행을 order_id로 통일.

> **MSA 참고**: 어차피 FK 물리 제약을 안 쓰므로(위), 모놀리식→MSA 전환 시 **참조 방식은 그대로**다. 달라지는 건 동기 호출을 이벤트(결과적 일관성)로 바꾸고 상관관계 ID·멱등키를 다는 것, Outbox/Saga로 신뢰성·보상을 보강하는 것. 지금은 "나중에 이벤트로 바뀔 지점"만 표시해두면 충분.

---

## 결정 4. redirect 역조회를 위해 reserve 매핑을 저장한다

- **문제(실전 막힌 지점)**: 네이버가 redirect로 `merchantPayKey`, `pgPaymentId`만 준다(order_id 없음). 키를 저장해둔 적이 없으면 주문을 못 찾는다.
- **핵심**: reserve 시점에 **merchantPayKey ↔ order_id 매핑을 저장**해야 redirect로 주문을 되찾는다. → reserve 단계가 **RESERVED Payment 행을 생성**해야 하는 이유.
- **역조회 진입점 변경**: 기존엔 `Order.merchantPayKey`로 역조회(키가 Order에 박혀 가능했던 방식). 재설계 후엔 **Payment를 merchantPayKey로 조회**해 order_id 확보(Order 안 거침).
- **정보 모델 관점**: 결국 두 종류 사실뿐 — (1) 우리 안에서 일어난 일(reserve), (2) 외부 PG로 나간 요청(승인/취소+결과). reserve를 별도 테이블로 뺄지 합칠지는 정합성 문제가 아니라 "노이즈 정리" 취향(결정 13의 A/B).

---

## 결정 5. 별도 "완료 상태" 테이블의 성격이 약하다 → 단일 Payment 테이블로 충분

- **고민**: 승인 성공 시도가 이미 완료에 필요한 모든 사실(order_id, pgPaymentId, 금액, 시각, 수단)을 들고 있다. 별도 완료-상태 테이블은 "성공 시도의 복사본"일 뿐.
- **결정**: 별도 완료-상태 테이블을 두지 않는다. 결제 도메인은 **Payment 단일 테이블**로 충분.
  - "결제 완료 = 성공한 APPROVE Payment 행이 존재함"으로 정의.
- **집계 테이블이 정당해지는 유일한 경우**: 부분취소처럼 여러 시도를 집계해야 현재 잔액이 나오고, 그 계산이 잦고 무거울 때. 그때 집계 캐시 테이블을 재도입(이름은 `Payment`가 아니라 `PaymentSummary` 등, 주문당 1개). 지금은 불필요.
- **네이밍**: 단일 시도 테이블 이름은 **`Payment`**(주문당 N개, append-only). 1:1 한 건이 아님.

---

## 결정 6. 이중결제 멱등은 "주문 단위 잠금"으로 보장한다 (MySQL: NULL 트릭으로 우회)

- **고민**: 사용자가 네이버페이와 카카오페이에 둘 다 결제요청 → 둘 다 승인되면? 서로 다른 merchantPayKey라 PG 멱등으로 못 막는다.
- **핵심 통찰**: merchantPayKey 멱등은 "한 PG 안에서 같은 키 중복"만 막는다. "여러 PG에 걸친 같은 주문 중복"은 원리적으로 못 막음 → 멱등을 **주문 단위로** 끌어올려야 한다.
- **불변식**: *한 주문에는 성공한 승인이 최대 1개만 존재한다.*

### 원래 하려던 것 (PostgreSQL이라면)

```sql
-- 부분 unique 인덱스: 조건을 만족하는 행들 사이에서만 유일성 강제
CREATE UNIQUE INDEX uk_order_approved
  ON payment (order_id)
  WHERE type = 'APPROVE' AND status = 'SUCCEEDED';
```
성공한 APPROVE만 주문당 1개로 제한하고, 실패한 APPROVE는 조건 밖이라 여러 개 OK. 이게 가장 직관적이다.

### 문제: MySQL(InnoDB)에는 partial(조건부) unique index가 없다

`WHERE ...` 절을 가진 부분 인덱스 자체를 지원하지 않는다. 위 DDL이 그대로는 안 먹는다. 그렇다고 `UNIQUE(order_id)`를 통째로 걸면 실패·취소 시도까지 막혀버려 안 된다(주문당 시도가 여러 개여야 하므로).

### 결정: "조건 만족 시에만 값, 아니면 NULL" 컬럼 + 일반 unique

InnoDB unique index의 성질 — **NULL은 중복으로 치지 않는다(여러 NULL 허용)** — 을 이용한다.

```sql
-- Payment에 컬럼 추가: 성공한 APPROVE일 때만 order_id, 그 외엔 NULL
ALTER TABLE payment ADD COLUMN approved_order_key BIGINT NULL;
CREATE UNIQUE INDEX uk_approved ON payment (approved_order_key);
```
- APPROVE가 SUCCEEDED 되는 순간에만 `approved_order_key = order_id` 를 set. 그 외(REQUESTED/FAILED/CANCEL/RESERVE) = **NULL**.
- NULL은 중복 허용이라 실패·취소 시도는 제약을 받지 않음.
- 같은 주문에 두 번째 성공 APPROVE → 같은 order_id 값이 두 번 → **unique 위반으로 차단.**
- 효과는 partial unique와 **동일**(DB가 물리적으로 막아줌). 동시 승인의 race condition도 차단.
- **주의**: `status=SUCCEEDED`와 `approved_order_key` set은 반드시 **같은 트랜잭션·같은 UPDATE**에서. 엔티티의 `succeed()` 메서드 안에서 둘을 항상 함께 바꾸도록 캡슐화(중간 불일치 상태 방지).

- **막는 걸로 끝이 아니다 — 보상 취소**: 두 번째 승인은 이미 PG에서 돈이 빠졌으므로, unique 위반 감지 시 그 승인을 즉시 CANCEL(환불). (CANCEL Payment 행이 보상취소 기록이 됨)
- **방어선 사슬**: (1) 진행 중 결제 락(1차, 불완전) → (2) `uk_approved`(최종 방어선) → (3) 보상 취소.
- **중복 콜백 멱등**: 중복이면 에러가 아니라 **기존 결과 반환**(진짜 멱등). 앱 레벨 체크만 믿지 말고 DB unique를 최종 방어선으로.

---

## 결정 7. "결제 완료" 판단은 마지막 행이 아니라 조건의 존재로 한다

- **문제**: "마지막 행"은 "마지막 시도 결과"지 "현재 상태"가 아니다. 예: `#1 APPROVE SUCCEEDED`, `#2 CANCEL FAILED`(마지막) → 마지막만 보면 오판하지만 실제론 PAID.
- **결정**: 조건의 존재 여부(EXISTS)로 판단.
  ```text
  결제 완료 = (성공한 APPROVE 존재) AND (그것을 무효화한 성공 CANCEL 부재)
  ```
- **상태 도출 표**
  | 성공 APPROVE | 성공 CANCEL | 현재 상태 |
  |---|---|---|
  | 없음 | - | 미결제(진행중/실패) |
  | 있음 | 없음 | **PAID** |
  | 있음 | 있음(전액) | 취소됨 |
  | 있음 | 있음(부분) | 부분취소(일부 유효) |
- **조회 키**: 내부 조회는 order_id. orderNumber는 외부 입구로만(문의 → orderNumber→id 변환 후 order_id로 조회).

---

## 결정 8. 부분취소 확장성 — 모델은 열어두되 로직은 전액만 구현

- **현재 방침**: 우선 전액취소만 운영. 나중에 부분취소로 바꿀 때 큰 작업이 안 되게 모델만 미리 열어둠.
- **분기점**: 취소를 boolean/status로 모델링하면 부분취소 추가가 모델 수술+마이그레이션(지옥). **금액 가진 CANCEL Payment 행으로** 모델링하면 전액취소는 부분취소의 특수 케이스(취소액=원금)라 기능 추가 수준.
- **지금 해두는 것(부담 거의 없음)**
  1. CANCEL Payment 행에 `amount`. 전액취소라도 실제 금액을 적는다.
  2. 잔액은 SUM으로 도출(저장 X):
     ```sql
     SELECT
       SUM(CASE WHEN type='APPROVE' THEN amount ELSE 0 END)
       - SUM(CASE WHEN type='CANCEL' THEN amount ELSE 0 END)
     FROM payment
     WHERE order_id = :orderId AND status = 'SUCCEEDED';
     ```
- **amount 규칙**: 각 시도는 "그 시도가 움직인 금액"을 적는다(APPROVE=+, CANCEL=−). 계산값(현재 잔액)은 행에 박지 않는다. (타입은 `int` 유지 — 원 단위 정수. 큰 거래액 시 int 상한 21.4억 원만 의식)
- **나중에 부분취소 켤 때 변경 범위**: 데이터 모델 0, 잔액 조회 0, 추가되는 건 취소금액 파라미터·과다취소 검증(결정 15의 lock)·PG cancelAmount·UI 정도.
- **YAGNI**: 부분취소 로직까지 미리 만들지 않는다. 모델만 열어두고 구현은 전액만.
- **부분 unique와 충돌 없음**: 이중결제 제약은 APPROVE 성공에만 걸리므로 CANCEL 여러 개 SUCCEEDED여도 무관.

---

## 결정 9. 부분취소도 PG에 요청한다

- **결정**: 부분/전액 모두 **반드시 PG(NaverPay)에 취소 요청**. 우리 DB만 고치지 않는다.
- **이유**: 돈이 빠진 곳이 PG. 환불도 PG만 가능. 우리 잔액은 "우리 인식"일 뿐, 실제 환불은 PG가 사용자에게 돌려줘야 완성.
- **순서**: PG 취소 성공 → 그 결과를 받아 CANCEL Payment 행 기록(SUCCEEDED, amount=취소액). 먼저 적고 보내지 않는다.
- **요청 항목**: 원 승인의 `pgPaymentId` + 취소금액 + 사유. (APPROVE 때 pgPaymentId 저장 이유)
- **챙길 것**: 과다취소 검증(결정 15), 취소 멱등키(타임아웃 재시도 과다환불 방지), 네이버 부분취소 스펙(허용 횟수·복합결제 취소 순서·면세/과세 분해 — 문서 확인).

---

## 결정 10. PG 응답 원문 보관 테이블(PaymentEvent류)은 지금 만들지 않는다

- **성격**: 결제 로직이 아니라 운영 안전망(대사·분쟁·디버깅·웹훅 멱등 증거). 핵심 기능은 Payment 하나로 동작.
- **결정**: 지금은 만들지 않고 **로그로 대체**(YAGNI).
- **조건 둘**: (1) PG 응답 **원문(JSON)은 로그에라도 반드시 남긴다**(분쟁 시 유일한 증거). (2) **승격 트리거**를 정해둔다(분쟁/CS 증가, 대사 자동화 필요, 결제량 규모 초과 → 테이블 승격).
- **로그 대체의 취약점(인지하고 감수)**: 조건 쿼리 불가, 로테이션 소실/정합성 약함.
- **네이밍(승격 시)**: `PaymentPGResponse`는 약어 중복 + "Response"가 웹훅을 못 담아 아쉬움. `PgTransactionLog`/`PaymentTransactionLog`처럼 요청·응답·웹훅 포괄 + 로그성이 드러나는 이름. 확정은 승격 시점으로.

---

## 결정 11. 실패 사유는 Payment 행에 컬럼으로 남긴다 (로그로 빼지 않는다)

- **구분(중요)**: 추상화 수준이 다른 두 가지를 헷갈리면 안 됨.
  - **원문 전체(PaymentEvent)** = PG 응답 날것 JSON → 무거움 → **로그로 미룸**(결정 10).
  - **실패 사유** = "왜 실패했나"의 해석된 요약 → 쿼리·안내·분석·재시도 분기에 직접 쓰임 → **DB(Payment 행) 컬럼으로 유지**.
- **결정**: 실패 사유는 DB에 유지한다. (현재 코드의 `failCode` + `failDetail`이 이미 정답)
  ```
  failCode    : 내 정규화 enum (CARD_LIMIT_EXCEEDED, INSUFFICIENT_BALANCE, TIMEOUT, USER_CANCELED, UNKNOWN ...)
  failDetail  : PG 원문 메시지(그대로, CS용)
  ```
- **이유**: PG마다 에러 코드가 달라 분석·분기는 내 코드 기준이어야 일관됨. UNKNOWN(결과 모름)도 사유를 남겨 대사 단서로.
- **점검**: `failDetail`에 length 제한(예: 255). 원문 통째는 로그로, 여기엔 사람이 읽을 요약만.
- **결정 1과 모순 아님**: "현재 결제 상태"엔 FAILED를 안 넣지만, 개별 시도(Payment 행)의 결과로서 FAILED+사유를 남기는 건 정상.

---

## 결정 12. 중복 요청(따닥) 방어 — 지점별로, "1차 필터 + 최종 방어선"

따닥(더블클릭/연타/네트워크 재시도)은 흐름의 여러 지점에서 발생. 지점마다 방어가 다르며, **진짜 사고(이중결제)는 결정 6의 unique가 최종적으로 막는다.** 앞단 방어는 "노이즈 감소 + UX"지 "사고 방지"가 아님 — 여기에 과하게 힘 빼지 않는다.

- **승인 콜백 따닥**: 이미 막힘. 결정 6 unique + 중복 시 기존 결과 멱등 반환. 추가 작업 없음.
- **취소 따닥**: CANCEL은 여러 개 SUCCEEDED 가능하므로 이중결제 unique로 안 막힘.
  - 전액취소: "이미 취소됨?" 체크 후 멱등 반환 + 행 잠금(결정 15).
  - 부분취소: 취소 멱등키로 같은 취소 두 번 환불 방지(PG 취소 API에도 동봉).
- **reserve(ready) 따닥**: 상세는 결정 13. (행 노이즈일 뿐 사고 아님)

### Redis냐 DB냐 — 장애 전파 차단 원칙

(Order 생성에서 이미 적용한 원칙을 결제에도 동일하게)

- **Redis만**: 빠르지만 best-effort. Redis 장애 시 dedup이 빠져 따닥이 통과.
- **DB unique**: 물리적 차단, 절대 안 뚫림(강한 보장).
- **결정**: Redis는 **1차 필터(없어도 시스템 동작)**, **최종 보장은 DB 제약**. Redis가 죽어도 결제는 (조금 느릴 뿐) DB unique로 안전 → **장애가 결제로 전파되지 않음.** 결제는 돈이므로 Order보다 더 이 원칙을 지킨다.
- **reserve 따닥에 한해**: 클라이언트 idempotencyKey(Redis+DB)보다 **order 단위 unique가 더 단순**(키 발급·전달 없이 "주문당 진행중 reserve 1개"를 DB가 강제). → 결정 13에서 채택.

---

## 결정 13. ready(reserve) 단계 재설계 — RESERVED 행 생성 + expiresAt 기반 재사용/만료

- **현재 코드(`PaymentReadyService`)의 문제**
  1. merchantPayKey를 **Order에 저장**(null일 때만 발급) → 키 고정·이력 소실·provider 미반영(결정 2).
  2. ready가 **RESERVED 행을 만들지 않고** 키만 반환 → redirect 역조회·따닥 재사용·만료를 할 수가 없음.
  3. expiresAt/provider 기반 **재사용·만료 로직 부재** → 한번 박힌 키가 영원히 재사용됨.
- **결정**: ready를 "키 발급+반환"에서 **"RESERVED Payment 행 생성/재사용 + 반환"** 으로 바꾼다.

### 박제(stale RESERVED) 문제와 해결

- **겪은 문제**: 진행 중 reserve 재사용만 하면, SUCCESS 전환 중 DB 장애 등으로 RESERVED가 남고, "유효한 RESERVED 있음"으로 판단해 **죽은 키를 계속 반환** → 답 없음.
- **원인**: "유효한 RESERVED"를 **status만으로** 판단(시간 미반영).
- **해결**: 판단 기준에 시간을 넣는다.
  ```
  유효한 RESERVED = (status = RESERVED) AND (expiresAt > now)
  ```
  박제된 행은 만료되는 순간 재사용 대상에서 빠지고, 다음 요청이 새로 발급 → **자동 복구.** (재사용은 반드시 expiresAt과 함께)
- **정합성 보강(원칙만)**: 승인 호출 후 DB 반영 실패 = "결과 모름"이므로 **UNKNOWN으로라도 흔적을 남긴다**(아무 흔적 없이 RESERVED만 남는 게 가장 위험). 사용자 응답·해소는 결정 14.

### 재사용 조건 — provider까지 일치

- RESERVED엔 provider가 들어있다. "유효한 RESERVED 있으면 무조건 반환"하면, 네이버로 띄웠다 카카오를 누른 사용자에게 **네이버 결제창**이 뜨는 사고.
- **조건**: `status=RESERVED ∧ expiresAt>now ∧ provider = 요청 provider`
  - 같은 수단 따닥 → 재사용. 다른 수단 선택 → 새 provider로 새 RESERVED 발급(기존은 만료).

### 반환 정보

- 만료 전 유효 RESERVED가 있으면 그 행의 **merchantPayKey + 결제창 호출 정보**(clientId/chainId/returnUrl/금액/상품명 등, 현재 `PaymentReadyResult`)를 반환.
- 반환은 **결제창 여는 최소 정보**만(PG 시크릿·내부 id 금지). redirect로 돌아온 값은 신뢰하지 말고 승인 시 금액/키 재검증(현재 `verifyApprovedResponse`).

### 동시 따닥 차단 — (MySQL) NULL 트릭으로 우회

**원래 하려던 것 (PostgreSQL이라면)**
```sql
CREATE UNIQUE INDEX uk_order_provider_reserved
  ON payment (order_id, provider)
  WHERE status = 'RESERVED';
```
"(주문, 수단)당 RESERVED 1개"를 강제. 그러나 이것도 **partial unique라 MySQL(InnoDB)에서 안 된다.**

**MySQL 우회 — 결정 6과 같은 방식**
```sql
-- RESERVED일 때만 "order_id:provider" 조합값, 그 외(승인/만료/취소)엔 NULL
ALTER TABLE payment ADD COLUMN reserved_key VARCHAR(96) NULL;
CREATE UNIQUE INDEX uk_reserved ON payment (reserved_key);
```
- RESERVED일 때만 `reserved_key = "{order_id}:{provider}"`, RESERVED를 벗어나면(승인되거나 만료 전이되면) **NULL로 비운다.**
- RESERVED인 행들 사이에서만 (order, provider) 유일성 → 동시 따닥 두 번째 INSERT 차단 + RESERVED 무한 적재 방지.
- 만료된 RESERVED를 새로 발급할 때: 기존 행의 `reserved_key`를 NULL로(또는 status를 EXPIRED로 전이하며 비움) → 슬롯이 비어 새 RESERVED 가능.
- **주의**: `reserved_key` set/NULL을 status 변경과 같은 트랜잭션·같은 UPDATE에서. 엔티티 메서드(`createReserved`, `expire`, 승인 전이)에 캡슐화.

### 의사코드

```
readyPayment(orderId, memberId, provider):
    order = 조회 + checkPayable()
    reserved = findReusableReserved(orderId, provider, now())   // RESERVED ∧ not expired ∧ provider 일치
    if reserved:
        key = reserved.merchantPayKey                            // 재사용
    else:
        key = "PAY-" + ULID                                      // 서버 발급
        save(Payment.createReserved(orderId, provider, key, amount, expiresAt = now + N분))
        // 동시 따닥은 uk_reserved 가 두 번째를 차단
    return PaymentReadyResult(clientId, chainId, key, productName, totalPayAmount, returnUrl(key), ...)
```

### 배치 삭제

- 보류(우선순위 낮음). expiresAt **조회 필터**만으로 기능 전부 동작. RESERVED가 테이블을 비대하게 만들 때 가서 청소 batch 검토.

---

## 결정 14. UNKNOWN(승인 결과 모름) 처리 정책

- **상황**: 승인 API를 호출했는데 응답을 못 받음(타임아웃/네트워크 단절). 네이버가 승인했는지(돈 빠짐) 아닌지 **알 수 없음.** → 그 시도를 SUCCEEDED도 FAILED도 아닌 **UNKNOWN**으로 남긴다.
- **위험**: "실패"로 단정하면 사용자가 재시도 → 실제론 승인됐던 경우 이중결제(결정 6 unique가 막아주나, 막힌 승인은 이미 돈이 빠져 보상취소 필요). "성공"으로 단정하면 미결제 주문을 PAID 처리. 둘 다 위험.
- **결정(권장)**: **모를 땐 모른다고 안내하고, 행동을 막는다.**
  - 사용자 안내: "결제 결과를 확인 중입니다. 잠시 후 주문 내역에서 확인해 주세요."
  - 재시도: **차단**(이미 승인됐을 수 있으므로 새 결제 시작 금지).
  - 확정: 추후 **대사(네이버 조회 API)** 로 UNKNOWN → SUCCEEDED/FAILED 보정.
- **가벼운 대사 권장(여력 되면)**: 전체 배치 대사는 나중에 만들더라도, 사용자가 "확인 중" 화면 재진입/주문내역 조회 시 **그 주문의 merchantPayKey로 네이버 조회 API를 단건 호출**해 결과를 확정하는 정도는 작아서 권장. UNKNOWN이 영영 안 풀리는 것을 막는다.

---

## 결정 15. 동시성 — 존재 보장은 unique, 계산 판단은 PK 단일 행 lock (gap lock 회피)

- **배경 고민**: 결제는 예민하니 lock을 걸까? 그런데 lock이 남의 결제까지 막지 않을까?
- **사실 정리(InnoDB)**
  - **Row lock**: `SELECT ... FOR UPDATE`로 특정 행을 잠그면 **그 행에만** 영향. 서로 다른 주문이면 충돌 없음 → 남의 결제 안 막음.
  - **Gap lock(next-key lock)**: 기본 격리수준(REPEATABLE READ)에서 **존재하지 않는 행을 조건으로 잠그거나 범위로 잠그면**, 그 사이 빈 구간까지 잠가 **다른 트랜잭션의 INSERT를 막는다.** 인덱스를 따라 걸리므로 적절한 인덱스가 없으면 더 넓게 잠긴다. → 이게 "남의 결제가 막히는" 원인.
  - 특히 **"없는 행 조회 FOR UPDATE → 없으면 INSERT" 패턴이 gap lock을 유발**(reserve 재사용에서 쓰려던 바로 그 패턴).
- **결정**
  - **존재 보장(없으면 INSERT, 있으면 막기)** → **lock이 아니라 unique 제약**으로. unique는 그냥 INSERT 시도 후 충돌 시 거부 → gap을 미리 안 잠금 → 남의 결제 안 막음. (이미 결정 6·13에서 unique 채택)
  - **계산 기반 판단(여러 행을 읽어 합산 후 결정)** → **PK로 order 단일 행 FOR UPDATE.** 대표 사례: 부분취소 과다취소 검증(잔액=SUM 후 취소액≤잔액 확인). PK 단일 행 lock은 gap을 만들지 않고, 그 주문에만 직렬화되어 남의 결제 영향 없음.
  - **피할 것**: "없는 행 조회 FOR UPDATE → INSERT". gap lock으로 옆 범위 INSERT까지 막을 수 있음.
- **요약 표**
  | 상황 | 방법 | 남의 결제 영향 |
  |---|---|---|
  | 이중결제 / reserve 따닥 (없으면 INSERT) | **unique 제약**(lock 아님) | 없음. gap lock 안 생김 |
  | 부분취소 과다취소 검증 (계산 후 판단) | order 행 **FOR UPDATE**(PK 단일 행) | 없음(그 주문만), gap도 거의 없음 |
  | 피해야 할 것 | 없는 행 조회 FOR UPDATE → INSERT | gap lock으로 남의 INSERT 막음 ⚠️ |
- **결론**: "lock을 안 걸고 싶다"는 직감은 옳았다. 단 그 대신 **unique 제약이 거는 짧은 잠금에 맡기는 것**이고, lock이 꼭 필요한 곳(계산 판단)은 **PK 단일 행**으로 좁게 건다.

---

## 결정 16. 트랜잭션 경계 — 외부 호출은 트랜잭션 밖, 두 DB 쓰기는 한 트랜잭션 안

- **문제**: 승인 성공 시 `payment(SUCCEEDED)` 와 `order(PAID)` 두 쓰기가 따로 커밋되면 "결제됐는데 주문 미결제"(또는 반대) 불일치. 반대로 외부 API 호출을 트랜잭션 안에 넣으면 커넥션을 오래 점유.
- **결정(아마 현재도 지키는 중)**
  ```
  [트랜잭션 밖]  네이버 승인 API 호출 → 결과 수신
  [트랜잭션 안]  payment SUCCEEDED(+ approved_order_key set) + order PAID   ← 같은 트랜잭션
  ```
- **원칙**: 외부 호출(승인/취소)은 트랜잭션 밖에서, 그 결과를 받아 **DB 반영(payment + order)은 하나의 짧은 트랜잭션**으로. 박제 문제(승인 성공 후 DB 반영 실패)도 이 경계에서 발생하므로, 반영 실패 시 UNKNOWN 흔적(결정 14)으로 처리.

---

## 결정 17. 취소 시 주문 상태 전이 (취소 도입 시)

- **현황**: 결제 취소는 아직 미구현. 현재 결제 플로우가 검증되면 취소를 추가할 예정.
- **결정(모놀리식)**: 모놀리식이므로 **성공한 CANCEL이 생기면 같은 트랜잭션에서 Order를 바로 취소 상태로 전이**한다(별도 이벤트/비동기 불필요).
  ```
  CANCEL SUCCEEDED  →  같은 트랜잭션에서 order.cancel() (orderStatus = CANCELED)
  ```
- **부분취소 시(나중에)**: Order는 단일 상태라 "일부 환불"을 다 담지 못하므로, 전액취소면 CANCELED, 부분취소면 별도 표기(예: 부분환불 플래그/환불금액)를 둘지 그때 결정. 전액취소만 하는 동안은 단순히 CANCELED로 충분.

---

## 최종 결론 (재설계 목표 구조)

### 엔티티

```
Order
 ├─ id                 (PK, 내부)
 ├─ orderNumber        (외부 노출용, 불변)
 ├─ memberId
 ├─ orderItems         (1:N, 주문 시점 스냅샷)
 ├─ totalPrice         (승인 금액 대조용)
 ├─ orderStatus        (CREATED → PAID → CANCELED/REFUNDED)
 └─ createdAt
        ※ merchantPayKey 컬럼 제거 (Payment로 이동)

Payment               (결제 시도 단위, append-only, 주문당 N개. 현재 PaymentAttempt)
 ├─ id
 ├─ orderId            (소속 참조 — Order PK 값, FK 제약 없음)
 ├─ provider           (NAVER_PAY ...)
 ├─ merchantPayKey     (서버 발급, redirect 역조회 키)
 ├─ pgPaymentId        (네이버 발급. RESERVE 단계엔 아직 없음 → nullable)
 ├─ type               (RESERVE / APPROVE / CANCEL)
 ├─ status             (RESERVED / REQUESTED / SUCCEEDED / FAILED / UNKNOWN / (EXPIRED))
 ├─ amount             ("그 시도가 움직인 금액", int)
 ├─ failCode           (실패 사유 — 내 정규화 enum, nullable)
 ├─ failDetail         (실패 사유 — PG 원문 메시지, nullable, length 제한)
 ├─ expiresAt          (RESERVE 만료 시각)
 ├─ approved_order_key (성공 APPROVE일 때만 order_id, 그 외 NULL — 이중결제 unique용)
 ├─ reserved_key       (RESERVED일 때만 "orderId:provider", 그 외 NULL — reserve 따닥 unique용)
 ├─ respondedAt
 └─ createdAt
```

### 필수 제약 (MySQL/InnoDB — NULL 트릭으로 partial unique 대체)

```sql
-- 이중결제: 주문당 성공 승인 1개 (결정 6)
--   approved_order_key = 성공 APPROVE일 때만 order_id, 그 외 NULL
CREATE UNIQUE INDEX uk_approved ON payment (approved_order_key);

-- reserve 따닥: (주문, 수단)당 진행 중 RESERVED 1개 (결정 13)
--   reserved_key = RESERVED일 때만 "orderId:provider", 그 외 NULL
CREATE UNIQUE INDEX uk_reserved ON payment (reserved_key);
```

> 왜 이렇게? PostgreSQL이라면 `WHERE` 조건 부분 unique 인덱스로 깔끔하지만, **MySQL(InnoDB)은 partial(조건부) unique index를 지원하지 않는다.** 그래서 "조건을 만족할 때만 값, 아니면 NULL" 컬럼 + 일반 unique로 대체한다(InnoDB는 NULL을 중복으로 치지 않음). 멱등 보장 수준은 동일. 조건 컬럼 set/NULL은 status 변경과 같은 트랜잭션에서 캡슐화.
>
> 현재 코드의 `(merchant_pay_key, provider, pg_payment_id, type)` unique는 "같은 시도의 중복 기록"은 막지만 "주문 단위 이중결제/이중 reserve"는 못 막는다. 위 두 unique를 추가로 둔다.

### 패키지 구조 (PG 격리)

```
com.commerce
├─ order      (domain / application / presentation / repository)
└─ payment
   ├─ domain          (Payment, PaymentStatus, PaymentType, PaymentFailCode, PaymentProvider)
   ├─ application     (ready/approve/cancel 오케스트레이션)
   ├─ presentation    (redirect/callback 진입점)
   ├─ repository
   └─ provider/gateway
      ├─ PaymentGateway          (인터페이스: reserve/approve/cancel/provider)
      └─ naver
         ├─ NaverPayGateway      (네이버 응답을 도메인 결과로 번역)
         ├─ NaverPayApiClient    (순수 HTTP 통신)
         ├─ NaverPayProperties   (설정)
         └─ dto                  (네이버 전용 DTO — 패키지 밖으로 안 나감)
```

### 핵심 흐름

```
1. 주문 생성   OrderService.createOrder() → Order(CREATED), orderNumber 발급. 결제는 모름.
2. ready       유효 RESERVED 재사용(있으면) 또는 RESERVED Payment 행 생성(서버가 merchantPayKey 발급, expiresAt·reserved_key 부여).
               결제창 정보 반환. 동시 따닥은 uk_reserved 가 차단.
3. 결제창      프론트가 네이버 SDK로 결제창. 사용자 결제 → redirect(merchantPayKey, pgPaymentId)
4. 역조회      merchantPayKey로 Payment(RESERVED) 조회 → orderId 확보 (Order 안 거침)
5. 승인        [트랜잭션 밖] 네이버 승인 API 호출(merchantPayKey 동봉) → 결과 수신
                            응답 금액과 order.totalPrice 대조(redirect 파라미터는 신뢰 안 함)
               [트랜잭션 안] 성공: APPROVE 행 SUCCEEDED + approved_order_key=orderId + order PAID
                            (uk_approved 가 주문당 1개 보장)
                            실패: FAILED + failCode/failDetail
                            결과 모름(타임아웃 등): UNKNOWN 흔적 → 결정 14 정책
                            uk_approved 위반(이미 결제된 주문): 중복 → 보상 CANCEL
6. 취소        (도입 시) order 행 FOR UPDATE → 잔액 검증(SUM) → CANCEL 행 → 네이버 취소 API(pgPaymentId, 금액, 사유)
               → 성공: SUCCEEDED, amount=취소액, 같은 트랜잭션에서 order 취소 전이(결정 17)
```

### 멱등 / 신뢰 / 동시성 요약

- merchantPayKey: 서버 발급. 같은 시도 재전송엔 같은 키, 새 시도엔 새 키.
- 이중결제: `uk_approved`(approved_order_key NULL 트릭) + 보상취소.
- reserve 따닥: `uk_reserved`(reserved_key NULL 트릭) + expiresAt 재사용/만료(+ 프론트 버튼 disable).
- 중복 콜백: 에러가 아니라 기존 결과 반환. DB 제약이 최종 방어선.
- Redis는 1차 필터(없어도 동작), DB가 진실. 장애가 결제로 전파되지 않게.
- redirect 파라미터(브라우저 경유)는 불신 → PG 승인/조회 API로 실제 금액 재확인 후 PAID.
- 존재 보장은 unique 제약(gap lock 회피), 계산 판단은 PK 단일 행 FOR UPDATE. (결정 15)
- 외부 API 호출은 트랜잭션 밖, payment+order 쓰기는 한 트랜잭션 안. (결정 16)
- UNKNOWN은 "확인 중" 안내 + 재시도 차단, 추후(또는 단건) 대사로 확정. (결정 14)

---

## 부록: 현재 코드 대비 변경점 (Claude Code 작업 가이드)

### `PaymentReadyService.readyPayment`
- merchantPayKey를 `order.assignMerchantPayKey`로 Order에 저장하던 것 → **Payment(RESERVED) 행 생성 시 서버 발급**으로 이동. Order에서 merchantPayKey 컬럼 제거.
- 키만 반환하던 것 → **RESERVED 행 생성/재사용** 추가(`findReusableReserved` + expiresAt + provider 조건, reserved_key set).
- `buildReturnUrl(merchantPayKey)`는 유지하되, 역조회는 Order가 아니라 Payment를 키로 조회.

### `PaymentAttempt`(→ `Payment`) 엔티티
- 이름 `Payment`로 통일.
- `orderId` 컬럼 추가(Order PK 값으로 참조, DB FK 제약은 걸지 않음).
- `pgPaymentId` **nullable**로(RESERVE 단계엔 없음) — 또는 RESERVE를 별도 테이블로 분리(열린 항목).
- `type`에 **RESERVE** 추가, `status`에 **RESERVED / UNKNOWN**(필요 시 EXPIRED) 추가.
- `expiresAt`, `approved_order_key`, `reserved_key` 추가, `createReserved(...)` 팩토리 추가.
- 부분 unique 대체 인덱스 2개 추가(`uk_approved`, `uk_reserved`) + 조건 컬럼 set/NULL을 상태 전이 메서드(`succeed`/`fail`/`expire`)에 캡슐화. 기존 4컬럼 unique는 필요 시 유지.
- `failCode`/`failDetail` 유지(결정 11). `failDetail` length 제한.

---

## 지금은 하지 않는 것 / 열린 항목 (승격 트리거)

| 항목 | 현재 | 승격/결정 조건 |
|---|---|---|
| 집계(완료상태) 테이블 `PaymentSummary` | 안 만듦 | 부분취소 도입 + 잔액 SUM이 잦고 무거울 때 → 집계 캐시로 |
| PG 원문 보관 테이블 | 로그로 대체(원문 로그 보존 필수) | 분쟁/CS 증가, 대사 자동화 필요, 결제량 규모 초과 → `PgTransactionLog`로 |
| 부분취소 로직 | 모델만 열어둠(amount+SUM), 구현은 전액만 | 부분취소 요구 시 — 입력/검증(과다취소 lock)/PG호출/UI만 추가 |
| 결제 취소 자체 | 미구현 | 현재 플로우 검증되면 추가. 성공 CANCEL → 같은 트랜잭션 order 취소(결정 17) |
| 대사(reconciliation) | 보류(원칙만: UNKNOWN 흔적) | UNKNOWN 단건 조회는 권장. 전체 자동 배치는 분쟁/규모 커질 때 |
| 클라이언트 idempotencyKey | 안 씀(reserve는 order unique로 충분) | 결제 시작 API 중복 호출이 실제 문제될 때 → Redis(1차)+DB unique |
| RESERVE 저장 위치 (A/B) | **미정** | A: Payment에 type=RESERVE 추가(pgPaymentId nullable 감수). B: 별도 Reservation 테이블(TTL)로 노이즈 분리·Payment 깔끔 |
| reserve의 네이버 예약 API 호출 여부 | 미확정 | (a) 서버가 네이버 예약 API 호출해 reserveId 수신 → 만료 RESERVED는 **신규 행**(기존 EXPIRED 전이 후) / (b) 서버가 키만 발급 → in-place 갱신 가능. 이게 만료 재발급 방식과 반환 필드를 확정 |
| RESERVED batch 삭제 | 보류 | expiresAt 조회 필터로 충분. 테이블 비대 시 청소 batch |
| MSA 이벤트화 | 안 함(모놀리식, 단 FK 물리 제약은 이미 미사용) | 결제 분리 시 — 동기 호출을 이벤트(결과적 일관성)로 + Outbox/Saga (참조 방식은 그대로) |

> **RESERVE 저장 위치(A vs B) 권장**: 현재 엔티티가 이미 `type`(APPROVE/CANCEL)을 갖고 있어 **A(type=RESERVE 추가)가 자연스럽다.** 다만 "ready 노이즈를 시도와 섞기 싫다" + "pgPaymentId not-null 유지"가 중요하면 **B(별도 Reservation)**. 둘 다 정합성엔 차이 없고 노이즈 분리/스키마 취향의 문제.

---

## 한 줄 요약

> Order는 불변 식별자(id/orderNumber)와 memberId만 갖고 결제를 모른다. 결제는 **Payment 단일 테이블**(시도 단위, append-only, Order PK를 `Long` 값으로 참조 — DB FK 제약은 걸지 않음)로 표현하고, merchantPayKey는 **서버가 ready 때 발급**해 RESERVED 행에 둔다. **MySQL이라 partial unique index가 안 되므로**, 이중결제(`approved_order_key`)와 reserve 따닥(`reserved_key`)은 "조건 만족 시에만 값·아니면 NULL" 컬럼 + 일반 unique로 대체한다(NULL은 중복 허용). ready 따닥은 그 unique + **expiresAt 재사용/만료**(박제 자동 복구)로, 이중결제는 그 unique + 보상취소로 막는다. 결제 완료는 마지막 행이 아니라 **조건 존재(성공 APPROVE ∧ 무효화 CANCEL 부재)**. 동시성은 **존재 보장=unique(gap lock 회피), 계산 판단=PK 단일 행 lock**, 외부 호출은 트랜잭션 밖·DB 쓰기는 한 트랜잭션 안. 타임아웃은 **UNKNOWN으로 남겨 "확인 중" 안내 + 재시도 차단**. 실패 사유(failCode/failDetail)는 DB 컬럼으로 유지하고, 원문 전체·집계 테이블·부분취소 로직은 모델만 열어두고 지금은 만들지 않는다. 모든 방어는 **1차 필터 + 최종 방어선(DB 제약)** 이중 구조.
