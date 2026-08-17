---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [database, enum, hibernate, check-constraint, schema, jpa]
created: 2026-06-02
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]"
---

# enum 컬럼 DB CHECK 미사용 — 유효성 보장을 application layer로 위임

## 정본 위치

이 결정의 정본은 코드 repo(commerce-backend)에 남긴 pr-184 신규 ADR이다. 같은 PR의 [[flyway-도입-ddl-auto-validate-전환]]과 짝을 이룬다. 여기 wiki에는 문제 이해·대안·다시 본다면만 남기고 스키마 스크립트 본문은 복사하지 않는다.

## 내가 이해한 문제 — 자동 CHECK + validate 미검 silent mismatch

Hibernate가 `@Enumerated(STRING)` 컬럼에 자동 생성한 `CHECK (column in (...))` 제약이 V1 베이스라인에 굳어 있었다. 문제는 이 제약이 [[silent-schema-drift-패턴]]과 같은 결의 함정을 잠재한다는 점이다.

- `ddl-auto: validate`는 CHECK **내용**을 검증하지 않는다.
- 따라서 Java enum에 새 값을 추가하면 validate는 통과하지만, 그 값으로 INSERT하는 순간 CHECK 위반으로 런타임 실패한다.
- 즉 스키마(오래된 CHECK)와 코드(새 enum 값)가 조용히 어긋난 채 배포되고, 특정 값이 실제로 쓰이는 시점에야 터진다.

## 검토한 대안

- **(A) CHECK 유지** — DB가 invalid enum을 막는 이중 안전망. 그러나 enum 진화마다 마이그레이션 마찰이 붙고, validate 미검 silent mismatch 위험이 그대로 남는다. 기각.
- **(B) V1에서만 제거하고 향후 자동 생성은 방치** — 다음 `ddl-auto: create` 기반 dump 시 CHECK가 다시 자동 생성돼 회귀한다. 기각.
- **(C, 택함) 의식적 제거 + application layer 위임** — V1의 CHECK를 모두 제거하고 enum 유효성을 Java enum 타입 시스템 + `@Enumerated(STRING)`에 위임. Hibernate 차원의 자동 생성 차단은 후속 task로.

## 결정 + 이유

V1의 CHECK 제약을 모두 제거하고 enum 유효성 보장을 application layer에 위임한다(C).

- **이중 안전망의 실용 가치가 작다:** 단일 백엔드 + JPA 단일 INSERT 경로 + 외부 시스템 직접 INSERT 없음. `@Enumerated(STRING)`이 invalid enum을 application layer에서 이미 차단해 DB CHECK는 실질적으로 발동될 일이 없다.
- **enum 진화 마찰:** 결제 fail_code·주문 status·이벤트 type 등 enum은 도메인이 자라면 종종 추가되는데, CHECK를 유지하면 추가마다 마이그레이션 스크립트가 필요하다.
- **자동 생성이라 의식되지 않는다:** 두는 것도 안 두는 것도 의식적이어야 하는데 Hibernate가 자동으로 만들어버려 의식되지 않는다. 이 결정은 그 자동 동작을 명시적으로 우회하는 의미다 — [[silent-schema-drift-패턴]]의 "의도적 제거" 처방의 한 사례다.

## 지금 다시 본다면

Hibernate 차원에서 CHECK 자동 생성을 차단하는 게 근본책이다. `ddl-auto: create` 기반으로 dump를 다시 뜨면 Hibernate가 enum CHECK를 또 자동 생성하므로, 그때마다 의식적 제거가 계속 필요하다. 설정 또는 엔티티 어노테이션 차원에서 자동 생성을 끄는 방법은 후속 task로 남겼다.

## 재검토 트리거

외부 시스템이 같은 DB에 INSERT하는 시나리오(마이크로서비스 분리, BI/ETL 도구 직접 접근, 운영자 raw SQL 수정)가 일상화되면, application layer만으로는 안전망이 부족해져 DB CHECK 재도입을 검토해야 한다. 현재 전제(단일 INSERT 경로)가 깨지는 시점이 곧 재검토 시점이다.

> [!note] 인접 결정
> 반대편 축은 [[multi-column-unique-length-명시-컨벤션]]이다 — 그쪽은 스키마에 **명시할 것**을 정했고 이쪽은 **안 맡길 것**을 정했다.

## 근거

- [[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]
