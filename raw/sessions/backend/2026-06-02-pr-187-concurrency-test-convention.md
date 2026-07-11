---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 187 }
---

# 동시성 테스트 작성 규칙 정립 — "타이밍을 운에 맡기지 않는다"

AI에게 테스트 작성을 맡겼을 때 `Thread.sleep`으로 타이밍 맞추기, race 무대 컴포넌트의 분기 결과를 stub으로 강제하기 같은 안티패턴이 반복 출현하는 게 컨벤션화의 트리거였다. 한두 번은 일회성 지시로 잡을 수 있지만, 같은 패턴이 계속 보이면 매번 같은 토론을 되풀이하지 않도록 컨벤션으로 박아야 한다. 이슈 #186으로 "동시성 테스트 작성 컨벤션 정립 + 기존 테스트 분류"를 메타이슈로 열고, PR #187에서 테스트 컨벤션 문서에 "동시성 테스트 작성 규칙" 섹션을 신규 추가하면서 위반 테스트를 정리했다. 이 세션에는 그 규칙에 이른 *사고 과정*을 남긴다 — 규칙 본문 자체는 프로젝트의 테스트 컨벤션 문서가 정본이다.

## 결정한 것

### 핵심 원칙: 타이밍을 운에 맡기지 않는다 — 두 갈래로 분기

처음엔 "타이밍을 제거한다"만 헤드라인으로 잡았는데, 그러면 latch/seam으로 인터리빙을 의도적으로 통제하는 접근과 어긋난다는 다른 AI 검토 피드백을 받아 원칙을 **두 갈래로 나눴다**:

- **제거** — 결과를 인터리빙과 무관하게 만든다. N thread를 동시에 던지고, 어떤 순서로 섞이든 성립하는 **불변식만 단언**한다("재고는 음수가 안 됨", "성공은 정확히 1건" 같은 식). 이게 기본이자 광범위 race 검증용.
- **통제** — 인터리빙을 결정론적으로 확정한다. 알려진 race 버그를 회귀로 박제할 때만 쓰는 예외 패턴. seam/hook으로 순서·타이밍을 고정한다.

자연적인 race에 의존하기(`Thread.sleep`으로 타이밍 맞추기, latch로 thread 간 *속도* 맞추기, 그냥 동시에 돌려서 어쩌다 일어난 걸 단언하기)는 전부 안티패턴이다.

### "race의 무대인가"가 stub 판단 기준

초안에서는 "DB, 락, application service는 stub하지 않는다"처럼 **컴포넌트 종류**로 기준을 잡았다. 사용자 피드백을 받아 **"검증하려는 race가 그 컴포넌트에서 일어나는가"**라는 기준으로 전환했다. 그 컴포넌트가 race의 무대면 진짜를 쓰고(stub하는 순간 무대가 사라져 검증이 무의미해진다), 무대 밖이면 mock한다. 예를 들어 PG gateway는 그 race의 무대가 아니라 mock해도 되고, idempotency store 같은 인프라도 race 검증 대상이 아니면 mock 가능하다. 판단은 컴포넌트 종류가 아니라 "무대인가"로 한다.

### `willAnswer`의 "관찰 vs 결정" 경계

받은 검토 피드백 중 가장 정확했던 지적. 같은 `willAnswer`라도 무엇을 하느냐로 갈린다:

- **관찰 (OK)**: `AtomicInteger`로 호출 횟수를 세는 것 — 외부 호출을 *기록*만 한다.
- **결정 (금지)**: `doReturn(false).when(service).hasStock()` — race 무대의 결과를 *가짜로 정한다*.

이 한 줄이 "mock 응답은 thread-safe하게 작성하라(권장)"와 "race 무대 컴포넌트의 분기 결과를 stub으로 강제하지 마라(금지)"의 양립을 명확하게 만들었다. mock이 race를 관찰만 하면 되고, 결과를 대신 결정하기 시작하면 그건 이미 동시성 검증이 아니다.

### 통제 패턴의 결과 단언 한계 — window는 강제, 결과는 invariant

데드락이나 락 타임아웃처럼 결과가 DB 스케줄러에 달린 경우, 어느 thread가 winner가 될지는 결정론으로 단언할 수 없다. 그래서 **인터리빙 window 자체는 결정론적으로 강제하되, 결과는 invariant로만 단언**하는 hybrid가 정상이다. 주문 생성 비관적 락의 데드락을 검증하는 통합 테스트(`OrderConcurrencyServiceDeadlockTest`)의 latch hook 예시가 정확히 이 형태다:

- 재고 감소 서비스(`DecreaseStockService.decrease` — 주문 생성 시 재고를 차감하며 비관적 락을 잡는 지점)에 `willAnswer`를 걸되 `callRealMethod()`로 **실제 로직은 그대로 실행**하고, 개수 2짜리 `CountDownLatch`로 양쪽 thread가 각자 첫 락을 잡을 때까지 서로 대기시켜 "반대 순서로 락을 잡는 window"를 결정론적으로 강제한다. (응답을 강제하는 게 아니라 순서만 확정한다.)
- 결과는 `errors.isNotEmpty()`(적어도 하나는 실패)와 `orderCount < 2L`(둘 다 성공할 수는 없음) 같은 invariant로 단언한다. DisplayName도 "반대 순서로 락을 잡는 동시 요청에서 양쪽 다 성공하지 못하고 적어도 한 주문은 실패한다"로 정리해, 특정 예외를 결정론으로 단언하는 대신 불변식을 서술하도록 맞췄다. `assertThatThrownBy`로 특정 예외를 못박으면 flaky가 된다.

### stress/soak는 컨벤션 범위 밖

부하·성능 측정은 JUnit/Spring 영역이 아니라 k6, Gatling 같은 전용 부하 도구의 영역이다. 별도 트랙으로 분리하고, 이 컨벤션은 결정론적 동시성 검증만 다룬다.

### 위반 케이스 처리: 둘 다 삭제

네이버페이 결제 승인 흐름의 동시성 통합 테스트(`NaverPayServiceConcurrencyTest`)에서 새 규칙을 어기는 케이스 2건을 삭제했다. 둘 다 race 무대인 결제 승인 서비스(`PaymentApprovalService`)를 spy로 잡아 그 결과를 강제하고 있었다:

- **`completeApprovedPayment`를 `doThrow`로 강제하던 케이스**(동시 중복 결제 보상 취소 경로에서 cancel attempt가 하나만 생성되는지 보던 테스트) → 자연 race로 발생하는 DUPLICATE 시나리오는 같은 클래스의 다른 테스트들이 이미 커버한다.
- **`isCompensationRequired`를 `doReturn(false)`로 강제하던 케이스**(다른 thread가 이미 Payment를 완료해 보상 cancel이 skip되는지 보던 테스트) → 같은 시나리오를 결제 승인 보상 로직 단위 테스트(`PaymentApprovalCompensationServiceTest`)가 mock으로 검증하고 있다.

진짜 race 시나리오로 재설계하는 것도 가능하지만(다른 thread가 먼저 Payment를 commit하는 timing을 통제해야 함), 단위 테스트가 이미 검증하는 시나리오라 동시성 테스트로서의 가치가 낮았다 — 거기까지 가서 얻는 가치보다 비용이 크다고 봐 삭제를 택했다. 함께, 위 데드락 테스트의 DisplayName도 결과를 invariant로 서술하는 새 형태에 맞게 정리했다.

## 배운 것

### flaky의 본질 재정의

처음엔 "자연 race = flaky"로 단정했다. 검토 피드백을 받고 정정했다: flaky의 원인은 **인터리빙에 의존하는 단언**이지 자연 race 자체가 아니다. 인터리빙과 무관한 invariant를 단언하면 자연 race로도 안정적이다 — "재고는 음수가 안 됨", "성공은 정확히 1건" 같은 식.

### 진짜 race-safety vs 분기 시뮬레이션

삭제한 위반 케이스들은 사실 *분기 결과 시뮬레이션*이었지 동시성 검증이 아니었다. 이런 건 단위 테스트가 더 적합하다. 동시성 자원(DB, 락)이 살아있는 상태에서 N thread를 던지고 invariant를 보는 게 진짜 race-safety 검증이다.

### 테스트로 잡을 일이 아닌 것

타이밍에 따라 결과가 달라진다는 건 **로직이 race에 노출됐다**는 신호다. 답은 "모든 타이밍을 테스트하자"가 아니라 "타이밍과 무관하게 설계하자"다. 락 / 유니크 제약 / 원자적 UPDATE(`SET qty = qty - 1 WHERE qty >= 1`) / 트랜잭션 경계 재설계로 위험한 인터리빙을 애초에 *불가능*하게 만들고, 테스트는 그 설계가 작동하는지 invariant로 확인한다.
