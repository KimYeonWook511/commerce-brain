---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 193 }
---

# pr-193-test-meta-cleanup

## 한 일

- 테스트 운영 메타 정비 (#177 태그 차원, #190 학습용 격리, #191 @MockitoBean)
- @Tag 두 축 다차원 모델 도입 (환경 요구 docker/sandbox + 격리 분류 concurrency/batch/learning)
- build.gradle 재정의: ciTest 제거, dockerTest→integrationTest, batchTest 신설, naverPaySandboxTest→sandboxTest
- CI 잡 분리 (unit-slice + integration 병렬, fail-fast: false)
- learning tag 신설 + 학습/디버그 테스트 격리 (@Disabled + 이중 안전망)
- review resolve + 후속 (gemini의 disjoint 제안 accept, docker daemon guard helper 추출)

## 결정한 것

### 단일 차원 vs 다차원 — 다차원 선택
- 처음엔 단일 차원이 매력적이었음. 이유: drift 우려 (`@Testcontainers` 와 `@Tag("docker")` 둘 다 적어야 하는 정보 중복), 한 클래스에 태그 N개 부여의 인지 부담
- 그러나 현실이 이미 다차원: `docker + concurrency` 가 함께 부여된 클래스가 실제로 존재
- 결정적 이유: CI 잡 분리에 docker tag 가 필요. 단일 차원으로 가면 단위/슬라이스 vs 도커 통합을 가르는 메커니즘이 클래스명/패키지 컨벤션으로 새서 더 복잡
- drift 위험은 컨벤션 문서로 PR 리뷰에서 잡는 수준이 적당
- 정본: `commerce-backend/docs/testing-conventions.md` "태그 규칙" 절

### task 이름 정리
- `dockerTest` → `integrationTest`: 도커는 "환경 요구" 이고 task 의도는 "통합 테스트 격리" → 의도에 맞는 이름
- `naverPaySandboxTest` → `sandboxTest`: PG 추가 가능성 고려한 일반화

### concurrency / sandbox 는 CI 미포함
- 동시성: flaky 가능성 + 시간 오래. 동시성 회귀를 PR 단계에서 못 잡는 비용 감수
- sandbox: 외부 환경에 실 호출 — 정말 필요할 때만

### batchTest 신설 + CI 에는 포함
- 클래스 1개라도 격리 사유(컨텍스트 비용) 명확
- CI 의 integration 잡에서 integrationTest 와 같이 호출 (별 잡 분리는 잡 워밍업 오버헤드 > 절약 시간)

### docker tag 의 미래 (#189 이후 재검토)
- H2 사라지면 docker tag 의 정보 가치 일부 약화 (코드의 @Testcontainers 와 중복)
- 그러나 CI 잡 분리 도구로서는 유효
- 의미적 라벨 (integration) 로 교체할지는 #189 끝난 후 결정

### 동시성 테스트 명명 통일 무리
- 현재 4가지 패턴: A `*ConcurrencyTest` (Cart/Stock), B `*ServiceConcurrencyTest` (NaverPay/Payment 4개), C `*ConcurrencyServiceXxxTest` (Order 도메인), D `*ConcurrencyIntegrationTest` (1개)
- C 는 사실 도메인 네이밍 — `OrderConcurrencyService` 가 실제 서비스 이름 (카테고리 라벨 아님)
- 패턴 통일은 무리. D 1개 (`OrderCreateConcurrencyIntegrationTest` → `OrderCreateServiceConcurrencyTest`) 만 정리

### learning 테스트 위치 유지 — 폴더 분리 안 함
- `@Tag("learning") + @Disabled` 이중 안전망으로 격리 효과는 동일
- 패키지 이동 비용 대비 이득 부족

### "이중 안전망" 의 이유 (@Tag + @Disabled 동시 부여)
- 태그만으로는 새 클래스 작성 시 `@Tag("learning")` 누락하면 노출됨
- `@Disabled` 만으로는 build task 분리 못 함 (모든 task 에서 후보로 잡힌 뒤 skip)
- 둘 다 있으면 한쪽 누락에도 안전 + task 영역에서 아예 빠짐

### 베이스 클래스 미도입 재확인
- 슬라이스/통합 양쪽에서 베이스 클래스 안 만든다
- `@DynamicPropertySource` 각 테스트 명시 선언 패턴 유지 — 의존 범위가 한눈에

## 막힌 점

### rebase 후 batchTest 회귀 발견 → 같은 PR 에서 fix
- develop CI 가 batch tag 를 excluded 시켜와서 `OrderExpirationBatchTest` 단언 오타가 누적
- 본 PR 의 CI 잡 분리가 batchTest 를 CI 에 포함시키며 비로소 노출
- 단언 `times(1)` → `times(2)` 한 줄 fix (만료 주문 2개 setup → 첫 배치 2회 호출)
- **의미**: "CI 에 자동 검증 포함시키는 변경" 의 가치가 첫 발동 = 회귀 자동 발견. PR 의 의도 (batch 를 CI 포함) 가 직접 효과를 본 첫 사례.

### docker+concurrency 클래스의 자동 검증 공백
- `integrationTest` 는 excludeTags concurrency 라 제외
- `concurrencyTest` 는 CI 미포함
- → 6개 클래스가 CI 자동 검증 안 됨: `OrderConcurrencyServiceDeadlockMysqlTest`, `OrderCreateServiceConcurrencyTest`, `PaymentApprovalServiceConcurrencyTest`, `PaymentApprovalAttemptServiceConcurrencyTest`, `PaymentCancellationAttemptServiceConcurrencyTest`, `NaverPayServiceConcurrencyTest`
- 의도된 상태 (수동 정책) — testing-conventions.md 에 명시로 인지 비용 해소

## 배운 것

### Tag 다차원 모델의 진짜 직교성
- 처음엔 두 축이 안 직교로 보임 ("둘 다 격리 사유 라벨")
- 격리 사유 관점으로 다시 보면: docker/sandbox 는 "환경만 있으면 일반 흐름 OK", concurrency/batch/learning 은 "환경 있어도 격리 필수"
- 격리 *종류* 차원에서 진짜 직교

### CI 자동 검증 공백의 인지 비용
- 단순히 CI 에서 안 도는 게 아니라, "어떤 변경 시 수동 검증이 필요한지" 가 문서에 명시되어야 다음 세션이 인지함
- testing-conventions.md 의 "CI 자동 검증 범위" 절이 핵심

### gemini disjoint 제안의 의의
- `integrationTest` excludeTags 에 `sandbox` 추가 — 매칭 클래스 0개라 즉시 동작 영향은 없지만 모델 일관성 확보
- 자동 리뷰가 "모델 빈틈" 까지 짚는 사례

### helper 추출 pattern
- 같은 boilerplate 가 N 번 반복되면 추출
- docker daemon guard 가 4 task 에 같은 30줄 → `verifyDockerDaemon(taskName)` helper 로 단일화

## 다음 단계

### docker tag 의 정보 가치 재검토 (#189 후)
- H2 → Testcontainer 마이그레이션이 끝나면 docker tag 가 "DB 사용" 과 거의 동치
- 의미적 라벨 (integration) 로 교체할지 그때 결정

### sandbox 클래스 작성 시 컨벤션 적용
- `@Tag("sandbox")` + `@EnabledIfEnvironmentVariable` 류 조건 어노테이션으로 환경변수 부재 시 자동 skip
- testing-conventions.md 에 가이드 적었음

### docker+concurrency 자동 검증 트레이드오프 재검토
- 동시성 회귀를 PR 에서 못 잡는 비용이 실제로 운영에 영향을 주면 → concurrency 잡을 CI 에 별도 잡으로 분리하는 옵션 재검토
- 현재는 수동 정책
