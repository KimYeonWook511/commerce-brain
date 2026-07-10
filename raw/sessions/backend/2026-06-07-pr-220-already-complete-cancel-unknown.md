---
platform: backend
author: KimYeonWook511
created: 2026-06-07
origin:
  - { type: pr, repo: commerce-backend, ref: 220 }
---

# PR #220 — 결제 결과 불명 시 UNKNOWN 보존을 AlreadyComplete·cancel 경로로 확장 (#219)

## 한 일
- 이슈 #219. "PG 호출 결과가 불명일 때 FAILED로 단정하지 말고 UNKNOWN으로 보존한다"는 원칙을, 직전 PR #218이 approve **직접 승인 경로**에만 적용하고 미뤄뒀던 두 경로로 확장:
  - **AlreadyComplete 후 이력 재확인**: PG가 approve 호출에 "이미 처리됨(AlreadyComplete)"으로 답하면, 승인 상세(금액·merchantPayKey)를 모르므로 `getApprovalHistory`(이력 조회)로 결과를 재확인한다. 이 이력 조회가 결과 불명이면 UNKNOWN.
  - **cancel(보상 취소)**: 승인 보상 흐름에서 PG 취소 호출이 결과 불명이면 cancel 기록을 UNKNOWN.
- 결과 타입 3종(이력조회 결과 / PG 취소 결과 / 보상 흐름의 취소 결과)에 UNKNOWN 추가. `markUnknownIfRequested`(REQUESTED 상태인 결제 기록만 UNKNOWN 마킹)를 cancel 기록용으로 신설.
- 코드리뷰 후속 2건도 같은 PR에서 처리(아래 "결정한 것" 참조).

## 결정한 것
- **분류축 = "재시도 안전성 = PG가 그 요청을 처리했을 가능성"**. 처리했을 수 있으면 UNKNOWN(재시도 차단 → 이중결제 방어), 처리 안 했음이 확실하면 FAILED.
  - 결과 불명류 = 네트워크 오류 / PG 5xx / 응답 해석 불가. 요청은 나갔는데 결과를 확정 못 함 → PG가 처리했을 수 있음 → UNKNOWN.
  - AlreadyComplete는 PG가 멱등하게 "이미 됨"을 준 상태라, 이 지점에서 이력 조회가 불명이면 UNKNOWN 원칙이 가장 강하게 적용돼야 함. 여기서 FAILED로 두면 결제 기록이 REQUESTED로 남아, "결과 불명 결제가 있으면 재시도 차단"하는 가드(`existsUnknownByOrderId`)를 우회 → 사용자가 재시도 → PG가 이미 승인했다면 이중결제.
- **외부 응답 이상은 catch(NPE)가 아니라 명시적 null 체크로 가른다.** PG 응답 파싱 객체를 다루는 영역(gateway)에서 body/list/원소/merchantPayKey가 null이면 그 자리에서 UNKNOWN(또는 적절히 매핑)으로 반환하고, 그래도 남는 예상 못 한 NPE는 잡지 말고 전역 예외 핸들러(500)로 전파한다. 이건 #218이 approve에 세운 컨벤션이고, 이번에 history/cancel도 같은 결로 맞춤.
- **merchantPayKey: 누락(null) vs 값이 다름을 구분.** 승인 이력인데 merchantPayKey가 **누락(null)** = 외부 응답 이상 → UNKNOWN. **존재하나 우리 키와 다름** = PG가 다른 결제 건을 승인했다는 확정 정보 = 진짜 키 불일치 → FAILED. (주의: `payment.getMerchantPayKey().equals(null)`은 false라, 누락을 그냥 두면 "키 불일치 FAILED"로 오분류된다 — gateway에서 null이면 UNKNOWN으로 먼저 걸러야 함.)
- **빈 이력 목록 = FAILED(NOT_FOUND).** 목록 자체가 null(해석 불가)이면 UNKNOWN이지만, 목록이 비어 있음 = "PG에 기록 없음"이라는 확정 결과라 FAILED.
- cancel의 UNKNOWN 기록은 주문 재결제를 막지 않는다 — 재시도 차단 가드(`existsUnknownByOrderId`)는 승인(APPROVE) 타입 결과 불명만 검사하고 취소(CANCEL) 타입은 안 보기 때문. 취소 UNKNOWN은 대사(reconciliation) 흔적으로만 남는다.
- ADR 정본: `commerce-backend/docs/adr.md`의 ADR-028(이번 PR #219). 내 이해로는 이건 #218이 세운 "PG 요청 전송 시점을 경계로 전파(가시화) vs UNKNOWN 보존(이중결제 방어)을 가른다"는 원칙의 **적용 범위 확장**이지 새 정책 도입이 아님.

## 다음 단계
- cancel UNKNOWN의 자동 해소(보상 취소 실패 재시도, 정기 대사)는 이번 범위 밖 — 결제 도메인 배치/스케줄러 Epic(#208)으로 분리.
- 승인 이력인데 merchantPayKey 누락은 매우 드문 외부 이상. UNKNOWN으로 보존만 하고 자동 해소는 #208 대상.
