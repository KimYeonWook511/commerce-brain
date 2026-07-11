---
platform: backend
author: KimYeonWook511
created: 2026-06-07
origin:
  - { type: pr, repo: commerce-backend, ref: 218 }
---

# PG 승인 예외 포착 범위 축소 — "요청 전송 시점"을 경계로 버그 전파(가시화)와 UNKNOWN 보존(이중결제 방어)을 가른다

NaverPay(PG) 결제 승인 경로에 프로그래밍 버그(우리 코드의 NPE 등)까지 결제 결과 UNKNOWN으로 흡수해버리는 광범위한 `catch (Exception)`이 두 계층(client 변환 레이어, gateway 도메인 결과 매핑)에 걸려 있었다. UNKNOWN이 한 번 찍히면 그 주문의 reserve/approve가 영구 차단(brick)되고 자동 해소도 아직 없어서, 일시적 버그 한 번에 주문이 수동 개입 전까지 결제 불가로 막히는 문제(이슈 #206)를 고친 세션이다. 좁히기만 하면 반대 위험(이미 승인됐을 수 있는 예외를 그냥 전파 → 이중결제)이 생겨서, 어디까지 잡고 어디부터 전파할지의 경계를 "PG에 요청이 전송됐는가"로 잡았다. 이 PR에서 세 종류의 AI 코드 리뷰를 돌렸는데, 그 리뷰 운영 자체의 교훈은 별도 노트로 뺐고 여기서는 예외 경계 결정만 다룬다.

## 결정한 것 — 핵심: "요청 전송 시점"을 예외 처리 경계로

**배경 제약.** 결제 도메인 재설계에서 Payment에 UNKNOWN 상태를 도입했고, `existsUnknownByOrderId`(해당 주문에 결과 불명 결제가 있으면 재시도를 막는 판정)로 UNKNOWN 행이 있는 주문의 reserve/approve를 `PAYMENT_RESULT_PENDING`(409)로 영구 차단한다. UNKNOWN을 자동으로 풀어주는 대사(reconciliation)는 아직 미구현이라, 한 번 잘못 UNKNOWN으로 찍히면 수동 개입 전까지 그 주문은 결제 불가다. 즉 UNKNOWN은 함부로 남기면 안 되는 무거운 흔적이다.

이 제약 위에서 상반된 두 위험이 맞선다:
- **(a) brick:** 프로그래밍 버그가 UNKNOWN으로 흡수되면, 일시 오류 한 번에 주문이 영구 차단된다.
- **(b) 이중결제:** 반대로 PG가 이미 승인했을 수 있는 상태의 예외를 그냥 전파하면, UNKNOWN 흔적이 안 남아 `existsUnknownByOrderId` 차단을 우회해 재결제가 허용되고, PG가 정말 승인했다면 돈이 이중 출금된다.

- **핵심 통찰 — 두 위험은 "요청 전송 시점"으로 가르면 양립한다.**
  - **전송 전 버그**(요청 빌드 등): PG 부작용이 0이라 잡지 않고 전파해 안전망(500)으로 가시화한다. → (a) 해결.
  - **전송 후 / 전송 여부 불명**: PG가 이미 처리했을 수 있으니 UNKNOWN(승인 결과) / INVALID_RESPONSE(client 레벨)로 보존해 재시도를 차단한다. → (b) 해결.
  - **우선순위: 이중결제(돈 이중 출금) > brick(차단).** 애매하면 보존 쪽으로 기운다. 전송 시점은 이 두 요구를 동시에 만족시키는 유일한 경계다.

세부 결정:
- **분류 축은 errorCode 종류가 아니라 "재시도 안전성(PG 처리 가능성)".** 예외를 아무리 세분화해도 결국 FAILED(처리 안 함이 확실)와 UNKNOWN(처리됐을 수 있음) 두 갈래로 수렴한다. 그래서 PG 오류 분류 enum은 다섯 종(NETWORK / SERVER_ERROR / CLIENT_ERROR / AUTHENTICATION / INVALID_RESPONSE)을 그대로 두되, 그 다섯을 두 결과로 접는다.
- **UNKNOWN으로 보내는 건 NETWORK / SERVER_ERROR / INVALID_RESPONSE 세 종**(요청·응답 유실, PG 5xx 내부 처리 중 실패 가능, 응답은 왔으나 해석 불가 — "PG가 처리했을 수 있다"). **CLIENT_ERROR / AUTHENTICATION은 요청이 거절돼 처리 안 됨이 확실하므로 FAILED.** (판정은 `isResultUnknown`.)
- **알려진 외부 응답 이상만 도메인 결과로 보존한다.** 응답 본문 null, 파싱된 응답의 `body`/`detail`/`merchantPayKey` null, JSON 파싱 실패, RestTemplate 통신 계열 예외가 그 목록이다. 그 외 예상 못 한 버그는 영역별로 갈린다:
  - **우리 객체를 다루는 영역**(gateway의 응답 분기, `NaverPayGatewayImpl`)은 알려진 이상을 명시적 null 체크로 처리하고, 그 외 예상 못 한 버그는 전파한다. gateway 끝에 있던 후행 `catch (Exception) → UNKNOWN`을 제거한 게 이 결정의 몸통이다.
  - **외부 JSON 파싱 영역**(client의 응답 해석 단계, `NaverPayClient.post`)은 예외를 잡아 INVALID_RESPONSE로 보존한다. 여기서 터지는 건 우리가 통제 못 하는 외부 문자열을 우리 타입으로 역직렬화하다 나는 것이라, 전송 후일 수 있어 보존이 맞다.
- **PG가 Success(승인 확정)로 응답했는데 `detail` 또는 `merchantPayKey`가 비면 FAILED가 아니라 UNKNOWN.** 승인은 처리된 게 확실한데 본문만 빈 상황이라, FAILED로 두면 재결제가 열려 이중결제가 난다. 원래 이 경로는 `NullPointerException`을 잡아 INVALID_RESPONSE FAILED로 떨어뜨렸는데, 명시적 null 체크 후 UNKNOWN 보존으로 바꿨다.
- **"NPE는 안 잡는다"는 무조건이 아니라 전송 전에 한한 원칙이다.** 전송 후에는 정반대 — 이중결제를 막으려고 오히려 잡아서 보존한다. 같은 NPE라도 전송 시점에 따라 처리가 뒤집힌다.

`NaverPayClient.post`는 이 원칙에 맞춰 **전송 단계와 해석 단계를 분리**했다. 요청 빌드는 `try` 밖으로 빼 전송 전 버그가 그대로 전파되게 하고, `postForEntity` 호출부터가 전송 단계다 — 그 안에서 나는 예외는 실제 소켓 write 여부를 코드로 구분할 수 없어 "전송 여부 불명"으로 보고 INVALID_RESPONSE로 보존한다. 전송이 끝난 뒤 응답 본문 null 체크와 역직렬화는 별도 블록으로 내려, 여기서 나는 모든 예외(프로그래밍 버그 포함)도 INVALID_RESPONSE로 보존한다.

**같은 PR에서 반영/분리한 것.** 세 AI 코드 리뷰가 짚은 추가 지적은 이 PR에서 함께 반영했고, 한 갈래(AlreadyComplete 응답 후 이력 재확인 경로)는 범위가 커서 후속(#219)으로 분리했다.

## 막힌 점·해결

### 설계 입장을 여러 번 번복 — 결론은 일관, 과정은 빙 돌았다

경계를 잡기까지 client의 catch 범위를 반대 방향으로 두 번 뒤집었다:
- 처음엔 client의 광범위 catch를 통신 예외(`RestClientException`) 계열로 **좁혔다**. "외부 통신 오류만 도메인 결과로 보존하면 된다"는 판단.
- AI 코드 리뷰가 **"전송 후 응답 해석 중 버그가 나면 그건 좁힌 catch를 빠져나가 전파되고, 그러면 UNKNOWN 흔적 없이 재결제가 열려 이중결제"**라고 지적. 이걸 받고 다시 `catch (Exception)`으로 **넓혔다**.
- 좁혔다 넓혔다를 겪고 나서야 진짜 기준이 범위가 아니라 **"전송 시점"**임을 확정했다. 최종형은 client를 전송 단계와 해석 단계로 쪼갠 뒤 해석 단계에만 `catch (Exception)` 보존을 두고, gateway는 알려진 이상만 명시 null 체크하고 나머지 버그는 전파. 빙 돌았지만 결론은 처음부터 끝까지 "전송 후는 보존, 전송 전은 전파"로 일관했다.

### 두 계층에 같은 광범위 catch가 걸려 한쪽만 고치면 무력화된다

client 변환 레이어와 gateway 도메인 결과 매핑, **두 곳에 같은 성격의 광범위 catch**가 있었다. AI 코드 리뷰가 정확히 짚은 함정은 — 한쪽만 좁히면 다른 쪽의 넓은 catch가 그 예외를 도로 UNKNOWN으로 흡수해 의도가 무력화된다는 것이다. 외부 연동 예외는 두 계층을 **같은 원칙(전송 시점 경계)으로 함께** 봐야 한다는 게 이 지적의 교훈이고, 그래서 client와 gateway를 한 PR에서 같이 고쳤다.

### 이름만 같고 레벨이 다른 두 `getBody()` — null 체크가 양쪽 다 필요

`NaverPayClient.post`의 `response.getBody()`(HTTP 응답의 raw 문자열, `ResponseEntity<String>`)와 gateway의 `response.getBody()`(역직렬화된 응답 객체의 `body` 필드)는 **이름만 같고 완전히 다른 레벨**이다. 전자가 null이면 응답 자체가 비었다는 것이고, 후자가 null이면 JSON은 왔는데 우리가 기대한 필드가 없다는 것이다. 둘은 다른 실패라 **양쪽 다 null 체크가 필요**하다 — 한쪽만 막으면 다른 레벨의 빈 응답이 새어 나간다.

## 배운 것

- **같은 예외라도 "부작용이 이미 발생했을 수 있는가"로 처리가 갈린다.** NPE 하나도 전송 전이면 전파해 가시화, 전송 후면 잡아서 보존 — 예외 타입이 아니라 부작용 가능성이 결정축이다. 되돌릴 수 없는 외부 부작용(결제)을 다루는 경계에서는 예외 분류를 타입 기준으로 짜면 틀린다.
- **비대칭 위험에서는 "애매하면 어느 쪽으로 기울지"를 먼저 못 박아야 설계가 수렴한다.** 여기선 이중결제 > brick이라 회색지대는 무조건 보존(UNKNOWN)으로 정했고, 그 우선순위가 catch 범위 논쟁을 끝냈다.
- **같은 관심사가 두 계층에 흩어져 있으면 한쪽만 고치는 건 반쪽짜리다.** 광범위 catch가 client·gateway 양쪽에 있으면 아래 계층이 위 계층의 좁히기를 되돌려버린다 — 외부 연동 예외 정책은 계층을 가로질러 한 원칙으로 봐야 한다.
- **이름이 같은 심볼이 다른 레벨을 가리킬 때가 실수 지점이다.** 두 `getBody()`처럼, 같은 이름이 raw 문자열과 파싱된 필드를 각각 가리키면 한쪽 null 체크로 다른 쪽을 막았다고 착각하기 쉽다.

## 미해결·열린 질문

- **AlreadyComplete 경로의 동일 일관화(후속 #219로 분리).** PG가 AlreadyComplete로 응답한 뒤 `getApprovalHistory`(결제 이력 재조회)로 결과를 재확인하는데, 그 재조회가 실패하면 지금은 errorCode 구분 없이 FAILED로 떨어지고 `markUnknown` 없이 throw한다. 그러면 Payment가 진행중 상태로 남아 UNKNOWN 차단을 우회하는 구멍이 생긴다. 여기에도 "전송 후 / 결과 불명 → UNKNOWN 보존" 원칙을 똑같이 적용해야 하고, cancel(보상 취소) 경로도 같이 봐야 한다. 다만 AlreadyComplete는 PG의 멱등 반환이라 성격이 조금 다르다 — 직접적인 이중결제라기보다 "결제는 됐는데 미결제로 박제되는" 정합성 문제 쪽이다.
- **전송 회색지대는 즉시 500 가시화를 일부 양보한다.** `postForEntity` 진입 후에는 소켓 write 여부를 코드로 구분할 수 없어 전송 여부 불명으로 보고 보존하므로, 전송 후에 난 *예상 못 한* 버그는 즉시 500으로 뜨지 않고 UNKNOWN/INVALID_RESPONSE에 섞인다. 대신 원본 예외를 cause로 보존해 스택트레이스 로그와 INVALID_RESPONSE 모니터링으로 추적은 가능하다. 알려진 외부 이상을 다 명시 처리했으니 이 회색지대에 실제로 남는 케이스는 극히 드물 것으로 봤지만, 완전히 없앤 건 아니다.
