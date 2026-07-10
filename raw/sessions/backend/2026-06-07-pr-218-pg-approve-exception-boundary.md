---
platform: backend
author: KimYeonWook511
created: 2026-06-07
origin:
  - { type: pr, repo: commerce-backend, ref: 218 }
---

## 한 일
- 이슈 #206 fix — NaverPay(PG) 결제 승인 경로의 예외 포착 범위를 좁혀, 프로그래밍 버그(NPE 등)가 결제 결과 UNKNOWN으로 흡수돼 주문이 영구 차단(brick)되던 문제 수정. PR #218 머지.
- 코드리뷰(gemini/codex/claude 3개 AI) 추가 발견을 같은 PR에서 반영. 한 갈래(AlreadyComplete 경로)는 #219로 분리.
- 같은 PR의 AI 운영 교훈은 [[raw/sessions/backend/2026-06-07-pr-218-multi-ai-review-lessons-ai]]에 별도 기록.

## 결정한 것 — 핵심: "요청 전송 시점"을 예외 처리 경계로
정본: docs/adr.md ADR-026(UNKNOWN 마킹/차단 정책) + ADR-027(전송 시점 경계), docs/exception-strategy.md "결제 결과 UNKNOWN 처리".

배경 제약: Payment에 UNKNOWN 상태가 있고 existsUnknownByOrderId(해당 주문에 결과 불명 결제가 있으면 재시도 차단)로 reserve/approve를 영구 차단한다. UNKNOWN 자동 해소(대사)는 미구현이라, 한 번 잘못 찍히면 수동 개입 전까지 결제 불가. (이 차단 정책의 정본이 ADR-026)

상반된 두 위험:
- (a) 프로그래밍 버그가 UNKNOWN으로 흡수 → 일시 오류 한 번에 brick.
- (b) 반대로 PG가 이미 승인했을 수 있는 상태의 예외를 그냥 전파 → UNKNOWN 흔적 안 남음 → 재결제 허용 → 이중결제.

내 이해 — 이 둘은 "요청 전송 시점"으로 가르면 양립한다:
- 전송 전 버그(요청 빌드 등): PG 부작용 0 → 전파해 안전망(500)으로 가시화. (a) 해결.
- 전송 후 / 전송 여부 불명: PG가 처리했을 수 있음 → UNKNOWN(client는 INVALID_RESPONSE)으로 보존. (b) 해결.
- 우선순위: 이중결제(돈 이중출금) > brick(차단). 애매하면 보존.

세부:
- 예외 분류 결정축은 errorCode 종류가 아니라 "재시도 안전성(PG 처리 가능성)". 세분화해도 FAILED(처리 안 함 확실)/UNKNOWN(처리 가능) 두 갈래로 수렴 → errorCode 5종(NETWORK/SERVER_ERROR/CLIENT_ERROR/AUTHENTICATION/INVALID_RESPONSE) 유지.
- UNKNOWN으로 보내는 errorCode는 NETWORK/SERVER_ERROR/INVALID_RESPONSE 3종(isResultUnknown), CLIENT_ERROR/AUTHENTICATION은 처리 안 됨 확실이라 FAILED.
- 알려진 외부 응답 이상(body/detail/merchantPayKey null, JSON 파싱 실패, RestTemplate 통신 계열 예외)만 도메인 결과로 보존. 그 외 예상 못 한 버그는: 우리 객체를 다루는 영역(NaverPayGatewayImpl 응답 분기)은 명시 null 체크 후 전파, 외부 JSON 파싱 영역(NaverPayClient.post 해석 단계)은 보존.
- PG가 Success(승인 확정) 응답인데 detail/merchantPayKey 비면 FAILED 아닌 UNKNOWN.
- "NPE는 안 잡는다"는 무조건이 아니라 전송 전 원칙. 전송 후는 정반대(이중결제 방어 위해 잡아 보존).

다시 본다면:
- 두 계층(client 변환 레이어, gateway 도메인 결과 매핑)에 같은 광범위 catch가 있어 한쪽만 고치면 다른 쪽에서 의도가 무력화됨(codex가 정확히 지적). 외부 연동 예외는 양쪽을 같은 원칙으로 봐야.
- NaverPayClient.post의 response.getBody()(HTTP raw 문자열)와 gateway의 response.getBody()(파싱된 JSON의 body 필드)는 이름만 같고 다른 레벨 → 두 null 체크 다 필요.

## 막힌 점 / 과정의 거칠음
- 설계 입장을 여러 번 번복: client catch를 RestClientException으로 좁힘 → codex의 "전송 후 버그 이중결제" 지적으로 catch(Exception)으로 넓힘 → "전송 시점 경계" 원칙 확정. 최종: client는 전송/해석 단계 분리 후 해석 단계 catch(Exception) 보존, gateway는 명시 null 체크+버그 전파. 빙 돌았지만 결론은 일관.

## 다음 단계
- #219 (분리한 후속): PG가 AlreadyComplete 응답 후 getApprovalHistory(결제내역 재조회)가 실패하면 현재 errorCode 구분 없이 FAILED로 떨어지고 markUnknown 없이 throw → Payment가 진행중 상태로 남아 차단 우회. 같은 "전송 후/불명 → UNKNOWN 보존" 원칙을 이 경로에도. cancel 경로도 함께. (AlreadyComplete는 PG 멱등 반환이라 직접 이중결제보다 "결제됐는데 미결제 박제" 정합성 문제)
