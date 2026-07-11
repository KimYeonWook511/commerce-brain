---
platform: backend
author: KimYeonWook511
created: 2026-06-07
origin:
  - { type: pr, repo: commerce-backend, ref: 220 }
---

# 결과 불명 시 UNKNOWN 보존을 AlreadyComplete 이력 재확인·cancel 경로로 확장

"PG 호출 결과가 불명일 때 FAILED로 단정하지 말고 UNKNOWN으로 보존한다"는 원칙(직전 PR #218이 세운 것)을,
그때 approve **직접 승인 경로**에만 적용하고 미뤄뒀던 두 경로 — PG가 "이미 처리됨(AlreadyComplete)"으로
답한 뒤의 이력 재확인, 그리고 승인 보상 흐름의 취소(cancel) — 로 같은 결로 넓힌 세션이다(이슈 #219).
새 정책을 도입한 게 아니라, #218이 세운 "PG 요청 전송 시점을 경계로 전파(가시화) vs UNKNOWN 보존(이중결제
방어)을 가른다"는 원칙의 **적용 범위 확장**이다.

배경의 위험은 이랬다. 확장 전 이력 조회부(`getApprovalHistory`, PG에 승인 이력을 되묻는 호출)는 네트워크
오류 같은 결과 불명 예외까지 전부 FAILED로 떨어뜨렸다. 그러면 AlreadyComplete(=PG가 멱등하게 "이미 됨"을
준 상태) 직후 이력 조회가 네트워크 오류로 실패했을 때 결제 기록이 REQUESTED로 남고 UNKNOWN 흔적이 안 남는다
→ "결과 불명 결제가 있으면 재시도 차단"하는 가드(`existsUnknownByOrderId`, 해당 주문에 UNKNOWN 결제 행이
있는지 검사)를 우회 → 사용자가 재시도 → PG가 이미 승인했다면 이중결제. cancel 쪽도 결과 불명을 FAILED로
박제하면, PG가 실제로 취소했더라도 cancel 기록이 FAILED로 남아 대사(reconciliation)에서 누락될 수 있었다.

## 결정한 것

### 1. 분류축은 "재시도 안전성 = PG가 그 요청을 처리했을 가능성"

#218이 세운 것과 동일한 축을 그대로 쓴다. PG가 처리했을 수 있으면 UNKNOWN(재시도 차단 → 이중결제 방어),
처리 안 했음이 확실하면 FAILED.

- **결과 불명류 = 네트워크 오류 / PG 5xx 서버오류 / 응답 해석 불가.** 요청은 나갔는데 결과를 확정 못 함
  → PG가 처리했을 수 있음 → UNKNOWN.
- **AlreadyComplete 지점에서 이 원칙이 가장 강하게 적용돼야 한다.** PG가 멱등하게 "이미 됨"을 준 상태라,
  여기서 이력 조회가 불명인데 FAILED로 두면 결제 기록이 REQUESTED로 남아 위의 재시도 차단 가드를 우회하고
  이중결제로 직결된다.

### 2. 두 경로로 UNKNOWN 보존 확장

- **AlreadyComplete 후 이력 재확인:** PG가 approve 호출에 "이미 처리됨"으로 답하면 승인 상세(금액·
  merchantPayKey=가맹점 결제 키)를 모르므로 이력 조회로 결과를 재확인한다. 이 이력 조회가 결과 불명이면
  이력 결과 타입에 새로 추가한 UNKNOWN으로 반환한다. AlreadyComplete를 처리하는 응용 경로
  (`processAlreadyComplete`)는 이력 결과가 UNKNOWN이면 REQUESTED 상태인 결제 기록에만 UNKNOWN을 마킹
  (`markUnknownIfRequested`)하고 `PAYMENT_RESULT_PENDING`(409, "결제 결과 확인 중")을 던진다 —
  approve 직접 경로의 UNKNOWN 처리와 완전히 같은 결.
- **cancel(보상 취소):** 승인 보상 흐름에서 PG 취소 호출이 결과 불명이면 취소 결과 타입에 UNKNOWN을 추가해
  전달하고, 보상 흐름이 cancel 기록을 UNKNOWN으로 보존한다.
- 결과 타입 3종 — 이력 조회 결과 / PG 취소 결과(gateway 계층) / 보상 흐름의 취소 결과(application 계층) —
  모두에 UNKNOWN 상태를 새로 넣었다. 그리고 REQUESTED 상태의 CANCEL 기록만 골라 UNKNOWN 마킹하는
  `markUnknownIfRequested`를 취소 기록용으로 신설했다(승인 기록용 동명 메서드는 #218에서 이미 있던 것).

### 3. 외부 응답 이상은 catch(NPE)가 아니라 명시적 null 체크로 가른다

PG 응답 파싱 객체를 다루는 영역(gateway)에서 응답 본문·이력 목록·마지막 원소·merchantPayKey가 null이면 그
자리에서 UNKNOWN으로 반환하고, 그래도 남는 예상 못 한 NPE는 잡지 말고 전역 예외 핸들러(500)로 전파한다.
이건 #218이 approve에 세운 컨벤션이고, 이번에 history/cancel도 같은 결로 맞췄다. (실제 코드 리뷰 후속으로,
`try { list.getLast() } catch (NullPointerException | NoSuchElementException)` 뭉치를 본문 null → UNKNOWN /
빈 목록 → FAILED / 원소 null → UNKNOWN 으로 각각 명시 분기하도록 갈랐다.)

### 4. merchantPayKey: 누락(null) vs 값이 다름을 구분

- **승인 이력인데 merchantPayKey가 누락(null) = 외부 응답 이상(해석 불가) → UNKNOWN.**
- **존재하나 우리 키와 다름 = PG가 다른 결제 건을 승인했다는 확정 정보 = 진짜 키 불일치 → FAILED**
  (실패 코드 `MERCHANT_PAY_KEY_MISMATCH`로 결제 기록을 실패 처리하고 응답은 not-found).

### 5. 빈 이력 목록 = FAILED(not-found)

목록 자체가 null(해석 불가)이면 UNKNOWN이지만, 목록이 비어 있음 = "PG에 기록 없음"이라는 확정 결과라 FAILED.
결과 불명(보존)과 확정 실패(전파)를 목록의 null 여부로 가르는 지점이다.

### 6. cancel의 UNKNOWN 기록은 주문 재결제를 막지 않는다

재시도 차단 가드(`existsUnknownByOrderId`)는 승인(APPROVE) 타입 결과 불명만 검사하고 취소(CANCEL) 타입은
안 본다. 그래서 취소 UNKNOWN은 주문을 재결제 불가로 brick하지 않고, 대사 흔적으로만 남는다. 이건 의도된
비대칭이다 — 승인 UNKNOWN은 이중결제 위험이라 차단해야 하지만, 취소 UNKNOWN은 "환불이 됐는지 불명"일 뿐
새 결제를 막을 이유가 없다.

## 막힌 점·해결

- **`equals(null)` 오분류 함정.** merchantPayKey 누락을 gateway에서 UNKNOWN으로 먼저 걸러내지 않으면,
  누락(null)이 그대로 키 비교 분기까지 내려가 `payment.getMerchantPayKey().equals(null)`이 false를 리턴하고
  "키 불일치 → FAILED"로 오분류된다. 누락(외부 응답 이상, UNKNOWN 대상)과 불일치(확정 정보, FAILED 대상)는
  전혀 다른 사건인데 한 갈래로 뭉개지는 것. 그래서 null 체크를 키 비교보다 **앞에** 두어 누락을 먼저
  UNKNOWN으로 빼내야 한다.

## 미해결·열린 질문

- **cancel UNKNOWN의 자동 해소(보상 취소 실패 재시도, 정기 대사)는 이번 범위 밖.** 결제 도메인 배치/
  스케줄러 Epic(#208)으로 분리했다. 지금은 UNKNOWN으로 보존해 대사 대상으로만 남긴다.
- **승인 이력인데 merchantPayKey 누락은 매우 드문 외부 이상.** UNKNOWN으로 보존만 하고 자동 해소는 마찬가지로
  #208 대상으로 미뤘다.
