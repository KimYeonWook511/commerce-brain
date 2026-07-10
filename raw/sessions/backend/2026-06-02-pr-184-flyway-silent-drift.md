---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 184 }
---

## 한 일

- Flyway 도입 PR #184 끝까지 (Closes #158).
- ADR-024 (Flyway 도입), ADR-025 (enum DB CHECK 미사용) 신규.
- Review 단계에 silent drift 8가지 일괄 정리:
  1. outbox payload `tinytext` → `text` (`@Lob` 제거 + `@JdbcTypeCode(SqlTypes.LONGVARCHAR)`)
  2. `halt_on_error` 제거 (ADR-023 fragile dependency 해소)
  3. version `@Column(nullable=false)` 명시 (Order/Stock/CartItem)
  4. audit (`created_at`/`updated_at`) `NOT NULL` 명시 (BaseTimeEntity)
  5. 단일 unique 컨벤션 이름 (`uk_<target>_<columns>`)
  6. FK 컨벤션 이름 (`fk_<source_table>_<source_columns>`) — db-schema.md에 컨벤션 신규 추가
  7. enum DB CHECK 제거 (→ ADR-025)
  8. `AsyncTestEntity.status`에 `@JdbcTypeCode(SqlTypes.VARCHAR)` 누락 (Codex 지적) — ADR-018이 main 11개 entity는 정리됐지만 test 지원 entity가 빠져 있었음. dockerTest는 통과 → validate가 sql type 차이 strict 비교 안 한다는 silent zone 동시 발견. ADR-018·024 후속 메모로 기록.

## 결정한 것

세부는 `commerce-backend/docs/ADR.md` (ADR-024, ADR-025) 및 `docs/tasks/flyway-introduction/retrospective.md` 정본. 여기엔 "내 이해·다시 본다면" 중심.

### Flyway 도입 시점 — 입장이 뒤집힌 이유 (→ ADR-024)
그동안 단일 MySQL + ddl-auto: update로 충분이라는 입장. 두 사고(#142 ENUM silent drift, #176 multi-column unique silent 미적용)가 *같은 패턴*임을 보여줘 입장 뒤집힘 — "코드는 정상이지만 DB schema가 silent하게 어긋난 상태". 운영 DB 미가동 = baseline-on-migrate 절차 부담 없이 V1 단일 베이스라인 = 시간적 우위.

### outbox payload — JSON 컬럼 검토 끝에 SqlTypes.LONGVARCHAR (Gemini 지적 발단)
- `@JdbcTypeCode(SqlTypes.JSON)` + String: Hibernate가 ObjectMapper 통해 (de)serialize → double-quote 사고 위험. 객체 매핑이 정석.
- 객체 매핑하려면 outbox payload가 generic이라 sealed interface/Object/Map 강요 → outbox 본질(generic relay, payload schema 모름) 깨짐.
- `columnDefinition = "json"`은 ADR-018 원칙 위반 (엔티티에 MySQL 전용 SQL 박음).
- 결론: outbox는 *generic String 통과가 패턴 본질*. JSON 검증 가치 작음 (producer가 `writeValueAsString`로 만든 결과는 정의상 valid JSON).
- SqlTypes.LONGVARCHAR가 실제로 `text`(64KB)로 매핑됨을 dump로 확인. mediumtext 추정했는데 실제는 text.

### version — Long + nullable=false가 모순 아닌 이유
`Long`(null 가능) + `@Column(nullable=false)`는 직관적으로 모순 같지만 `@Version`이 INSERT 직전 0L 자동 set이라 NULL이 DB로 갈 일 없음. *NOT NULL은 엔티티 의도, DEFAULT는 DB 책임*이라는 layer 분리 (JPA @Column에 default 속성 없는 게 그 반영). V1의 `DEFAULT 0`은 raw INSERT/외부 도구 우회 안전망으로 유지.

### halt_on_error 제거 — ADR-023 fragile dependency가 본 PR에서 해소
ADR-023이 *명시한* fragile dependency를 본 PR이 해소해야 함에도 plan에서 trigger로 활용 못함. 회고/외부 조언으로 발견 → review 단계에서 제거. validate 전환되면 Hibernate가 DDL 안 실행 = 발동 조건 자체 사라짐. → ADR-023에 후속 메모.

### audit NOT NULL 명시
BaseTimeEntity가 `@Column(nullable=false)` 누락 → V1에서 `DEFAULT NULL`. *생성/수정 시각은 항상 채워짐*이라는 도메인 의도가 코드에 표현되지 않은 상태. 같은 silent mismatch 패턴.

### enum CHECK 미사용 (→ ADR-025)
사고 1·2와 같은 결의 silent mismatch 함정 — Java enum에 새 값 추가하면 validate는 통과시키지만 INSERT에서 CHECK 위반으로 런타임 실패. 결정 근거는 ADR-025 본문 참조.

**재검토 트리거**: 외부 시스템이 같은 DB에 INSERT하는 시나리오 추가 (마이크로서비스 분리, BI/ETL 직접 접근, 운영자 raw SQL 수정).

### 단일 unique/FK 컨벤션 이름
Hibernate 자동 hash 이름이 `docs/db-schema.md` 컨벤션 위반. `@Table(uniqueConstraints=...)`, `@JoinColumn(... foreignKey=@ForeignKey(name=...))`로 명시. FK 컨벤션은 db-schema.md에 신규 추가. ADR-020 cross-aggregate ID 참조 패턴(CartItem, StockHistory.admin_member_id)은 의도적으로 FK 없는 곳이라 그대로 유지.

### Flyway 도입 후에도 ADR-018이 살아남는 진짜 이유 — H2 parity와 validate silent zone (→ ADR-018·024 후속 메모)
처음엔 "Flyway + validate면 ddl-auto가 안 도니까 ADR-018(`@JdbcTypeCode(SqlTypes.VARCHAR)`)은 무용한 안전망 아닌가?"로 갔다. 사용자가 두 단계로 정정:
1. *기존 row 묻힘*만 본 것 — ADR-018의 두 갈래 중 (a)는 ddl-auto 경로라 닫혔지만 (b) *새 INSERT 시 NOT NULL silent fill*은 native ENUM 동작이라 ddl-auto와 무관. test/prod 모두에서 발생 가능.
2. *test 프로파일 H2 parity* — H2 URL `jdbc:h2:mem:testdb` pure mode + ddl-auto: create-drop이라 H2Dialect가 entity로 schema 생성. H2Dialect가 native ENUM 매핑하면 prod(MySQL + Flyway varchar)와 *INSERT 시 NOT NULL violation 동작*이 갈린다 — test는 silent fill로 통과, prod는 exception. *환경별 행위 갈림*이 진짜 비용.

dockerTest를 `--rerun-tasks`로 강제 재실행해 통과 확인 → Hibernate SchemaValidator가 enum vs varchar의 sql type 차이를 strict 비교 안 한다는 *validate silent zone* 동시 발견. ADR-018은 코드 규칙으로 유지하고, 정확한 역할은 *H2 ↔ MySQL profile parity + dialect 변경 안전망*으로 재정의.

### V900 collation 정리
test classpath의 `V900__test_support.sql`이 V1과 collation 불일치(`utf8mb4_unicode_ci` + column-level COLLATE). V1 기준(`utf8mb4_0900_ai_ci` 테이블 default + column COLLATE 제거)으로 맞춤. silent drift는 아니고 표면 일관성.

## 막힌 점

- **step 2 worker AC 실패**: harness AC executor가 multi-line shell 블록을 한 줄씩 실행. `for ...; do ...; done`이 syntax error로 깨짐. 단일 라인 명령으로 수정 + 부팅 검증은 "수동 검증 절차"로 이동.
- **step 2 worker가 application-test.yml까지 수정**: step 범위 위반. revert 후 step 3로 분리.
- **rebase --onto fa9d5fa HEAD에서 ADR-025 누락**: `git rebase --onto 3a8e17e fa9d5fa HEAD`는 `fa9d5fa..HEAD` 범위만 옮김. ADR-025 commit이 `fa9d5fa`의 *부모*라 범위에 안 들어가 사라짐. reflog로 복구 → cherry-pick. `git rebase --onto`의 upstream이 exclusive라는 점을 다시 새김.
- **issue #158 누락**: PR 시작 시점에 issue 검색 안 한 게 원인. plan 단계에서 GitHub issue 훑기 컨벤션화 가치.

## 배운 것

### silent drift는 단일 사고가 아니라 패턴
두 사고(#142, #176)로 시작 → review 단계 발견 5개 + audit = 총 7개. *공통 원인*:
1. Hibernate 자동 추론(dialect 길이 추론, 자동 hash 이름, 자동 CHECK 생성)에 암묵 의존
2. 엔티티가 도메인 의도를 명시하지 않음

해결 패턴: 표준 type code 명시 + 컨벤션 이름 명시 + 의도적 제거(CHECK는 application layer로).

### 외부 LLM review가 카테고리 확장 트리거
Internal plan만으로는 7개까지 못 갔을 것. silent drift 1·2(#142, #176)가 이미 있어도 본 PR plan은 그 둘만 인용. Gemini의 한 사건 지적(outbox payload)이 *카테고리 자체를 재검토*하게 만들어 6개 더 발견. → 자체 review만 하지 말고 외부 LLM review를 의식적으로 트리거하는 가치.

### `@Lob`의 실제 동작 ≠ 직관
`@Lob String + length 미지정` → tinytext. 의도("큰 텍스트")와 정반대 매핑. dialect 동작에 silent 의존하면 안 됨.

### outbox payload는 generic String 통과가 패턴 본질
JSON 컬럼 + 객체 매핑 검토 끝에 도달. *generic relay*가 payload schema 모르고 통과시키는 게 본질. 객체 매핑은 sealed interface 추상화 강요 → outbox 일반성 깨뜨림. *역설적으로 String이 더 확장성 좋음*.

### DEFAULT는 DB 책임, NOT NULL은 엔티티 의도
JPA @Column에 default 속성 없는 게 그 분리를 반영. version DEFAULT 0은 V1만의 안전망, 엔티티는 @Column(nullable=false) + JPA 자동 set에 위임.

### fragile dependency 메모는 plan trigger
ADR-023이 그 의존을 *명시*했는데 plan에서 활용 못함. 향후 ADR의 "fragile dependency", "depends on X", "couples with Y" 같은 메모는 *plan 단계 의존 그래프 trigger*로 의식적으로 훑어야. 회고 단계가 아니라 plan 단계에서.

### mysqldump raw 결과 ≠ V1 최종 형태
worker가 도메인 순서/일관 collation/audit NOT NULL/version DEFAULT 0 등으로 정리한 결과가 V1. *다시 dump 뜨면 회귀*. 엔티티가 의도를 코드에 명시해야 정합 유지.

### text vs mediumtext
가변 길이라 디스크 비용 차이 거의 없음. 한도 여유는 silent drift 방지 측면에서 오히려 안전. SqlTypes.LONGVARCHAR가 mediumtext일 거라 추정 → 실제는 text. *dump 떠보기 전엔 모름*.

### git rebase --onto upstream은 exclusive
`UPSTREAM..HEAD` 범위만 옮김. UPSTREAM 자체와 그 부모는 제외. ADR-025처럼 UPSTREAM의 *부모* commit이 본의 아니게 history에서 사라질 위험. reflog 복구 가능하지만 의도하지 않은 사고.

## 참고

- PR: KimYeonWook511/commerce-backend#184
- 정본 ADR: `commerce-backend/docs/ADR.md` (ADR-024, ADR-025, ADR-023 후속 메모)
- 회고: `commerce-backend/docs/tasks/flyway-introduction/retrospective.md` ("Review 단계에서 정리한 silent drift 사례들" 섹션 + 메타 관찰)
- Closes: #158
- 인용된 두 사고: #142 (ADR-018 연계), #176 (ADR-023 연계)
- Gemini review thread: PR #184 outbox payload 지적
