---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [database, schema, hibernate, stale-doc, jpa, mysql]
created: 2026-06-01
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]"
  - "[[raw/sessions/backend/2026-06-01-pr-179-unique-key-length]]"
---

# silent schema drift 패턴 — Hibernate 자동 추론 의존 + 엔티티 의도 미명시

## 한 줄 정의

코드(엔티티)는 정상으로 보이나 실제 DB 스키마가 조용히 어긋나는 사고들의 공통 패턴. Hibernate의 자동 추론에 암묵 의존하고 엔티티가 도메인 의도를 명시하지 않을 때, DDL 실패나 dialect 매핑이 WARN·silent fill로 묻힌 채 부팅이 계속되며 발생한다.

## 공통 원인

두 사고(#142, #176)로 시작해 pr-184 review에서 5건 + audit까지 총 7건을 더 발견하며 일반화됐다. 원인은 두 축으로 수렴한다.

1. **Hibernate 자동 추론에 암묵 의존:** dialect 길이 추론(`@Lob` → `tinytext`), 자동 hash unique/FK 이름(`UK31to4n8j2...`), enum 컬럼 자동 `CHECK` 생성, native ENUM silent fill. 두는 것도 안 두는 것도 의식적이어야 하는데 자동으로 만들어져 *의식되지 않는다.*
2. **엔티티가 도메인 의도를 명시하지 않음:** NOT NULL, `@Version` DEFAULT 0, audit(`created_at`/`updated_at`) NOT NULL 같은 의도가 코드에 표현되지 않아 스키마와 어긋나도 아무도 눈치채지 못한다.

## 사례 인벤토리

| # | 사례 | silent 진입점 |
|---|---|---|
| #142 | ENUM 컬럼 첫 값 silent fill | dialect가 native ENUM 매핑 → `ddl-auto: update`가 기존 row에 첫 값 채움 |
| #176 | multi-column unique 미적용 | `@Column(length)` 미명시 → utf8mb4 바이트 합이 InnoDB 3072 한도 초과, WARN만 남기고 부팅 |
| outbox payload | `tinytext`(255B)로 매핑 | `@Lob` + length 미지정 → dialect가 "작은 텍스트"로 추론 (의도는 큰 텍스트) |
| `@Version` | `bigint` nullable | `@Version Long`만 선언 → V1의 `NOT NULL DEFAULT 0` 의도 미반영 |
| audit | `DEFAULT NULL` | `BaseTimeEntity`가 `@Column(nullable=false)` 누락 |
| 자동 hash 이름 | 컨벤션 위반 unique/FK 이름 | `@Column(unique=true)`/`@JoinColumn`만 → `UK.../FK...` 자동 생성 |
| enum CHECK | 런타임 silent mismatch | 자동 `CHECK(col in (...))` → validate 미검, enum 값 추가 시 INSERT 실패 |

- #176은 [[multi-column-unique-length-명시-컨벤션]], enum CHECK는 [[enum-db-check-미사용-application-layer-위임]]에서 각각 결정으로 다뤘다.

## 해결 패턴

세 가지 처방으로 수렴한다.

1. **표준 type code 명시:** dialect 추론 대신 `@JdbcTypeCode(SqlTypes.LONGVARCHAR)`처럼 표준 코드를 박는다. (`@Lob` 제거) 실측 반전 사례: LONGVARCHAR가 `mediumtext`일 거라 추정했으나 dump를 떠보니 `text`(64KB)였다 — *dump 떠보기 전엔 모른다.*
2. **컨벤션 이름 명시:** `uk_<target>_<columns>`, `fk_<source>_<columns>`를 엔티티에 박아 자동 hash 회귀를 막는다.
3. **의도적 제거:** enum 유효성 CHECK는 DB가 아니라 application layer로 위임한다([[enum-db-check-미사용-application-layer-위임]]).

이 처방들의 공통 목표는 "엔티티가 의도를 코드에 명시해 dump를 다시 떠도 회귀하지 않게" 만드는 것이다. mysqldump raw 결과 ≠ V1 최종 형태 — 도메인 순서·일관 collation·audit NOT NULL·version DEFAULT 0으로 정리한 결과가 V1이고, 엔티티가 의도를 명시해야 그 정합이 유지된다.

## validate silent zone · ENUM→VARCHAR 매핑 parity

- **validate silent zone:** Hibernate SchemaValidator는 enum 매핑(native ENUM) vs 스키마(varchar)의 sql type 차이를 *strict 비교하지 않는다.* 즉 `@JdbcTypeCode(SqlTypes.VARCHAR)` 매핑이 빠져도 validate가 통과시키는 사각이 있다. review에서 `AsyncTestEntity.status`에 매핑이 빠진 걸 발견하고 integrationTest를 `--rerun-tasks`로 강제 재실행해서야 확인됐다.
- **ENUM→VARCHAR 매핑이 살아남는 이유:** Flyway + validate로 `ddl-auto`가 안 돌아도 이 규칙은 무용하지 않다. (a) 기존 row 묻힘 경로는 validate 전환으로 닫혔지만, (b) *새 INSERT 시 NOT NULL 컬럼 첫 값 silent fill*은 native ENUM 자체 동작이라 test/prod 모두에서 발생 가능하다. 특히 test 프로파일은 H2 + create-drop이라, H2Dialect가 native ENUM으로 매핑하면 prod(MySQL varchar)와 INSERT 시 NOT NULL 위반 동작이 갈린다(test는 silent fill 통과, prod는 exception). 이 *환경별 행위 갈림*이 규칙을 코드로 유지하는 진짜 비용이며 H2↔MySQL parity + dialect 변경 안전망 역할이다.

이 두 사각을 도구 차원에서 좁히는 것이 [[flyway-도입-ddl-auto-validate-전환]]이다.

## 외부 LLM review가 카테고리 확장 트리거

internal plan만으로는 7건까지 못 갔을 것이다 — 이미 있던 사고 둘(#142, #176)조차 이 PR plan은 그 둘만 인용했다. Gemini의 한 건 지적(outbox payload `tinytext`)이 *카테고리 자체를 재검토*하게 만들어 6건을 더 발견하게 했고, Codex의 `AsyncTestEntity` 지적이 validate silent zone을 드러냈다. 자체 review에 머물지 말고 외부 LLM review를 의식적으로 트리거하는 가치를 확인한 사례. 이는 architecture 문서가 심볼을 나열해 stale해지는 결([[문서-코드-정합성-개념정본-심볼최소화]])과 같은 "자동·암묵 의존" 계열의 문제다.

## 열린 질문

- Hibernate 차원에서 enum CHECK 자동 생성을 끄는 방법(후속 task) — 안 끄면 `ddl-auto: create` 기반 dump 시 CHECK가 또 생성돼 의식적 제거가 계속 필요하다.
- 다른 엔티티의 multi-column unique가 InnoDB 한도를 넘는지는 자동 검증되지 않는다. 신규 추가 시 [[multi-column-unique-length-명시-컨벤션]]을 참고해 length를 명시해야 한다.
- validate silent zone(sql type 미비교)을 보완하는 검증 수단은 아직 없다.

## 근거

- [[raw/sessions/backend/2026-06-02-pr-184-flyway-silent-drift]]
- [[raw/sessions/backend/2026-06-01-pr-179-unique-key-length]]
