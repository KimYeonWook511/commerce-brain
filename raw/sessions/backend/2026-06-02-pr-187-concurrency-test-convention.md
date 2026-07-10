---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 187 }
---

# 동시성 테스트 작성 규칙 정립 (#186, PR #187)

AI 에게 테스트 작성을 맡겼을 때 `Thread.sleep` / 분기 stub 강제 같은 안티패턴이 반복 출현하는 게 컨벤션화의 트리거였다. 한두 번은 일회성 지시로 잡지만 패턴이 보이면 컨벤션으로 박아 매번 같은 토론을 안 하게 만든다.

## 한 일

- 이슈 #186 생성: 동시성 테스트 작성 컨벤션 정립 + 기존 테스트 분류 메타이슈
- PR #187 (draft): `docs/testing-conventions.md` 에 "동시성 테스트 작성 규칙" 섹션 신규 추가 / `NaverPayServiceConcurrencyTest` 의 컨벤션 위반 케이스 삭제 / `OrderConcurrencyServiceDeadlockTest` 의 DisplayName 정리
- 기존 `@Tag("concurrency")` 테스트 전수 점검 후 컨벤션 패턴별 분류

## 결정한 것

컨벤션 본문은 정본 `commerce-backend/docs/testing-conventions.md` 의 "동시성 테스트 작성 규칙" 섹션. 여기엔 *결정에 이른 사고 과정*만 남긴다.

### 핵심 원칙: "타이밍을 운에 맡기지 않는다"

처음엔 "타이밍을 제거한다" 만 헤드라인이었는데, 패턴 2 (latch/seam 으로 timing 통제) 와 어긋난다는 다른 Claude 피드백을 받아 **두 갈래로 분기**:
- **제거** — 결과를 인터리빙과 무관하게 만든다 (패턴 1, 불변식 단언)
- **통제** — 인터리빙을 결정론적으로 확정한다 (패턴 2, seam/hook)

자연 race 의존 / `Thread.sleep` / latch 로 속도 맞추기는 안티패턴.

### "race 의 무대인가" 가 stub 판단 기준

초안에선 "DB, 락, application service 는 stub 하지 않는다" 로 *컴포넌트 종류* 기준이었음. 사용자 피드백으로 **"검증하려는 race 가 그 컴포넌트에서 일어나는가"** 로 전환. 무대 위면 진짜, 무대 밖이면 mock. PG gateway 는 mock OK, idempotency store 도 race 검증 대상 아니면 mock 가능.

### `willAnswer` 의 "관찰 vs 결정" 경계

다른 Claude 피드백 중 가장 정확했던 지적. 같은 `willAnswer` 라도:
- **관찰 (OK)**: `AtomicInteger` 로 호출 횟수 세기 — 외부 호출을 *기록*만
- **결정 (금지)**: `doReturn(false).when(service).hasStock()` — race 무대의 결과를 *가짜로 정함*

이 한 줄이 "thread-safe mock 권장"과 "분기 stub 강제 금지" 의 양립을 명확하게 만듦.

### 패턴 2 의 결과 단언 한계

DB 스케줄러 의존 결과(데드락/락 타임아웃) 는 어느 thread 가 winner 인지 결정론으로 단언 불가. **window 는 결정론적으로 강제, 결과는 invariant 로 단언** 하는 hybrid 가 정상. `OrderConcurrencyServiceDeadlockTest` 의 latch hook 예시가 정확히 이 형태:
- latch 로 양쪽 thread 가 각자 첫 락 잡을 때까지 대기 → window 강제
- 결과는 `errors.isNotEmpty() / orderCount < 2L` 같은 invariant

### stress/soak 는 컨벤션 범위 밖

JUnit/Spring 영역이 아니라 k6, Gatling 같은 부하 도구의 영역. 별도 트랙으로 분리.

### 위반 케이스 처리: 둘 다 삭제

NaverPayServiceConcurrencyTest 의 위반 케이스 2건:
- `completeApprovedPayment` 를 `doThrow` 로 강제하는 케이스 → 자연 race DUPLICATE 시나리오는 같은 클래스의 다른 테스트가 cover
- `isCompensationRequired` 를 `doReturn(false)` 강제하는 케이스 → 단위 테스트 `PaymentApprovalCompensationServiceTest` 가 같은 시나리오를 mock 으로 검증

진짜 race 시나리오로 재설계도 가능하지만, 단위 테스트가 이미 검증하는 시나리오라 동시성 가치가 낮음. 자연 race 로 만들려면 다른 thread 가 먼저 Payment commit 하는 timing 통제 필요한데 그것까지 가서 얻는 가치 < 비용.

## 배운 것

### flaky 의 본질 재정의

처음엔 "자연 race = flaky" 로 단정. 다른 Claude 피드백을 받고 정정: **인터리빙 의존적 단언이 flaky 의 원인**이지 자연 race 자체가 아님. 인터리빙 무관 invariant 단언이면 자연 race 로도 안정적이다 — "재고는 음수 안 됨", "성공은 정확히 1건" 같은 식.

### 진짜 race-safety vs 분기 시뮬레이션

기존 NaverPayServiceConcurrencyTest 의 위반 케이스들은 사실 *분기 결과 시뮬레이션*이었지 동시성 검증이 아니었음. 이런 건 단위 테스트가 더 적합. 동시성 자원(DB, 락)이 살아있는 상태에서 N thread 던지고 invariant 보는 게 진짜 race-safety 검증.

### 테스트로 잡을 일이 아닌 것

타이밍에 따라 결과가 달라진다는 건 **로직이 race 에 노출됐다**는 신호. "모든 타이밍을 테스트하자" 가 아니라 "타이밍과 무관하게 설계하자" 가 답. 락 / 유니크 제약 / 원자적 UPDATE / 트랜잭션 경계 재설계로 위험한 인터리빙을 *불가능*하게 만들고, 테스트는 그 설계가 작동하는지 invariant 로 확인.

## 참고

- 컨벤션 본문: `commerce-backend/docs/testing-conventions.md` "동시성 테스트 작성 규칙" 섹션
- 관련 이슈/PR: #186, PR #187
