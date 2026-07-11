---
platform: backend
author: KimYeonWook511
created: 2026-06-05
origin:
  - { type: pr, repo: commerce-backend, ref: 205 }
---

# 결제 도메인 Order↔결제 경계 재설계 — 예약/사건 두 테이블 분리와 자동 구현 결함 4건 정정

payment 도메인에서 Order 와 결제 사이 책임 경계를 다시 그은 세션이다(Issue #174, PR #205). 결제를 두 테이블로 쪼갰다 — PaymentReservation(결제창을 띄우기 위한 준비물, 상태 RESERVED/USED/EXPIRED)과 Payment(PG 에 실제로 보낸 사건 단위의 append-only 기록, 타입 APPROVE/CANCEL). 그동안 Order 가 들고 있던 결제 식별자 발급·저장 책임을 PaymentReservation 으로 옮기고, 이중결제·예약 따닥(동시 중복요청)의 최종 방어선을 MySQL NULL 트릭 partial unique 두 개로 세웠다. 구현은 harness 워크플로우(execute.py 워커)로 자동 생성했고, 여러 차례 AI 코드리뷰를 돌려 자동 구현이 흘린 결함 4건을 계단식으로 잡아 닫았다. 대안 비교와 결정 배경의 상세 원본은 이 재설계의 ADR·회고 문서에 있다.

## 결정한 것

### 1. 단일 테이블 → 예약/사건 두 테이블 분리

처음엔 `tbl_payment` 한 테이블에 결제 유형(예약/승인/취소)을 `type` 컬럼으로 다 담으려 했다(초기 A안). 진행하면서 4가지 위화감이 누적됐다.

1. **본질이 다르다.** 예약(임시·만료·상태변화·따닥으로 대량 발생하는 영역)과 승인/취소(PG 에 보낸 뒤 결과가 확정되면 불변인 사건)는 성격이 다르다. 한 테이블에 두면 예약의 임시성이 사건 테이블을 오염시킨다.
2. **NOT NULL 보장을 포기하게 된다.** 예약 단계엔 `pg_payment_id`(PG 가 발급하는 결제 식별자)가 아직 없어서, 그 컬럼을 NULL 허용으로 완화하게 된다. 승인/취소만 보면 `pg_payment_id NOT NULL` 이 자연스러운 보장인데 예약 때문에 포기하는 셈.
3. **status 컬럼에 두 의미가 혼재한다.** 예약 상태(RESERVED/EXPIRED)와 사건 상태(REQUESTED/SUCCEEDED/FAILED/UNKNOWN)가 한 컬럼에 섞여, "결제 시도 몇 건?" 같은 단순 조회마다 `type != 'RESERVE'` 필터를 달아야 한다.
4. **"단일 테이블이 단순하다"는 착시였다.** 테이블 수는 줄지만 각 행의 NULL 컬럼(`expires_at`, `reserved_key`)과 쿼리 분기가 오히려 늘어난다. 테이블 내부 복잡도와 쿼리 복잡도가 올라간다.

- **결정:** 두 테이블(B안)로 쪼갠다. 그러자 `tbl_payment` 는 "PG 사건"만 담는 순수 append-only 테이블이 되어 `pg_payment_id NOT NULL` 이 복원되고, `Payment.status`(REQUESTED/SUCCEEDED/FAILED/UNKNOWN)와 `PaymentReservation.status`(RESERVED/USED/EXPIRED)가 각자 깔끔한 enum 으로 갈렸다.
- **함께 정리된 것:** 기존에 "성공 결제 1:1" 단위를 표현하던 옛 `Payment` 테이블은 폐기했다. 그 테이블이 들고 있던 사실(order_id·pgPaymentId·amount·approvedAt·provider)은 성공한 APPROVE 사건 행이 이미 다 갖고 있어 *성공 시도의 복사본*에 불과했기 때문. 결제 완료 판단도 "마지막 행"이 아니라 "성공 APPROVE 행이 존재하는가(EXISTS)"로 두어, 나중에 부분취소(성공 APPROVE 존재 AND 무효화 성공 CANCEL 부재)로 확장할 여지를 열어뒀다. 현재는 단순히 성공 APPROVE 존재 여부만 조회한다.
- **트레이드오프:** 테이블 +1. 그러나 스키마·의미 단순화, 쿼리 명확성, NOT NULL 회복의 이득이 더 크다고 봐 택했다.

### 2. merchantPayKey 발급/저장 책임을 Order → PaymentReservation 으로 이동

결제 식별자(merchantPayKey — PG redirect 가 되돌아올 때 우리 쪽 결제를 역조회하는 entry 키) 발급·저장 책임을 Order 에서 떼어 PaymentReservation 으로 옮겼다. Order 의 관련 컬럼·필드·메서드·조회 메서드(`assignMerchantPayKey`, `findByMerchantPayKey*` 등)를 전량 제거했다.

- **이유:** Order 의 책임은 "무엇을 얼마에 산다"인데, 결제 수단 식별자("그 돈을 어떻게 받는다")까지 들고 있던 건 책임 과적이다. 도메인이 다르다.
- **기존 코드의 부정합:** `Order.assignMerchantPayKey()` 는 null 일 때만 값을 넣는 멱등 setter 라 *주문당 키 하나*로 고정됐다. 재시도나 금액 변경 시 새 키가 필요한 가변 값을, 불변 성격의 Order 엔티티에 박아둔 모순이었다. 설계 문서엔 "amount 변경 시 새 키"라고 적혀 있었으나 실제 코드엔 새 키 발급 경로 자체가 없어, 문서와 코드가 어긋나 있었다.
- **효과:** Order 가 결제 수단을 모르게 되면서, 같은 주문의 다중 PG 재시도나 금액 변경이 *새 Reservation 발급*으로 자연스럽게 표현된다. redirect 역조회의 주인도 결제창 발급 entry 인 PaymentReservation 으로 일원화됐다.

### 3. NULL 트릭으로 partial unique 우회

MySQL InnoDB 는 PostgreSQL 의 조건부 unique 인덱스(`CREATE UNIQUE INDEX ... WHERE`)를 지원하지 않는다. 그래서 "조건 충족 시에만 값을 넣고 아니면 NULL"인 컬럼 + 일반 unique 제약으로 우회했다(InnoDB 가 NULL 을 unique 중복으로 치지 않는 특성 이용). 방어선 두 개를 세웠다.

- **성공 승인 유일성** — `tbl_payment.approved_order_key`(BIGINT NULL) + `uk_payment_approved_order_key`. 승인이 성공(APPROVE + SUCCEEDED)일 때만 `order_id` 를 넣고, 그 외엔 NULL.
- **예약 따닥 방지** — `tbl_payment_reservation.reserved_key`(VARCHAR(96) NULL) + `uk_payment_reservation_reserved_key`. RESERVED 상태일 때만 `"{order_id}:{provider}"` 를 넣고, USED/EXPIRED 면 NULL.
- **핵심은 캡슐화다.** 상태와 트릭 컬럼을 *같은 도메인 메서드 한 번의 호출에서 같은 UPDATE 로* 동시에 set 한다 — `Payment.succeed()` 는 `status=SUCCEEDED` 와 `approved_order_key=orderId` 를, `PaymentReservation.markUsed()`/`markExpired()` 는 `status` 와 `reserved_key=NULL` 을 함께 바꾼다. 둘이 따로 변경되면(예: 상태만 바꾸고 트릭 컬럼은 안 비움) unique 가 무력화돼 정합성이 무너진다. NULL 트릭은 SQL 마법이라, 우회 setter 없이 이 "한 메서드에서 두 필드 동시 변경"이 지켜져야 의미가 산다. 그래서 도메인 테스트로 "한 메서드 호출에서 두 필드 동시 변경"을 단언으로 박았다.

### 4. 성공 결제 unique 는 PG(provider) 를 안 가린다 — cross-provider 이중결제 차단 (의도)

`approved_order_key` 에는 provider 를 섞지 않고 `order_id` 만 넣는다. 즉 한 주문에 성공한 승인은 네이버·카카오 등 전 PG 를 통틀어 1건만 허용된다. 사용자가 여러 PG 에 동시에 결제를 보내 둘 다 승인되는 *cross-provider 이중결제*를 막는 게 목적이다. (예약 따닥용 `reserved_key` 는 반대로 `provider` 를 포함해 (주문, 수단)당 RESERVED 1개를 보장한다 — 방어하려는 대상이 다르기 때문.)

### 5. `/payments/ready` → `/payments/reserve` rename

의미가 흐려진 외부 API 이름을 정정했다. "ready" 보다 "reserve" 가 이 단계의 실제 의미(예약 생성/재사용)에 맞다. frontend 가 아직 미개발이라 하위 호환을 깨도 무비용인 구간이었고, 의미가 틀린 이름은 운영 트래픽이 붙을수록 정정 비용이 커지므로 지금 바로잡는 게 맞다고 판단했다.

## 막힌 점·해결 — 자동 구현 후 리뷰에서 계단식으로 드러난 결함 4건

execute.py 워커가 자동 구현한 결과를 여러 차례 AI 코드리뷰로 검증하며, 자동 구현이 흘린 결함을 계단식으로 잡았다. 각 리뷰 라운드마다 성격이 다른 결함이 드러났다(어느 리뷰어가 어떤 결에 강했는지의 비교는 같은 PR 의 harness 운영 메모에 별도 정리). 핵심 4개는 아래와 같고, 넷 다 후속 커밋으로 닫았다.

### 1. 취소 결제의 order_id 가 NULL 로 새던 문제 (초기 리뷰)

취소(CANCEL) 결제 행을 만드는 팩토리가 orderId 를 받지 않아, 취소 결제의 `order_id` 가 NULL 로 저장됐다. 스키마 문서엔 NOT NULL 인데 마이그레이션 SQL(NULL 허용)·엔티티·팩토리가 서로 따로 놀았다. 취소 흐름이 후속 task 라 *실제로 DB 에 저장하는 실행 경로가 테스트에 없어서* 회귀로 안 잡힌 잠복 결함이었다.

- **해결:** orderId 전파를 팩토리까지 이어주고, 마이그레이션 SQL 을 `NOT NULL` 로, 엔티티를 `nullable=false` 로 맞춰 네 지점(팩토리·SQL·엔티티·스키마 문서)의 정합을 복구했다. (커밋: "CANCEL Payment 생성 시 orderId 누락을 전파", "V6 마이그레이션의 order_id 를 NOT NULL 로 바로잡는다", "Payment.orderId 에 NOT NULL 제약을 명시".)

### 2. 이미 사용된 예약의 미완료 승인이 영구 차단되던 문제 (2차 리뷰)

같은 결제키로 redirect 가 다시 오면 기존 결제 결과를 멱등 응답하도록 설계했다(USED 상태 예약을 만나면 차단이 아니라 기존 결과를 200 으로 흡수). 그런데 초기 구현은 *성공한 승인 행(SUCCEEDED)만* 찾아 반환하고, 없으면 404(PAYMENT_NOT_FOUND)를 던졌다. PG 가 "처리 중(PROCESSING)"을 반환했거나 호출 직전 끊겨서 승인 시도가 미완료(REQUESTED) 상태로 남은 경우, redirect 재시도가 404 로 막혀 결제를 영영 못 끝낸다.

- **해결:** 기존 승인 시도를 찾아 상태별로 재처리하도록 고쳤다 — REQUESTED 면 PG 에 재확인, SUCCEEDED 면 멱등 응답, FAILED 면 실제 실패 사유 반환.
- **교훈:** "멱등 = 완료된 것 재반환"이라는 가정이 *진행 중* 상태를 빠뜨렸다.

### 3. 만료된 예약이 자리를 안 비워 재예약이 막히던 문제 (2차 리뷰 — 내 설계 맹점)

당시 설계 결정은 "만료된 예약은 EXPIRED 같은 상태로 따로 마킹하지 않고, 재사용 판단 시 만료 시각 필터로만 걸러낸다"였다. 마킹을 두면 "누가 언제 마킹하느냐"는 또 다른 박제 위험이 생긴다고 봤기 때문이다. 그런데 만료된 예약 행은 상태가 여전히 RESERVED 라 `reserved_key`(`"{order_id}:{provider}"`)를 계속 점유한다. 만료 시각 필터는 *재사용*만 막을 뿐, *새 발급*은 같은 키로 `uk_payment_reservation_reserved_key` 위반을 일으켜 **같은 주문이 영원히 재예약 불가**가 된다. "필터로 충분하다"가 재사용 한쪽만 본 결론이었던 것이다.

- **해결:** 만료 상태(EXPIRED)를 다시 도입하되, reserve 진입 시점에 만료/무효 예약을 그 자리에서 lazy 회수(status=EXPIRED + reserved_key=NULL)한 뒤 새로 발급하도록 했다. reserve 를 호출하는 요청이 *자기가 쓸 자리를 직접 정리*하므로, 초기에 우려했던 별도 배치/스케줄러의 박제 위험은 생기지 않는다. 금액 변경 같은 무효화도 같은 경로로 회수된다.
- **교훈:** 상태/필드를 단순화로 제거할 때 영향을 *모든 경로(재사용 + 재발급)에서* 따져야 한다. 한쪽만 보고 단순화하면 정합성 구멍이 생긴다.

### 4. 결과 불명 오류를 '실패'로 분류해 이중결제가 열려 있던 문제 (3차 리뷰 — 가장 심각)

PG 승인 호출의 timeout/네트워크 오류를 PG 클라이언트가 네트워크 예외로 감싸는데, 게이트웨이가 이를 '실패(FAILED)'로 반환하고 있었다(그 아래에 결과 불명(UNKNOWN)으로 보내는 분기가 있었지만, 위에서 다 잡혀 도달 못 하는 죽은 코드였다). timeout 은 *PG 가 승인을 처리했는지 불명*인데 '실패'로 기록하면, "그 주문에 결과 불명 결제가 있는지 검사해 재시도를 차단하는 로직"이 안 걸린다 → 재결제가 허용되고, PG 가 이미 승인한 상태였다면 **이중결제**가 난다.

- **해결:** 결과가 불명한 오류(네트워크/PG 서버 오류/응답 해석 불가)는 결과 불명(UNKNOWN)으로, 처리되지 않음이 확실한 거절(인증 실패/잘못된 요청)만 '실패(FAILED)'로 분류하도록 정정했다.
- **교훈:** 외부 호출 예외를 '실패'로 뭉뚱그리면 *결과 모름*과 *처리 안 됨*이 섞여 정합성이 깨진다 — 성공/실패 외에 '결과 모름(UNKNOWN)'이라는 세 번째 상태가 필요한 이유다.

## 배운 것

- **잠복 결함은 "후속 task 라 실행 경로가 테스트에 없는" 곳에 숨는다.** 취소 결제의 order_id, 취소 흐름 등. 수용 기준(단위/통합 테스트)을 통과해도 미구현 경로의 정합성은 안 잡힌다.
- **단순화의 영향은 모든 경로에서 따진다.** 만료 상태를 없앤 게 재사용 판단은 단순화했지만 재발급을 깼다.
- **외부 연동 예외 분류는 '결과 모름' vs '처리 안 됨'을 반드시 구분한다.** 돈이 두 번 빠지느냐가 여기서 갈린다.
- **여러 AI 리뷰어가 서로 다른 결을 잡는다.** 한 라운드로 끝내지 않고 여러 번 돌리니 성격이 다른 결함이 라운드마다 새로 드러났다. (어떤 리뷰어가 무엇에 강했는지의 비교는 같은 PR 의 harness 운영 메모에 별도 정리.)

## 미해결·열린 질문

- **PG 예외 분류 후속 정비 (#206)** — 공통 예외 정리 작업(#198)과 같은 PR 에서 처리 예정. 이중결제를 여는 핵심(결과 불명 오분류)은 닫았지만, 프로그래밍 버그(예: NPE)를 결과 불명이 아니라 시스템 예외(500)로 보내는 미세 조정이 남았다.
- **UNKNOWN 해소 대사 서비스(PaymentReconciliationService) — 미구현.** 이번 task 는 결과 불명 결제를 *마킹하고 차단*까지만 했고, *해소*(PG 단건 조회로 실제 결제됐는지 확인)는 후속으로 분리했다. 현재는 결과 불명 결제를 차단만 하고 풀지 않아서, 한 번 결과 불명이 찍힌 주문이 영영 안 풀릴 수 있다. 위 결함 수정들로 타이밍성 차단은 줄였지만, 진짜 통신 장애의 해소는 이 후속이 있어야 닫힌다.
- **취소(CANCEL) 흐름 실제 구현 / 부분취소** — 모델(Payment.type=CANCEL 행에 amount)만 열어두고 실제 구현은 안 했다.
