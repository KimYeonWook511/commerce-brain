---
platform: backend
author: KimYeonWook511
created: 2026-07-11
origin:
  - { type: pr, repo: commerce-backend, ref: 281 }
---

# 영속성 예외 노출 경계를 예외의 추상 수준으로 다시 긋기

백엔드는 4계층(presentation → application → domain ← infrastructure)이고, 영속성 예외(JPA/Hibernate가 던지는 예외)를 어느 계층까지 노출할지의 경계를 다시 정한 작업이다.

## 결정한 것

**문제.** 기존 경계는 "Spring DAO 예외 전체를 안쪽 계층(application·domain·presentation)에서 참조·catch 금지"였다. 그런데 Spring의 영속성 예외 계층은 두 층으로 나뉜다:
- **특정 구현에 묶이지 않은 추상 예외** — `org.springframework.dao.*`(예: `OptimisticLockingFailureException`, `DataIntegrityViolationException`, `DuplicateKeyException`). JDBC/JPA/Hibernate/MyBatis 어느 구현도 가리지 않는 Spring의 추상 계층.
- **특정 구현체에 묶인 구체 예외** — `org.springframework.orm.*`, `org.hibernate.*` 예외, `jakarta.persistence` 예외. JPA/Hibernate라는 구현에 직접 묶인다.

기존 규칙은 이 둘을 같은 금지선에 묶었다. 그래서 순수 재시도처럼 예외를 업무적으로 따로 처리하지 않는 경우에도, infrastructure 어댑터가 flush를 앞당겨 도메인 예외로 번역하는 코드를 강제당했다(불필요한 flush + 변환 보일러플레이트 증가).

**결정 1 — 경계를 예외의 추상 수준으로 다시 긋는다.**
- application·presentation은 **구현체에 묶인 구체 예외만** 참조 금지. 특정 구현에 안 묶인 Spring DAO 추상 예외(`org.springframework.dao.*`)는 다뤄도 된다.
- domain은 추상·구체 **둘 다** 금지(가장 안쪽, 순수 도메인 로직).
- 근거: Spring 자체는 교체 대상이 아니고 dao 추상 계층은 어느 영속성 구현도 가리지 않는다. 그리고 추상 상위 타입을 catch하면 그 하위 구현체 타입이 다형적으로 걸리므로, 상위 계층이 구현체 타입 이름을 직접 부를 일이 없다. 즉 "격리해야 하는 건 구현체에 묶인 구체 예외뿐이고, 구현 비의존 추상은 상위가 공유해도 순수성이 안 깨진다".

**결정 2 — 번역은 의무가 아니라 선별적이다.** 기술 예외를 도메인 예외로 번역하는 것은 **안쪽 계층이 그 예외에 따라 처리를 달리해야 할 때만** infrastructure 어댑터가 한다(유니크 위반 → "이미 존재"/이중결제 차단, 버전 충돌 → skip 판단 등). 순수 재시도처럼 예외를 구분해 처리할 필요가 없으면 번역하지 않고, 상위 계층이 DAO 추상 예외(`OptimisticLockingFailureException`)를 직접 잡아 새 트랜잭션으로 재시도하거나 최종 예외 핸들러로 흘려보낸다. 판단축은 "안쪽이 그 예외를 실제로 다루는가"다.

**검토한 대안과 트레이드오프.**
- (기존 유지) dao 추상까지 안쪽에서 계속 금지 — 도메인 순수성은 가장 강하지만, 상위가 Spring 추상 하나에도 의존 못 해 순수 재시도 경로에서까지 flush+번역을 강제 → 불필요 코드. 기각.
- (채택) 구현체 구체 예외만 격리 — application이 Spring 추상 타입 하나(`org.springframework.dao`)에 의존하는 대가를 진다. 다만 이는 특정 구현이 아니라 추상이라 영속성 구현 교체(JPA ↔ MyBatis 등)에 흔들리지 않는다. 대가 대비 이득(불필요 번역 제거)이 크다.

**기계 강제(ArchUnit)를 추상 수준 기준으로 교체.**
- domain은 dao 추상 예외까지 전부 참조 금지.
- application·domain·presentation은 구현체 예외(`org.springframework.orm`·`org.hibernate` 예외 계층·`jakarta.persistence` 예외 계층)만 금지 — dao 추상은 허용.
- presentation은 낙관 락 충돌 예외 계층을 catch하지 않는다(전파 → application 재시도 또는 끝단 매핑). 단 Spring Batch의 fault-tolerance(배치 step에서 특정 예외를 재시도/건너뛰기 대상으로 `.retry`/`.skip` 선언)는 충돌 타입을 프레임워크에 **선언적으로 신고**하는 경계라 실제 catch가 아니므로, 그 배치 설정 한 곳만 좁은 예외처로 인정.
- 완화(상위 금지를 푸는 방향)라 신규 위반이 생기지 않았고, 전역 예외 핸들러는 계층 규칙 대상 밖(공통 패키지)이라 예외처가 필요 없어져 규칙이 오히려 단순해졌다.

## 배운 것

- **경계를 "추상 vs 구현 구체"로 그으면 상위 계층이 벤더 추상까지 다뤄도 도메인 순수성이 안 깨진다.** 추상 상위를 catch하면 구현체 하위가 다형적으로 걸려, 상위가 구현체 타입 이름을 쓸 일이 없기 때문. "무엇을 격리하느냐"를 계층이 아니라 예외의 추상 수준으로 판단하는 게 핵심.
- **번역은 규칙이 아니라 판단**: "안쪽이 그 예외로 분기(차단·skip)하는가"만 보고, 아니면 추상 예외를 그대로 흘린다. 이걸로 순수 재시도 경로의 flush+번역 보일러플레이트를 없앨 수 있다.
- ArchUnit(경계 검증 도구)은 **맨 catch절의 예외 타입은 의존성으로 잡지 못한다**(메서드 시그니처·필드·`.class` 리터럴 참조는 잡음). 그래서 "application이 구체 예외를 catch만 하는" 경우는 규칙이 못 잡을 수 있다 — 규칙이 실제로 무엇을 탐지하는지 알고 설계해야 한다.

## 미해결·열린 질문

- 이 시점에 도메인 예외는 여전히 HTTP 상태를 직접 들고 있었다(도메인이 전송 계층에 의존) — 이 경계 재정의와 별개 문제로 판단해 후속으로 미뤘고, 이어지는 작업 [[raw/sessions/backend/2026-07-11-pr-282-domain-exception-error-category]]에서 완결했다.
- infrastructure 어댑터의 기존 번역(예약 사용 → "이미 사용됨", 승인 저장 → "이미 완료된 주문", 상태 저장 → "동시 수정됨")은 전부 안쪽이 그 결과로 분기(차단·skip)하는 경로라 유지가 맞다고 결론 — 되감을(불필요하게 번역하던) 어댑터는 없었다.
