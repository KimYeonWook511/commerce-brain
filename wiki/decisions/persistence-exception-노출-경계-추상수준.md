---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, exception-strategy, layered-architecture, domain-purity, adapter, archunit, spring-dao, optimistic-lock, spring-batch]
created: 2026-07-11
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-07-11-pr-281-exception-exposure-boundary-abstraction]]"
---

# 영속성 예외 노출 경계를 예외의 추상 수준(추상 vs 구현 구체)으로 다시 긋기

## 컨텍스트·문제 — 두 층으로 나뉜 Spring 영속성 예외와 과잉 금지선

백엔드는 4계층(presentation → application → domain ← infrastructure)이고, JPA/Hibernate가 던지는 영속성 예외를 어느 계층까지 노출할지의 경계가 쟁점이다. 기존 규칙은 "Spring DAO 예외 **전체**를 안쪽 계층(application·domain·presentation)에서 참조·catch 금지"였다.

그런데 Spring의 영속성 예외 계층은 성격이 다른 두 층으로 나뉜다.

- **구현 비의존 추상 예외** — `org.springframework.dao.*`(`OptimisticLockingFailureException`, `DataIntegrityViolationException`, `DuplicateKeyException` 등). JDBC/JPA/Hibernate/MyBatis 어느 구현도 가리지 않는 Spring의 추상 계층.
- **구현체에 묶인 구체 예외** — `org.springframework.orm.*`, `org.hibernate.*`, `jakarta.persistence` 예외. JPA/Hibernate라는 특정 구현에 직접 묶인다.

기존 규칙은 이 둘을 한 금지선에 묶었다. 그 결과 예외를 업무적으로 따로 처리하지 않는 순수 재시도 경로에서까지, infrastructure 어댑터가 flush를 앞당겨 도메인 예외로 번역하는 코드를 강제당했다 — 불필요한 조기 flush + 변환 보일러플레이트 증가.

## 결정 1 — 경계를 추상(dao.*) vs 구현체 구체로 다시 긋기

- application·presentation은 **구현체에 묶인 구체 예외만** 참조 금지. 구현 비의존 Spring DAO 추상 예외(`org.springframework.dao.*`)는 다뤄도 된다.
- domain은 추상·구체 **둘 다** 금지(가장 안쪽 순수 도메인 로직).

근거: Spring 자체는 교체 대상이 아니고 dao 추상 계층은 어느 영속성 구현도 가리지 않는다. 또한 추상 상위 타입을 catch하면 그 하위 구현체 타입이 다형적으로 걸리므로, 상위 계층이 구현체 타입 이름을 직접 부를 일이 없다. 즉 **격리해야 하는 것은 구현체에 묶인 구체 예외뿐이고, 구현 비의존 추상은 상위가 공유해도 도메인 순수성이 깨지지 않는다.** "무엇을 격리하느냐"의 기준을 계층이 아니라 예외의 추상 수준으로 옮긴 것이 핵심이다.

이 "추상 상위 타입으로 다형적 catch → 구현체 이름을 안 부른다"는 기법은 [[security-common-leaf-재편과-토큰-포트-네이밍]]에서 필터가 인증 도메인의 구체 예외를 공통 예외 베이스로 잡아 leaf를 유지한 것과 같은 원리다.

## 결정 2 — 번역은 선별적: 안쪽이 그 예외로 분기하는가

기술 예외를 도메인 예외로 번역하는 것은 **의무가 아니라 판단**이다. 어댑터는 **안쪽 계층이 그 예외에 따라 처리를 달리해야 할 때만** 번역한다(유니크 위반 → "이미 존재"/이중결제 차단, 버전 충돌 → skip 판단 등). 순수 재시도처럼 예외를 구분해 처리할 필요가 없으면 번역하지 않고, 상위 계층이 DAO 추상 예외(`OptimisticLockingFailureException`)를 직접 잡아 새 트랜잭션으로 재시도하거나 끝단 핸들러로 흘려보낸다.

판단축은 하나 — "안쪽이 그 예외를 실제로 다루는가(분기·차단·skip)". 이 선별적 번역의 실제 사례가 결제 어댑터의 유니크 위반 → 도메인 예외 번역([[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]], [[결제승인완료-보상-완료우선-이중결제-adapter매핑]])이며, 그 경로들은 모두 안쪽이 결과로 분기(차단·skip)하므로 조기 `saveAndFlush` flush 의존과 함께 유지가 맞다고 결론냈다.

## 검토한 대안과 트레이드오프

- **(기각) 기존 유지 — dao 추상까지 안쪽에서 계속 금지.** 도메인 순수성은 가장 강하지만, 상위가 Spring 추상 하나에도 의존 못 해 순수 재시도 경로에서까지 flush+번역을 강제 → 불필요 코드.
- **(채택) 구현체 구체 예외만 격리.** application이 Spring 추상 타입 하나(`org.springframework.dao`)에 의존하는 대가를 진다. 다만 이는 특정 구현이 아니라 추상이라 영속성 구현 교체(JPA ↔ MyBatis 등)에 흔들리지 않는다. 대가 대비 이득(불필요 번역 제거)이 크다.

## ArchUnit 기계 강제를 추상 수준 기준으로 교체 (Spring Batch fault-tolerance carve-out)

- domain은 dao 추상 예외까지 전부 참조 금지.
- application·domain·presentation은 구현체 예외(`org.springframework.orm`·`org.hibernate`·`jakarta.persistence` 예외 계층)만 금지 — dao 추상은 허용.
- presentation은 낙관 락 충돌 예외 계층을 catch하지 않는다(전파 → application 재시도 또는 끝단 매핑).
- **Spring Batch fault-tolerance carve-out**: 배치 step에서 특정 예외를 재시도/건너뛰기 대상으로 `.retry`/`.skip` 선언하는 것은 충돌 타입을 프레임워크에 **선언적으로 신고**하는 경계라 실제 catch가 아니므로, 그 배치 설정 한 곳만 좁은 예외처로 인정([[주문만료-spring-batch-chunk-retry-skip]]의 배치 설정이 이에 해당).

이번 변경은 상위 금지를 푸는 **완화** 방향이라 신규 위반이 생기지 않았고, 전역 예외 핸들러는 계층 규칙 대상 밖(공통 패키지)이라 예외처가 필요 없어져 규칙이 오히려 단순해졌다.

## 리스크·한계

- **ArchUnit은 맨 catch절의 예외 타입을 의존성으로 잡지 못한다.** 메서드 시그니처·필드·`.class` 리터럴 참조는 잡지만, 순수 `catch (SomeException e)`는 탐지 못 한다. 따라서 "application이 구체 예외를 catch만 하는" 경우는 규칙이 못 잡을 수 있으니, 규칙이 실제로 무엇을 탐지하는지 알고 설계해야 한다. (같은 catch절 바인딩의 위험이 [[jwt-예외-catch-footgun-bare-securityexception]]에서 다른 형태로 터졌다.)

## 미해결·후속 — 도메인 예외 HTTP 의존 제거

이 시점에 도메인 예외는 여전히 HTTP 상태(`HttpStatus`)를 직접 들고 있었다(도메인이 전송 계층에 의존). 이 경계 재정의와는 별개 문제로 판단해 후속으로 미뤘고, 이어지는 [[도메인-예외-httpstatus-제거-errorcategory]]에서 완결했다. 그 노트가 다시 [[인프라-일시장애-503-예외-매핑-판단축]]으로 이어져 "직접 정의한 port만 변환, 표준 인프라 예외는 전파"라는 이 노트의 선별 번역 원리를 인프라 장애까지 확장한다.

되감을(불필요하게 번역하던) 어댑터는 없었다 — 기존 번역(예약 사용 → "이미 사용됨", 승인 저장 → "이미 완료된 주문", 상태 저장 → "동시 수정됨")은 전부 안쪽이 그 결과로 분기하는 경로라 유지가 맞았다.

## 근거

- [[raw/sessions/backend/2026-07-11-pr-281-exception-exposure-boundary-abstraction]]
