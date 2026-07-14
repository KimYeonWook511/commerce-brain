---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [database, flyway, migration, schema, ddl, hibernate, mysql]
created: 2026-06-02
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]"
---

# Flyway 도입 + ddl-auto validate 전환 — 단일 DB에서도 silent drift는 코드 내부 요인으로 일어난다

## 정본 위치

이 결정의 정본은 코드 repo(commerce-backend)에 남긴 **두 신규 ADR**(Flyway 도입, [[enum-db-check-미사용-application-layer-위임]])과 pr-184 태스크 회고 문서다. 여기 wiki에는 "내가 이해한 문제·왜 입장을 뒤집었나·다시 본다면"만 남기고, 스크립트 본문·설정 전문은 복사하지 않는다.

## 내가 이해한 문제 — 두 사고가 같은 패턴

DB는 단일 MySQL 하나뿐이고 다중 DB 계획도 없어, 그동안 JPA `ddl-auto: update`로 충분하다는 입장으로 Flyway 도입을 미뤄왔다. 이 입장을 무너뜨린 건 최근 두 사고가 *같은 패턴*이라는 인식이었다.

- **사고 1 (#142, ENUM silent drift):** Hibernate 6.x가 `@Enumerated(STRING)`을 MySQL native ENUM으로 매핑하도록 dialect가 바뀌면서, `ddl-auto: update`로 NOT NULL ENUM 컬럼을 추가할 때 기존 row에 의도치 않은 첫 번째 값이 조용히 채워졌다.
- **사고 2 (#176, multi-column unique 미적용):** `tbl_payment_attempt`의 4-column unique 대상 컬럼들이 길이 미지정으로 utf8mb4에서 InnoDB 한도(3072 bytes)를 넘어 MySQL이 unique 생성을 거부했는데, Hibernate 기본 핸들러가 WARN으로만 로그하고 부팅을 계속해 unique가 빠진 채 운영됐다. → [[multi-column-unique-length-명시-컨벤션]]

두 사고 모두 "코드는 정상으로 보이지만 실제 DB 스키마가 조용히 어긋난 상태"다. 이 공통 결을 일반화한 것이 [[silent-schema-drift-패턴]]이며, 이 결정은 그 패턴을 도구 차원에서 해소하는 핵심 축이다.

## 검토한 대안

- **`ddl-auto: update` 유지** — 두 사고의 silent drift를 그대로 안고 간다. 코드 변화량에 비례해 재발 위험. 기각.
- **`validate`만 켜고 마이그레이션은 손으로 SQL 관리** — validate는 컬럼/타입 누락을 부팅 실패로 잡아 사고 1 유형은 방어하나, 적용 순서·이력이 코드 밖으로 새고 환경 간 일관성이 사람 기억에 의존한다. 사고 2 유형(unique 누락)은 validate가 unique 제약을 검사하지 않아 여전히 못 잡는다. 기각.
- **Liquibase** — XML/YAML DSL 추상화. MySQL/JPA 단일 스택에서 추상화 가치가 제한적이고, SQL을 그대로 다루는 게 디버깅·리뷰에 유리해 기각.
- **택함: Flyway** — SQL 그대로 버전 관리 + Spring Boot auto-config 내장. (a) validate로 컬럼/타입 누락 가시화, (b) 명시적 스크립트로 변경 의도 코드화, (c) `flyway_schema_history`로 환경 간 일관성 추적 — 세 축으로 drift 패턴을 해소한다.

## 결정 + 이유 — 입장 뒤집힘·시점 우위

- **핵심 인식:** 단일 DB라도 drift는 일어난다. 원인이 다중 DB나 팀 간 분기 같은 *외부* 요인이 아니라 dialect 변경 / DDL silent fail 같은 *코드 내부* 요인이라는 점이 결정적이었다. "단일 DB라서 Flyway가 필요 없다"는 입장은 두 사고를 나란히 본 시점에서 무너졌다.
- **시점의 우위:** 운영 DB가 아직 미가동이라 현 엔티티 기준 단일 베이스라인(V1) 스크립트로 출발할 수 있다. 운영이 돌기 시작하면 baseline-on-migrate, 운영 dump → baseline 작성 → checksum 검증 같은 도입 절차가 모두 필요해진다. 지금은 그게 다 불필요하다 — 도입이 늦어질수록 절차 비용이 기하급수로 는다.

## 프로파일별 적용 방식·Batch 메타테이블 제외·clean-disabled

- **운영/로컬/integrationTest:** `validate` + Flyway 활성.
- **test(H2):** create-drop + Flyway 비활성 유지. 단위/슬라이스 테스트의 부팅 속도가 자산이고, V 스크립트가 MySQL 문법이라 H2에 직접 적용 불가하기 때문.
- **integrationTest(Testcontainers MySQL):** Flyway 활성 — 컨테이너 싱글톤 재사용 + 컨텍스트 캐싱 + `deleteAllInBatch()` 격리 모델과 자연스럽게 맞물린다.
- **Spring Batch 메타테이블:** Flyway 관리에서 제외하고 Batch 자체 `initialize-schema` 유지. 버전업 시 마이그레이션 책임이 프로젝트로 넘어오는 비용을 회피한다.
- **운영 안전망:** `flyway.clean-disabled: true` 명시.

## halt_on_error 제거 — pr-179 도입 lifecycle 종료

직전 결정([[multi-column-unique-length-명시-컨벤션]]의 부속 결정)에서 스키마 회귀를 부팅 시점에 노출시키려고 `application-local.yml`에만 `hibernate.hbm2ddl.halt_on_error: true`를 켰었다. 이 PR에서 재검토해 제거했다.

> [!note] lifecycle 관계
> `validate`로 전환하면 Hibernate가 DDL을 실행하지 않으므로 `halt_on_error`의 발동 조건(Hibernate DDL 실행 중 실패) 자체가 사라진다. 스키마 변경 실패 차단 책임은 이제 Flyway가 가져간다(마이그레이션 SQL 실패 시 Flyway가 부팅을 막는다). pr-179에서 이 설정은 도입 당시 이미 "local `ddl-auto: update` 전제에 묶이며, ddl-auto를 바꿀 때 함께 재검토해야 한다"는 fragile dependency로 명시돼 있었다 — 그 예고대로 소멸했다.

놓친 지점: 직전 결정이 그 fragile dependency를 *명시했음에도*, 이 PR의 plan 단계에서 그 메모를 의존 그래프 trigger로 활용하지 못했고 회고/외부 조언으로 review 단계에서야 잡았다.

## 인정한 비용·validate 사각지대

- **운영 복잡성 증가(도입을 미뤄온 이유):** 엔티티를 바꾸면 같은 PR에서 마이그레이션 스크립트를 함께 써야 하고, 로컬에서 엔티티만 고치고 부팅하면 실패한다. 두 사고의 silent drift 비용이 이 복잡성보다 크다고 봐 수용했다.
- **validate가 못 잡는 것:** validate는 컬럼/타입 누락은 잡지만 **unique·인덱스·CHECK는 검사하지 않는다.**
  - 사고 2 유형(unique 누락)은 Flyway 도입 후에도 unique 대상 컬럼 length 명시([[multi-column-unique-length-명시-컨벤션]])로 1차 방어한다. Flyway는 "변경 이력이 명시적이라 리뷰에서 잡힐 가능성을 높인다"는 간접 효과로만 기여한다.
  - CHECK 미검은 [[enum-db-check-미사용-application-layer-위임]]에서 별도로 다뤘다.
  - enum 컬럼의 `ENUM→VARCHAR` 매핑 규칙이 Flyway 도입 후에도 살아남는 이유(H2↔MySQL parity + validate가 sql type 차이를 strict 비교하지 않는 silent zone)도 이 사각지대의 한 갈래다 — [[silent-schema-drift-패턴]] 참고.

## 근거

- [[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]
