---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [testing, ci, gradle, junit, testcontainers, batch, convention]
created: 2026-06-02
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-193-test-meta-cleanup]]"
  - { repo: commerce-backend, path: "docs/test-convention.md#tag-model" }
---

# 테스트 @Tag 2축 모델(환경 요구·격리 분류) + CI 잡 단위슬라이스/통합배치 분리

> 정본(태그 규칙·CI 자동 검증 범위)은 코드 repo의 테스트 컨벤션 문서에 있다. 여기에는 왜 2축·왜 다차원·어떤 트레이드오프를 감수했는지만 남긴다(companion).

## 문제 — 환경 요구와 격리 분류가 한 태그에 뒤섞임

테스트가 늘면서 `@Tag`가 두 개의 서로 다른 질문을 한 평면 라벨에 뭉쳐 담고 있었다: **(1) 어떤 환경이 있어야 도는가**(docker 필요 등)와 **(2) 어떤 격리 그룹에 속하는가**(동시성·배치 등). 이 둘이 섞여 CI 잡을 가르는 기준으로도, 격리 태스크를 나누는 기준으로도 애매했다.

## 검토한 대안 — 단일 차원 vs 다차원, 그리고 2축 직교성

**단일 차원(평면 라벨)이 처음엔 매력적이었다.** (a) drift 우려 — 코드에 `@Testcontainers`가 이미 있는데 `@Tag("docker")`까지 달면 같은 정보를 두 군데 적는 중복이다. (b) 한 클래스에 태그 N개를 붙이는 인지 부담.

**그러나 다차원을 택했다.** 결정적 이유 셋:
- 현실이 이미 다차원이다 — `docker + concurrency`가 함께 부여돼야 맞는 클래스(동시성인데 Testcontainers도 필요)가 실제로 존재해 단일 라벨로는 못 담는다.
- CI 잡 분리에 docker 축이 필요하다 — 단위/슬라이스 잡과 도커 통합 잡을 가르려면 "이 클래스가 도커를 요구하는가"를 태그로 알아야 한다. 단일 차원이면 그 구분이 클래스명·패키지 컨벤션으로 새 오히려 복잡해진다.
- drift 위험은 태그 자동화까지 가지 않고 컨벤션 문서 + PR 리뷰로 잡는 선에서 감수.

**진짜 직교성은 "격리 종류" 관점에서 드러난다.** 처음엔 두 축이 안 직교로 보였으나("둘 다 결국 격리 사유 라벨 아닌가"), **환경 요구 축**(docker/sandbox — "환경만 있으면 일반 흐름대로 돌아도 OK")과 **격리 분류 축**(concurrency/batch/learning — "환경이 있어도 격리가 필수")으로 나누면 진짜 직교다.

## Gradle 태스크 재정의·CI 2잡 병렬 분리

- **`ciTest` 제거** — "일반 + 도커"를 한 번에 돌리던 CI 전용 태스크. 잡을 쪼개며 존재 이유 소멸.
- **`dockerTest` → `integrationTest`** — 도커는 "환경 요구"일 뿐, 태스크의 진짜 의도는 "통합 테스트를 기본 test와 격리해 돌린다"라 의도에 맞게 개명.
- **`naverPaySandboxTest` → `sandboxTest`** — PG 추가 가능성 고려해 일반화.
- **`batchTest` 신설** — 배치 컨텍스트 기동 비용이 커 기본 test와 섞으면 안 되므로 격리. 단 CI에서는 `integrationTest`와 같은 통합/배치 잡에서 함께 호출(배치 클래스 수가 적어 잡을 또 나누면 워밍업 오버헤드가 절약보다 큼).
- **CI 2잡 병렬:** `unit-slice`(`./gradlew test`, 도커 없이 빠른 피드백) / `integration`(`./gradlew integrationTest batchTest`, Testcontainers). 서로 독립 top-level 잡이라 한쪽 실패가 다른 쪽을 취소하지 않고 양쪽 결과를 다 본다.
- **concurrency / sandbox는 CI 미포함(수동 정책):** 동시성은 flaky + 실행 시간이 길고, sandbox는 외부 API 실호출이 나가 정말 필요할 때만 수동.

## learning @Tag + @Disabled 이중 격리

학습/디버그 목적 테스트(예: 주문 동시성 흐름을 `Thread.sleep`으로 관찰하던 클래스 — [[order-concurrency-service-학습코드-격리]])는 운영 검증 대상이 아니지만 회고용으로 보존한다. 옛 `@Tag("test")`를 `@Tag("learning")`으로 교체하고, 격리 대상에 `@Tag("learning")`과 `@Disabled`를 **동시에** 부여했다.

- 태그만으론 새 클래스에서 `@Tag("learning")`을 빠뜨리면 노출되고, `@Disabled`만으론 빌드 태스크 분리가 안 된다(모든 태스크가 후보로 잡은 뒤 skip). 둘 다 있으면 한쪽을 빠뜨려도 안전하고 태스크 영역에서 아예 빠진다.
- 별도 모듈로 분리하지 않고 `src/test` 위치 유지 — 운영 코드 리팩터 시 IDE가 컴파일 깨짐을 알려주는 안전망을 겸한다(폴더 분리 비용 대비 이득 부족).

## 동시성 테스트 명명 통일 보류(D만 정리)·베이스클래스 미도입

당시 동시성 테스트 이름이 4패턴(A `*ConcurrencyTest` / B `*ServiceConcurrencyTest` / C `OrderConcurrencyService*Test` / D `*ConcurrencyIntegrationTest` 1개)으로 갈려 있었다. **패턴 통일은 무리라고 봤다** — 특히 C는 카테고리 라벨이 아니라 **도메인 네이밍**(`OrderConcurrencyService`가 실제 서비스명이라 "Concurrency"가 대상 클래스 이름의 일부)이라 억지 통일하면 뜻이 어긋난다. **D 하나만** `OrderCreateConcurrencyIntegrationTest` → `OrderCreateServiceConcurrencyTest`로 개명해 패턴 B에 합류.

공통 베이스 클래스는 다시 만들지 않기로 재확인했다. `@DynamicPropertySource`로 각 테스트가 필요한 컨테이너만 명시 선언하는 패턴을 유지 — 그래야 의존 범위가 한눈에 드러난다. 공통 베이스로 묶으면 모든 통합 테스트가 모든 컨테이너에 엮이거나 조합별 베이스 트리가 생겨 의존 범위가 흐려진다.

## docker+concurrency CI 공백(의도)·배치 회귀 자동 발견

- **배치 회귀 자동 발견:** `batchTest`가 CI에 처음 포함되며, develop CI가 batch 태그를 exclude해 그동안 안 돌던 `OrderExpirationBatchTest`의 누적 회귀(재고 복원 아웃박스 이벤트 생성 호출 단언이 `times(1)`인데 만료 주문 2개 취소로 실제 2회 호출)를 즉시 잡았다. `times(1)` → `times(2)` 한 줄 fix. 이 PR의 의도(배치를 CI에 넣는다)가 첫 발동에서 곧바로 효과를 본 사례다. 만료 배치 자체는 [[주문만료-spring-batch-chunk-retry-skip]] 참조.
- **docker+concurrency 6클래스 CI 공백(의도된 상태):** 태스크를 disjoint하게(한 클래스 = 정확히 0 또는 1개 태스크) 구성하니, 격리 축 태그가 붙은 클래스는 환경 축이 docker라도 격리 태스크로만 매칭돼 CI에서 빠진다(`integrationTest`는 concurrency exclude, `concurrencyTest`는 CI 미포함). `NaverPayServiceConcurrencyTest` 등 6개가 여기 해당하며, 수동 정책으로 의도된 상태다. 다만 "어떤 변경 시 수동 검증이 필요한가"를 테스트 컨벤션 문서에 명시해 인지 비용을 문서로 해소했다.

## 미해결

- **docker 태그의 정보 가치 재검토(#189 이후).** H2 → Testcontainers 마이그레이션이 끝나면 docker 태그가 "DB를 쓴다"와 거의 동치가 돼 코드의 `@Testcontainers`와 정보가 중복된다. 그래도 CI 잡을 가르는 도구로는 여전히 유효 — 의미적 라벨(integration 등) 교체 여부는 #189 완료 후 결정.
- **docker+concurrency 자동 검증 트레이드오프.** 동시성 회귀를 PR에서 못 잡는 비용이 운영에 실제 영향을 주면 concurrency를 CI 별도 잡으로 분리하는 옵션을 다시 본다.
- Docker daemon 기동 점검 boilerplate(~30줄)를 `verifyDockerDaemon(taskName)` 헬퍼로 단일화해 4개 태스크에 적용 — 반복 boilerplate 추출의 사례. 이 세션은 [[silent-schema-drift-패턴]] 검증(integrationTest `--rerun-tasks`로 validate silent zone 확인)과 같은 CI/테스트 인프라 결이며, 동시성 단언 규칙은 [[동시성-테스트-작성-규칙과-단언-전략]], 같은 세션의 태그 재설계 발단이 된 사고는 [[multi-column-unique-length-명시-컨벤션]]에 있다.

## 근거

- [[raw/sessions/backend/2026-06-02-pr-193-test-meta-cleanup]]
