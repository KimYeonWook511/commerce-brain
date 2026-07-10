---
platform: backend
author: KimYeonWook511
created: 2026-06-10
origin:
  - { type: pr, repo: commerce-backend, ref: 237 }
---

## 맥락
- PR #237 대사 작업의 코드리뷰 대응·문서 동기화 중 정리한, 대사 동작의 본질과 미확정 차단의 정합성 구조. 같은 PR #237의 상태 모델·결제-주문 결합·리뷰 발견은 별도 메모.

## 대사가 PG에 실제로 하는 것 — 승인 재요청은 없다
- NaverPay gateway는 approve(승인)·getApprovalHistory(이력 조회)·cancel(취소) 3종.
- 대사가 PG에 보내는 건 **조회(getApprovalHistory) + 필요 시 보상 취소(cancel)뿐. approve(승인 재요청)는 절대 하지 않는다.**
- 이유: 대사 대상은 "approve를 이미 PG에 보냈는데 응답을 못 받은(UNKNOWN) / 응답 저장 전 끊긴(stale REQUESTED)" 결제다. 여기서 approve를 또 보내면 이중 승인 = 이중 과금.
- 그래서 재승인 대신 getApprovalHistory로 "그 approve가 PG에서 실제 성공했는지"를 조회만 한다.
  - 이력이 APPROVED → **확정 직전에 실시간 승인과 동일하게 PG 이력의 merchantPayKey·금액을 재검증**하고, 통과하면 우리 DB만 SUCCEEDED로 확정한다(PG 호출 없이 결제를 SUCCEEDED·주문을 PAID로 기록만 맞춤). 대사라고 검증을 건너뛰지 않는다(실시간 경로와 대칭).
  - 이력상 PG에서 취소됨 / 이력 자체가 없음 → FAILED로 확정.
- cancel(보상 취소)을 호출하는 경우 — PG에 실제 과금된 거래가 있을 때 그걸 환불한다: 승인 확정했는데 주문이 이미 CANCELED / 위 재검증의 금액 불일치 / 중복 결제(주문 PAID + 다른 성공 결제 존재).
- 반면 **merchantPayKey 불일치는 PG 취소를 하지 않고 FAILED로 격리한다** — PG에 우리 키로 승인된 거래가 없어 취소할 대상 자체가 없기 때문이고, 자동으로 안 풀리는 사람-개입 신호(공격 시도/데이터 정합성 위반)다. (정본 결정: commerce-backend/docs/adr.md — 키/금액 불일치 격리, 보상된 승인은 새 상태 없이 FAILED+failCode로 표현)
- 핵심: 대사는 "PG에 이미 일어난 사실을 조회해 우리 기록을 거기 맞추는 것"이지 새 거래를 만들지 않는다. 이게 '대사(장부 맞추기)'라 부르는 이유이자 이중과금 방어의 핵심이다.

## 미확정 차단과 대사의 정합성 구조 — over-blocking은 종착 한 곳으로 풀린다
- 만료 차단 쿼리(미확정 결제 걸린 주문을 만료에서 제외)는 **시간 상한이 없다** — 결제 status가 UNKNOWN/REQUESTED인 한 무조건 차단한다. 반면 대사 스캔은 약 6시간 상한이 있다(그보다 오래된 건은 escalation으로 자동 대사에서 제외).
- 그래서 6시간 초과 미확정 결제가 걸린 주문은 자동 대사도 안 되고 만료도 영구 차단된다(over-blocking). 코드리뷰가 이를 가장 심각한 항목으로 지적했다.
- 그러나 이 갭은 만료 차단 대상에 REQUESTED를 추가한 이번 변경 이전부터 UNKNOWN에 동일하게 있던 것이고, 이번 변경이 대상을 REQUESTED로 넓혔을 뿐 새로 만든 게 아니다.
- 구조적으로 한 곳에서 풀린다: escalation 종착(후속 #238)이 6시간 초과 건의 status를 UNKNOWN/REQUESTED 밖으로 확정하면, 만료 차단 조건(status가 UNKNOWN/REQUESTED일 때만 차단)에서 빠져 만료 차단이 자동 해제되고 대사 재시도도 멈춘다. 종착 한 곳이 차단과 대사를 동시에 해소한다. (정본 결정: commerce-backend/docs/adr.md — escalation은 새 상태 대신 스캔 시간 윈도우 상한으로 자동 제외, 운영 통지는 NotificationPort hook만, 보상·escalation 종착에 새 결제 상태 미도입)
- 차단 쿼리에 6시간 상한을 넣으면 안 된다 — 6시간 넘은 미확정이 실제로 PG에 과금됐을 수 있는데 그 주문을 만료시키면 돈이 샌다(애초에 차단하는 이유가 그것).
- escalation 분기는 이미 WARN 로그(paymentId·orderId 포함)로 관측 가능하다.

## starvation의 본질 — 정책은 이미 분리, 스캔 쿼리만 어긋났다
- 진입 지연 정책은 이미 UNKNOWN 약 1분 / REQUESTED 약 15분으로 분리돼 있었다(REQUESTED는 승인 가능 시간 10분 + 마진).
- 그런데 대사 스캔 쿼리만 둘 다 1분 하한으로 긁어서, 1~15분 사이 REQUESTED가 id 오름차순 첫 페이지를 차지하고 정책에서 "아직 대상 아님"으로 매 주기 버려져 뒤의 실제 후보가 고사(starvation)했다.
- 즉 버그의 본질은 "진입 지연 정책과 스캔 쿼리 하한의 불일치"였고, 스캔을 정책에 맞춰 REQUESTED 15분 하한으로 분리해 해소했다. 정책과 스캔 쿼리가 같은 상수를 단일 출처로 공유하게 했다.

## 다음 단계
- escalation 종착·통지(#238)가 over-blocking을 닫는 핵심.
- 단일 테이블 결합 부채: 승인(APPROVE)과 취소(CANCEL)를 한 테이블(tbl_payment)에 type 컬럼으로 묶어둔 설계가 — 모든 쿼리에 type 필터 강제, 스캔 모집단 부풀음, 정책 분기 비대화, 락 경합 확대로 연쇄한다. 스캔 쿼리 인덱스 부재(풀스캔+filesort)와 함께 #239에 기록. 인덱스로 단기 완화 vs 테이블 분리로 근본 해소의 트레이드오프.
