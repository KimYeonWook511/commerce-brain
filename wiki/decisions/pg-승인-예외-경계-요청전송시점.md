---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, payment, pg-gateway, naverpay, double-payment, unknown-status, idempotency, reconciliation, adapter, external-integration]
created: 2026-06-07
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-07-pr-218-pg-approve-exception-boundary]]"
---

# PG 승인 예외 포착 범위 축소 — "요청 전송 시점"을 경계로 버그 전파와 UNKNOWN 보존을 가른다

## 컨텍스트 — 광범위 catch가 버그를 UNKNOWN으로 흡수 (brick)

NaverPay(PG) 결제 승인 경로에, 프로그래밍 버그(우리 코드의 NPE 등)까지 결제 결과 UNKNOWN으로 흡수해버리는 광범위한 `catch (Exception)`이 두 계층(client 변환 레이어, gateway 도메인 결과 매핑)에 걸려 있었다(Issue #206 / PR #218).

배경 제약: 결제 도메인 재설계에서 Payment에 UNKNOWN 상태를 도입했고, `existsUnknownByOrderId`가 UNKNOWN 행이 있는 주문의 reserve/approve를 `PAYMENT_RESULT_PENDING`(409)로 영구 차단한다([[payment-unknown-결과불명-처리와-예외분류]]). UNKNOWN을 자동으로 푸는 대사(reconciliation)는 아직 미구현이라, 한 번 잘못 UNKNOWN으로 찍히면 수동 개입 전까지 그 주문은 결제 불가(brick)다. 즉 UNKNOWN은 함부로 남기면 안 되는 무거운 흔적이다.

## 상반된 두 위험 — brick vs 이중결제, 우선순위 못 박기

- **(a) brick:** 프로그래밍 버그가 UNKNOWN으로 흡수되면, 일시 오류 한 번에 주문이 영구 차단된다.
- **(b) 이중결제:** 반대로 PG가 이미 승인했을 수 있는 상태의 예외를 그냥 전파하면, UNKNOWN 흔적이 안 남아 `existsUnknownByOrderId` 차단을 우회해 재결제가 허용되고, PG가 정말 승인했다면 돈이 이중 출금된다.

좁히기만 하면 (b)가, 넓히기만 하면 (a)가 산다. 비대칭 위험이라 **애매하면 어느 쪽으로 기울지를 먼저 못 박았다: 이중결제(돈 이중 출금) > brick(차단).** 회색지대는 무조건 보존(UNKNOWN)으로 정했고, 그 우선순위가 catch 범위 논쟁을 끝냈다.

## 핵심 결정 — "요청 전송 시점"이 두 요구를 동시에 만족하는 경계

두 위험은 "PG에 요청이 전송됐는가"로 가르면 양립한다.

- **전송 전 버그**(요청 빌드 등): PG 부작용이 0이라 잡지 않고 전파해 안전망(500)으로 가시화한다. → (a) 해결.
- **전송 후 / 전송 여부 불명**: PG가 이미 처리했을 수 있으니 UNKNOWN(승인 결과) / INVALID_RESPONSE(client 레벨)로 보존해 재시도를 차단한다. → (b) 해결.

전송 시점은 이 두 요구를 동시에 만족시키는 유일한 경계다.

## 세부 결정 — 5종→2결과 접기, UNKNOWN 3종, Success+빈본문=UNKNOWN, NPE 뒤집힘

- **분류 축은 errorCode 종류가 아니라 "재시도 안전성(PG 처리 가능성)".** 예외를 아무리 세분화해도 결국 FAILED(처리 안 함 확실)와 UNKNOWN(처리됐을 수 있음) 두 갈래로 수렴한다. PG 오류 분류 enum 다섯 종(NETWORK / SERVER_ERROR / CLIENT_ERROR / AUTHENTICATION / INVALID_RESPONSE)은 그대로 두되 두 결과로 접는다.
- **UNKNOWN으로 보내는 건 NETWORK / SERVER_ERROR / INVALID_RESPONSE 세 종**(요청·응답 유실, PG 5xx 내부 처리 중 실패 가능, 응답은 왔으나 해석 불가). **CLIENT_ERROR / AUTHENTICATION은 요청이 거절돼 처리 안 됨이 확실하므로 FAILED.** (판정 `isResultUnknown`.)
- **알려진 외부 응답 이상만 도메인 결과로 보존한다.** 응답 본문 null, 파싱된 응답의 `body`/`detail`/`merchantPayKey` null, JSON 파싱 실패, RestTemplate 통신 예외가 그 목록. 그 외 예상 못 한 버그는 영역별로 갈린다:
  - **우리 객체를 다루는 영역**(gateway 응답 분기, `NaverPayGatewayImpl`): 알려진 이상만 명시 null 체크, 그 외 버그는 전파. gateway 끝의 후행 `catch (Exception) → UNKNOWN` 제거가 이 결정의 몸통.
  - **외부 JSON 파싱 영역**(client 응답 해석 단계, `NaverPayClient.post`): 예외를 잡아 INVALID_RESPONSE로 보존. 통제 못 하는 외부 문자열 역직렬화라 전송 후일 수 있어 보존이 맞다.
- **PG가 Success(승인 확정)로 응답했는데 `detail`/`merchantPayKey`가 비면 FAILED가 아니라 UNKNOWN.** 승인은 처리됐는데 본문만 빈 상황이라 FAILED로 두면 재결제가 열려 이중결제. 원래 `NullPointerException`을 잡아 INVALID_RESPONSE FAILED로 떨어뜨리던 걸 명시 null 체크 후 UNKNOWN 보존으로 바꿨다.
- **"NPE는 안 잡는다"는 무조건이 아니라 전송 전에 한한 원칙이다.** 전송 후에는 정반대 — 이중결제를 막으려고 오히려 잡아 보존한다. 같은 NPE라도 전송 시점에 따라 처리가 뒤집힌다. 예외 타입이 아니라 부작용 가능성이 결정축.

## client 전송 단계/해석 단계 분리

`NaverPayClient.post`는 이 원칙에 맞춰 전송 단계와 해석 단계를 분리했다. 요청 빌드는 `try` 밖으로 빼 전송 전 버그가 그대로 전파되게 하고, `postForEntity` 호출부터가 전송 단계다 — 그 안의 예외는 실제 소켓 write 여부를 코드로 구분할 수 없어 "전송 여부 불명"으로 보고 INVALID_RESPONSE로 보존한다. 전송 뒤 응답 본문 null 체크와 역직렬화는 별도 블록으로 내려, 여기서 나는 모든 예외(프로그래밍 버그 포함)도 INVALID_RESPONSE로 보존한다.

## 막힌 점 — 입장 두 번 번복, 두 계층 동시 수정 필요, 이름만 같은 두 getBody()

- **설계 입장을 반대 방향으로 두 번 뒤집었다.** 처음엔 client 광범위 catch를 통신 예외(`RestClientException`) 계열로 좁혔다 → AI 코드 리뷰가 "전송 후 응답 해석 중 버그가 좁힌 catch를 빠져나가 전파되면 UNKNOWN 흔적 없이 재결제가 열려 이중결제"라고 지적 → 다시 `catch (Exception)`으로 넓힘 → 좁혔다 넓혔다 끝에 진짜 기준이 범위가 아니라 **"전송 시점"**임을 확정. 결론은 처음부터 끝까지 "전송 후는 보존, 전송 전은 전파"로 일관, 과정만 빙 돌았다.
- **두 계층에 같은 광범위 catch가 있어 한쪽만 고치면 무력화된다.** 한쪽만 좁히면 다른 쪽 넓은 catch가 그 예외를 도로 UNKNOWN으로 흡수한다. 외부 연동 예외 정책은 계층을 가로질러 한 원칙으로 봐야 해서 client·gateway를 한 PR에서 같이 고쳤다.
- **이름만 같고 레벨이 다른 두 `getBody()`.** client의 `response.getBody()`(HTTP raw 문자열, `ResponseEntity<String>`)와 gateway의 `response.getBody()`(역직렬화된 응답 객체의 `body` 필드)는 이름만 같고 완전히 다른 레벨이라 양쪽 다 null 체크가 필요하다 — 한쪽만 막으면 다른 레벨의 빈 응답이 샌다.

## 트레이드오프·미해결

- **AlreadyComplete 경로의 동일 일관화(후속 #219로 분리).** PG가 AlreadyComplete로 응답한 뒤 `getApprovalHistory` 재조회가 실패하면 지금은 errorCode 구분 없이 FAILED로 떨어지고 `markUnknown` 없이 throw해, Payment가 진행중으로 남아 UNKNOWN 차단을 우회하는 구멍이 있다. 같은 "전송 후 / 결과 불명 → UNKNOWN 보존" 원칙을 적용해야 하고 cancel(보상 취소) 경로도 함께 봐야 한다. 다만 AlreadyComplete는 PG의 멱등 반환이라 성격이 조금 달라 직접 이중결제라기보다 "결제는 됐는데 미결제로 박제되는" 정합성 문제 쪽 — 이 후속이 [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]].
- **전송 회색지대는 즉시 500 가시화를 일부 양보한다.** `postForEntity` 진입 후엔 소켓 write 여부를 구분 못 해 전송 후 *예상 못 한* 버그가 즉시 500이 아니라 UNKNOWN/INVALID_RESPONSE에 섞인다. 원본 예외를 cause로 보존해 스택트레이스·INVALID_RESPONSE 모니터링으로 추적은 가능. 알려진 외부 이상을 다 명시 처리해 실제 남는 케이스는 극히 드물지만 완전히 없앤 건 아니다.
- 관련: 같은 결제 도메인의 이중결제 adapter 매핑·보상 부채는 [[결제승인완료-보상-완료우선-이중결제-adapter매핑]], UNKNOWN 회수 주체인 대사는 [[결제-후처리-대상식별-status중심-재설계]], 상류 장애(UPSTREAM_ERROR/502) 카테고리는 [[도메인-예외-httpstatus-제거-errorcategory]]. 이 PR의 세 AI 코드리뷰 운영 교훈은 [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]으로 분리했다.

## 근거

- [[raw/sessions/backend/2026-06-07-pr-218-pg-approve-exception-boundary]]
