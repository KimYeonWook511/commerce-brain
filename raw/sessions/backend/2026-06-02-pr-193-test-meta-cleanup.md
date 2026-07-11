---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 193 }
---

# 테스트 운영 메타 정비 — @Tag 두 축 모델 도입과 CI 잡 분리

테스트가 늘면서 "어떤 환경이 있어야 도는가"와 "어떤 격리 그룹에 속하는가"가 한 태그에 뭉쳐
있었다. 이걸 두 축으로 갈라 `@Tag`를 다차원으로 재편하고, Gradle 테스트 태스크 이름·구성과 CI
잡을 그 모델에 맞춰 정리한 세션이다. 태그 차원 정비(#177), 학습·디버그 테스트 격리(#190),
Spring 빈 Mock 어노테이션 규칙(#191) 세 갈래를 한 PR로 묶었고, 정비 과정에서 그동안 CI가 놓치고
있던 배치 테스트 회귀 하나를 같은 PR에서 잡았다.

## 결정한 것

### 1. 단일 차원 vs 다차원 — 다차원 선택

`@Tag`를 하나의 평면 라벨로 두는 단일 차원 대신, **환경 요구 축**(docker/sandbox — 어떤 환경이
있어야 돌릴 수 있는가)과 **격리 분류 축**(concurrency/batch/learning — 어떤 격리 그룹에 속하는가)
두 축으로 분리했다. 한 클래스가 두 축에 각각 값을 가질 수 있다(예: `docker + concurrency`).

- **처음엔 단일 차원이 매력적이었다.** 이유는 두 가지. (1) drift 우려 — 코드에 `@Testcontainers`가
  이미 있는데 `@Tag("docker")`까지 달면 같은 정보를 두 군데 적는 중복이다. (2) 한 클래스에 태그를
  N개 붙이는 인지 부담.
- **그러나 현실이 이미 다차원이었다.** `docker + concurrency`가 함께 부여돼야 맞는 클래스가 실제로
  존재했다(동시성 테스트인데 Testcontainers도 필요). 단일 라벨로는 이 둘을 못 담는다.
- **결정적 이유는 CI 잡 분리에 docker 축이 필요하다는 것.** 단위/슬라이스 잡과 도커 통합 잡을
  가르려면 "이 클래스가 도커를 요구하는가"를 태그로 알아야 한다. 단일 차원으로 가면 그 구분이
  클래스명·패키지 컨벤션으로 새서 오히려 더 복잡해진다.
- **drift 위험은 컨벤션 문서 + PR 리뷰로 잡는 수준이 적당하다고 봤다.** 태그 자동화까지 가지 않고,
  테스트 컨벤션 문서의 태그 규칙 절을 정본으로 두고 리뷰에서 확인하는 선에서 감수했다.

### 2. 테스트 태스크 이름·구성 정리

Gradle 태스크를 새 태그 모델에 맞춰 재정의했다.

- **`ciTest` 제거**: CI 전용으로 "일반 + 도커"를 한 번에 돌리던 태스크였는데, CI를 잡 두 개로
  쪼개면서(아래 "CI 잡을 단위/슬라이스 · 통합/배치 둘로 분리") 존재 이유가 사라졌다.
- **`dockerTest` → `integrationTest`**: 도커는 "환경 요구"일 뿐이고 이 태스크의 진짜 의도는
  "통합 테스트를 기본 test 태스크와 격리해 돌린다"이다. 의도에 맞는 이름으로 바꿨다.
- **`naverPaySandboxTest` → `sandboxTest`**: 지금은 NaverPay뿐이지만 PG가 추가될 가능성을 고려해
  일반화했다.
- **`batchTest` 신설**: Spring Batch 통합 테스트를 별도 격리 태스크로 뽑았다(아래 "batchTest 신설").
- 기본 `test` 태스크의 excludeTags를 새 축 어휘로 갱신했다 — `concurrency`, `batch`, `docker`,
  `sandbox`, `learning`을 제외해 단위/슬라이스만 남긴다(도커 불필요, 가장 빠름).

### 3. batchTest 신설 + CI에는 포함

배치 테스트는 격리하되 CI에서는 돌린다.

- **격리 사유는 명확하다.** 클래스가 하나라도, 배치 컨텍스트 기동 비용이 커서 기본 test 태스크와
  섞이면 안 된다.
- **별도 잡으로 더 쪼개진 않았다.** CI에서 `integrationTest`와 같은 잡(통합/배치)에서 함께 호출한다.
  배치 클래스 수가 적어, 잡을 또 나누면 잡 워밍업 오버헤드가 절약 시간보다 크다.

### 4. CI 잡을 단위/슬라이스 · 통합/배치 둘로 분리

CI 워크플로를 두 개의 독립 잡으로 갈랐다.

- `unit-slice`: `./gradlew test` — 도커 없이 빠른 피드백.
- `integration`: `./gradlew integrationTest batchTest` — Testcontainers 필요.
- **두 잡은 병렬로 돈다.** 서로 독립한 top-level 잡이라 한쪽이 실패해도 다른 쪽은 끝까지 돌아
  양쪽 결과를 다 본다(한쪽 실패로 다른 쪽을 취소하지 않는다).

### 5. concurrency / sandbox는 CI 미포함 (수동 정책)

- **동시성**: flaky 가능성 + 실행 시간이 길다. 동시성 회귀를 PR 단계에서 자동으로 못 잡는 비용을
  감수하고 수동 실행으로 둔다.
- **sandbox**: 외부 환경에 실제 API 호출이 나간다. 정말 필요할 때만 수동으로 돌린다.

### 6. 학습·디버그 테스트 격리 — "이중 안전망" (@Tag + @Disabled)

학습/디버그 목적의 테스트는 운영 검증 대상이 아니지만 회고용으로 보존한다. 이를 위해 옛
`@Tag("test")`를 `@Tag("learning")`으로 교체하고, 격리 대상에 `@Tag("learning")`과
`@Disabled`를 **동시에** 부여했다(예: 주문 동시성 흐름을 `Thread.sleep` 등으로 관찰하던 디버그용
클래스).

- **왜 둘 다인가.** 태그만으로는 새 클래스를 쓸 때 `@Tag("learning")`을 빠뜨리면 노출된다.
  `@Disabled`만으로는 빌드 태스크 분리가 안 된다(모든 태스크가 후보로 잡은 뒤 skip). 둘 다 있으면
  한쪽을 빠뜨려도 안전하고, 태스크 영역에서 아예 빠진다.
- 모든 test 태스크의 excludeTags에 `learning`이 들어가 `@Disabled`와 이중으로 겹친다.
- 학습용 클래스는 별도 모듈로 분리하지 않고 `src/test` 위치를 유지한다. 운영 코드 리팩토링 시
  IDE가 컴파일 깨짐을 알려주는 안전망 역할을 겸하기 때문. (폴더 분리 비용 대비 이득 부족 —
  `@Tag + @Disabled`만으로 격리 효과는 이미 동일.)

### 7. 동시성 테스트 명명 통일은 무리 — D 하나만 정리

당시 동시성 테스트 이름이 4가지 패턴으로 갈려 있었다.

- **A** `*ConcurrencyTest` — Cart/Stock(`CartConcurrencyTest`, `StockConcurrencyTest`)
- **B** `*ServiceConcurrencyTest` — NaverPay/Payment 4개(`NaverPayServiceConcurrencyTest`,
  `PaymentApprovalServiceConcurrencyTest`, `PaymentApprovalAttemptServiceConcurrencyTest`,
  `PaymentCancellationAttemptServiceConcurrencyTest`)
- **C** `OrderConcurrencyService*Test` — Order 도메인(`OrderConcurrencyServiceTest`,
  `...DeadlockTest`, `...DeadlockMysqlTest` 등)
- **D** `*ConcurrencyIntegrationTest` — 딱 1개(`OrderCreateConcurrencyIntegrationTest`)

- **패턴 통일은 무리라고 봤다.** 특히 C는 카테고리 라벨이 아니라 **도메인 네이밍**이다 —
  `OrderConcurrencyService`가 실제 서비스 이름이라, "Concurrency"가 테스트 분류가 아니라 대상
  클래스 이름의 일부다. 억지로 통일하면 뜻이 어긋난다.
- **D 하나만 정리했다**: `OrderCreateConcurrencyIntegrationTest` → `OrderCreateServiceConcurrencyTest`
  로 rename해 패턴 B에 합류시켰다.

### 8. 베이스 클래스 미도입 재확인

슬라이스/통합 양쪽 모두 공통 베이스 클래스를 만들지 않기로 다시 확인했다. `@DynamicPropertySource`로
각 테스트가 필요한 컨테이너(MySQL/Redis/Kafka)만 명시 선언하는 패턴을 유지한다 — 그래야 각 테스트의
의존 범위가 한눈에 드러나고 변경 영향 추적이 쉽다. 공통 베이스로 묶으면 모든 통합 테스트가 모든
컨테이너에 엮이거나 조합별 베이스 트리가 생겨 의존 범위가 흐려진다.

## 막힌 점·해결

### rebase 후 배치 테스트 회귀 발견 → 같은 PR에서 fix

- **증상**: 만료 배치를 두 번 돌려도 두 번째엔 아무것도 추가 처리하지 않아야 한다는 테스트
  (`OrderExpirationBatchTest`의 "같은 조건으로 배치를 다시 실행해도 추가 처리하지 않는다")에서,
  재고 복원 아웃박스 이벤트 생성 호출 횟수 단언이 틀려 있었다. 만료 주문 2개를 setup한 뒤 첫 배치가
  둘 다 취소하며 이벤트 생성을 2회 부르는데, 단언은 `times(1)`로 박혀 있었다.
- **왜 그동안 안 터졌나**: develop CI가 batch 태그를 excludeTags로 빼고 있어 이 배치 테스트가 CI에서
  아예 안 돌았다. 그 사이 단언 오류가 누적됐다. 이번 PR의 CI 잡 분리가 `batchTest`를 CI(통합/배치
  잡)에 처음 포함시키면서 비로소 드러났다.
- **fix**: `times(1)` → `times(2)` 한 줄.
- **의미**: "CI에 자동 검증을 포함시키는 변경"의 가치가 첫 발동에서 곧바로 나타났다 — 회귀 자동
  발견. 이 PR의 의도(배치를 CI에 넣는다)가 직접 효과를 본 첫 사례다.

### docker+concurrency 클래스의 CI 자동 검증 공백

새 태스크 모델을 disjoint하게(한 클래스는 정확히 0 또는 1개 태스크에서만 실행) 구성하다 보니,
격리 축 태그가 붙은 클래스는 환경 축이 docker라도 격리 태스크 쪽으로만 매칭돼 CI에서 빠진다.

- `integrationTest`는 excludeTags에 concurrency가 있어 제외.
- `concurrencyTest`는 CI 미포함(수동 정책).
- **결과: docker+concurrency 6개 클래스가 CI 자동 검증에서 빠진다** —
  `OrderConcurrencyServiceDeadlockMysqlTest`, `OrderCreateServiceConcurrencyTest`,
  `PaymentApprovalServiceConcurrencyTest`, `PaymentApprovalAttemptServiceConcurrencyTest`,
  `PaymentCancellationAttemptServiceConcurrencyTest`, `NaverPayServiceConcurrencyTest`.
- **이건 의도된 상태(수동 정책)**다. 다만 "어떤 변경 시 수동 검증이 필요한가"를 테스트 컨벤션 문서의
  CI 자동 검증 범위 절에 명시해 인지 비용을 문서로 해소했다.

## 배운 것

- **Tag 두 축의 진짜 직교성.** 처음엔 두 축이 안 직교로 보였다("둘 다 결국 격리 사유 라벨 아닌가").
  격리 사유 관점으로 다시 보면 갈린다 — docker/sandbox는 "환경만 있으면 일반 흐름대로 돌아도 OK",
  concurrency/batch/learning은 "환경이 있어도 격리가 필수". 즉 **격리 *종류* 차원**에서 진짜 직교다.
- **CI 자동 검증 공백은 그 자체보다 인지 비용이 문제.** 단순히 "CI에서 안 돈다"가 아니라, "어떤
  변경을 하면 수동 검증이 필요한지"가 문서에 명시돼야 다음 세션이 그걸 인지한다. 테스트 컨벤션
  문서의 CI 자동 검증 범위 절이 그 핵심이다.
- **자동 리뷰가 "모델 빈틈"까지 짚는다.** `integrationTest`의 excludeTags에 `sandbox`를 추가하라는
  disjoint 제안이 자동 코드 리뷰에서 나왔다 — 매칭되는 클래스가 0개라 즉시 동작에 영향은 없지만,
  태스크 모델의 일관성(한 클래스는 최대 1개 태스크)을 완성하는 제안이라 받아들였다.
- **같은 boilerplate가 N번 반복되면 추출한다.** Docker daemon 기동 여부를 점검하던 같은 ~30줄
  블록이 격리 태스크마다 중복됐다. `verifyDockerDaemon(taskName)` 헬퍼로 단일화하고
  `integrationTest`·`concurrencyTest`·`batchTest`·`sandboxTest` 4개 태스크에 일괄 적용했다.

## 미해결·열린 질문

- **docker 태그의 정보 가치 재검토(#189 이후).** H2 → Testcontainers 마이그레이션이 끝나면 docker
  태그가 "DB를 쓴다"와 거의 동치가 돼, 코드의 `@Testcontainers`와 정보가 중복된다. 그래도 CI 잡을
  가르는 도구로서는 여전히 유효하다. 의미적 라벨(예: integration)로 교체할지는 #189이 끝난 뒤 결정한다.
- **sandbox 클래스 작성 시 컨벤션 적용.** 새 sandbox 클래스는 `@Tag("sandbox")`에 더해
  `@EnabledIfEnvironmentVariable` 류 조건 어노테이션으로 필수 환경변수가 없으면 자동 skip되게 둔다.
  외부 PG sandbox에 실제 API 호출이 나가므로, 빌드 태스크 차단이 약해도 클래스 레벨에서 한 번 더
  안전망을 둔다. 가이드는 테스트 컨벤션 문서에 적어뒀다.
- **docker+concurrency 자동 검증 트레이드오프 재검토.** 동시성 회귀를 PR에서 못 잡는 비용이 실제로
  운영에 영향을 주기 시작하면, concurrency를 CI에 별도 잡으로 분리하는 옵션을 다시 본다. 지금은
  수동 정책.
