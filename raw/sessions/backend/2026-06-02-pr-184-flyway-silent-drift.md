---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: pr, repo: commerce-backend, ref: 184 }
---

# Flyway 도입 — silent schema drift를 부팅 실패로 가시화하고 review 단계에서 8건 일괄 정리

DB는 단일 MySQL 하나뿐이고 다중 DB 계획도 없어, 그동안 JPA `ddl-auto: update`로 충분하다는
입장으로 Flyway 도입을 미뤄왔다. 그런데 최근 두 사고(#142 ENUM silent drift, #176 multi-column
unique silent 미적용)가 *같은 패턴* — "코드는 정상으로 보이지만 실제 DB 스키마가 조용히 어긋난
상태" — 임을 보여줘 입장이 뒤집혔다. 이 PR에서 Flyway를 도입해 `ddl-auto`를 `validate`로 전환하고,
스키마를 현 엔티티 기준 단일 베이스라인(V1)으로 굳혔다. 그 과정의 review 단계에서 같은 결의 silent
drift 8건을 함께 발견해 일괄 정리했다. Flyway 도입과 enum 컬럼에 DB CHECK 제약을 두지 않기로 한
결정을 이 PR의 두 신규 ADR로 남겼고, 결정의 세부 근거는 그 두 ADR과 이 태스크 회고 문서를 정본으로
둔다 — 여기엔 "내 이해·왜·다시 본다면"을 담는다.

## 결정한 것

### 1. Flyway 도입 + `ddl-auto: validate` 전환 — 입장이 뒤집힌 이유

핵심은 두 사고가 *같은 패턴*이라는 인식이었다. 사고 1(#142)은 Hibernate 6.x가
`@Enumerated(STRING)`을 MySQL native ENUM으로 매핑하도록 dialect가 바뀌면서, `ddl-auto: update`로
NOT NULL ENUM 컬럼을 추가할 때 기존 row에 의도치 않은 첫 번째 값이 조용히 채워진 사건이다. 사고
2(#176)는 `tbl_payment_attempt`의 4-column unique key 대상 컬럼들이 길이 미지정으로 utf8mb4에서
InnoDB 한도(3072 bytes)를 넘어 MySQL이 unique 생성을 거부했는데, Hibernate 기본 핸들러가 WARN으로만
로그하고 부팅을 계속해 unique가 빠진 채 운영된 사건이다.

- **핵심 인식:** 단일 DB라도 drift는 일어난다. 원인이 다중 DB나 팀 간 분기 같은 *외부* 요인이 아니라
  dialect 변경 / DDL silent fail 같은 *코드 내부* 요인이라는 점이 결정적이었다. "단일 DB라서 Flyway가
  필요 없다"는 입장은 이 두 사고를 나란히 본 시점에서 무너졌다.
- **시점의 우위:** 운영 DB가 아직 미가동이라 V1 단일 스크립트로 출발할 수 있다. 운영이 돌기 시작하면
  baseline-on-migrate, 운영 dump → baseline 작성 → checksum 검증 같은 도입 절차가 모두 필요해진다.
  지금은 그게 다 불필요하다 — 도입이 늦어질수록 절차 비용이 기하급수로 는다.
- **검토한 대안:**
  - **`ddl-auto: update` 유지** — 두 사고의 silent drift를 그대로 안고 간다. 코드 변화량에 비례해
    재발 위험. 기각.
  - **`validate`만 켜고 마이그레이션은 손으로 SQL 관리** — validate는 컬럼/타입 누락을 부팅 실패로
    잡아 사고 1 유형은 방어하지만, 적용 순서·이력이 코드 밖으로 새고 환경 간 일관성이 사람 기억에
    의존한다. 사고 2 유형(unique 누락)은 validate가 unique 제약을 검사하지 않아 여전히 못 잡는다. 기각.
  - **Liquibase** — XML/YAML DSL 추상화. MySQL/JPA 단일 스택에서 추상화 가치가 제한적이고, SQL을
    그대로 다루는 게 디버깅·리뷰에 유리해 기각.
  - **택함: Flyway** — SQL 그대로 버전 관리 + Spring Boot auto-config 내장. (a) validate로 컬럼/타입
    누락 가시화, (b) 명시적 스크립트로 변경 의도 코드화, (c) `flyway_schema_history`로 환경 간 일관성
    추적 — 세 축으로 drift 패턴을 해소한다.
- **적용 방식:** 운영/로컬/integrationTest는 `validate` + Flyway 활성. test(H2)는 create-drop + Flyway
  비활성 유지(단위/슬라이스 테스트의 부팅 속도 자산이고, V 스크립트가 MySQL 문법이라 H2에 직접 적용
  불가). integrationTest(Testcontainers MySQL)는 Flyway 활성 — 컨테이너 싱글톤 재사용 + 컨텍스트 캐싱
  + `deleteAllInBatch()` 격리 모델과 자연스럽게 맞물린다. Spring Batch 메타테이블은 Flyway 관리에서
  제외하고 Batch 자체 `initialize-schema`를 유지(버전업 시 마이그레이션 책임이 프로젝트로 넘어오는
  비용 회피). 운영 안전망으로 `flyway.clean-disabled: true` 명시.
- **인정한 비용:** 도입을 미뤄온 가장 큰 이유였던 "운영 복잡성"이 실제로 는다 — 엔티티를 바꾸면 같은
  PR에서 마이그레이션 스크립트를 함께 써야 하고, 로컬에서 엔티티만 고치고 부팅하면 실패한다. 두
  사고의 silent drift 비용이 이 복잡성보다 크다고 봐 수용했다.
- **validate가 잡지 못하는 것:** validate는 컬럼/타입 누락은 잡지만 unique·인덱스 누락은 검사하지
  않는다. 사고 2 유형은 Flyway 도입 후에도 unique 대상 컬럼 길이 명시(#176 후속 코드 규칙)로 1차
  방어한다. Flyway는 "변경 이력이 명시적이라 리뷰에서 잡힐 가능성을 높인다"는 간접 효과로만 기여한다.

### 2. outbox payload — JSON 컬럼 검토 끝에 `SqlTypes.LONGVARCHAR`

review에서 outbox 이벤트 payload 컬럼 매핑을 손봤다(Gemini Code Assist의 `tinytext` 지적이 발단).
`OutboxEvent` 엔티티가 `@Lob` + length 미지정이라 Hibernate가 dialect 추론으로 `tinytext`(255 bytes)로
매핑하고 있었다 — "큰 텍스트"라는 의도와 정반대다. `@Lob`을 떼고 `@JdbcTypeCode(SqlTypes.LONGVARCHAR)`로
표준 type code를 명시했다.

- **JSON 컬럼을 먼저 검토했다:** `@JdbcTypeCode(SqlTypes.JSON)` + String은 Hibernate가 ObjectMapper로
  (de)serialize하며 double-quote 이스케이프 사고 위험이 있다 — JSON 컬럼을 제대로 쓰려면 객체 매핑이
  정석이다.
- **그런데 객체 매핑이 outbox 본질과 충돌:** 객체 매핑하려면 payload가 generic이라 sealed
  interface/Object/Map을 강요하게 되는데, 이는 outbox의 본질(payload schema를 모르고 그냥 통과시키는
  generic relay)을 깨뜨린다. 또 `columnDefinition = "json"`으로 박는 건 엔티티에 MySQL 전용 SQL을
  넣는 것이라 dialect 추론 의존을 끊자는 원칙에 어긋난다.
- **결론:** outbox는 *generic String 통과가 패턴 본질*이다. producer가 `writeValueAsString`으로 만든
  결과는 정의상 valid JSON이라 컬럼 레벨 JSON 검증의 가치도 작다. 그래서 표준 LONGVARCHAR로 갔다.
- **실측 반전:** LONGVARCHAR가 `mediumtext`로 매핑될 거라 추정했는데, dump를 떠보니 실제로는
  `text`(64KB)였다(V1의 payload 컬럼이 `text NOT NULL`로 확인됨). outbox payload(1~5KB)엔 충분한
  여유다. *dump 떠보기 전엔 모른다*.

### 3. `@Version` 컬럼 — `Long` + `nullable=false`가 모순이 아닌 이유

Order/Stock/CartItem 3개 엔티티가 `@Version private Long version;`만 명시해 Hibernate 매핑이
`bigint`(nullable)로 떨어졌다. V1 스크립트엔 도메인 의도(`NOT NULL DEFAULT 0`)가 박혀 있는데 엔티티엔
그 의도가 없던 silent mismatch다. `@Column(nullable = false)`를 추가했다.

- **직관적 모순의 해소:** `Long`(null 가능) + `@Column(nullable=false)`는 얼핏 모순 같지만, `@Version`은
  INSERT 직전 Hibernate가 0L로 자동 set하므로 정상 JPA 흐름에선 NULL이 DB로 갈 일이 없다.
- **layer 분리:** *NOT NULL은 엔티티의 의도, DEFAULT는 DB의 책임*이라는 분리다. JPA `@Column`에
  default 속성이 없는 것이 바로 이 분리의 반영이다. V1의 `DEFAULT 0`은 raw INSERT나 외부 도구 우회를
  위한 DB 쪽 안전망으로 유지하고, 엔티티는 `nullable=false` + JPA 자동 set에만 신경 쓴다.

### 4. `halt_on_error` 제거 — validate 전환으로 발동 조건이 사라짐

직전 결정(multi-column unique 대상 컬럼에 `@Column(length=...)`을 명시한 #176 대응)에서 스키마 회귀를
부팅 시점에 노출시키려고 `application-local.yml`에만 `hibernate.hbm2ddl.halt_on_error: true`를 켰었다.
이 PR에서 재검토해 제거했다.

- **왜 사라졌나:** `validate`로 전환하면 Hibernate가 DDL을 실행하지 않으므로 `halt_on_error`의 발동
  조건(Hibernate DDL 실행 중 실패) 자체가 없어진다. 스키마 변경 실패 차단 책임은 이제 Flyway가
  가져간다(마이그레이션 SQL 실패 시 Flyway가 부팅을 막는다).
- **놓쳤던 지점:** 직전 결정이 "`halt_on_error`는 local `ddl-auto: update` 전제에 묶이며, local
  ddl-auto를 바꿀 때 함께 재검토해야 한다"는 fragile dependency를 *명시했음에도*, 이 PR의 plan 단계에서
  그 메모를 의존 그래프 trigger로 활용하지 못했다. 회고/외부 조언으로 review 단계에서야 잡았다.

### 5. audit(`created_at`/`updated_at`) `NOT NULL` 명시

`BaseTimeEntity`(생성/수정 시각을 담는 공통 상위 엔티티)가 `@Column(nullable = false)`를 누락해 V1에서
audit 컬럼이 `DEFAULT NULL`로 떨어져 있었다. *생성/수정 시각은 항상 채워진다*는 도메인 의도가 코드에
표현되지 않은 상태 — 같은 결의 silent mismatch다. `BaseTimeEntity`에 `@Column(nullable = false)`를
추가하고 V1의 audit 컬럼도 `datetime(6) NOT NULL`로 정리했다.

### 6. enum 컬럼의 DB CHECK 제거

Hibernate가 `@Enumerated(STRING)` 컬럼에 자동 생성한 `CHECK (column in (...))` 제약이 V1에 굳어 있었다.
validate는 CHECK 내용을 검증하지 않으므로, Java enum에 새 값을 추가하면 validate는 통과시키지만 그
값으로 INSERT하는 순간 CHECK 위반으로 런타임 실패하는 silent mismatch가 잠재했다. 이는 앞선 사고들과
같은 결의 함정이라, V1의 CHECK 제약을 모두 제거하고 enum 유효성 보장을 애플리케이션 layer(Java enum
타입 시스템 + `@Enumerated(STRING)`)에 위임하기로 결정했다(이 PR의 신규 ADR로 남김).

- **이중 안전망의 실용 가치가 작다:** 단일 백엔드 + JPA 단일 INSERT 경로 + 외부 시스템 직접 INSERT
  없음. `@Enumerated(STRING)`이 invalid enum을 application layer에서 이미 차단해 DB CHECK는 실질적으로
  발동될 일이 없다.
- **enum 진화 마찰:** 결제 fail_code·주문 status·이벤트 type 등 enum은 도메인이 자라면 종종 추가되는데,
  CHECK를 유지하면 추가마다 마이그레이션 스크립트가 필요하다.
- **자동 생성이라 의식 안 됨:** 두는 것도 안 두는 것도 의식적이어야 하는데 Hibernate가 자동으로 만들어
  버려 의식되지 않는다. 이 결정은 그 자동 동작을 명시적으로 우회하는 의미다.
- **검토한 대안:** (A) CHECK 유지 — 이중 안전망이지만 마찰·silent mismatch 위험 그대로, 기각. (B) V1에서만
  제거하고 향후 자동 생성은 방치 — 다음 dump 시 회귀, 기각. (C 택함) 의식적 제거 + Hibernate 차원의 자동
  생성 차단은 후속 task에서 검토.

### 7. 단일 unique / FK 컨벤션 이름 명시

엔티티가 `@Column(unique = true)`/`@JoinColumn(unique = true)`만 쓰면 Hibernate가 자동 hash
이름(예: `UK31to4n8j2vslkf7jvfo408sta`)을, `@JoinColumn(name=...)`만 쓰면 FK도 자동 hash
이름(예: `FKo8mybc2mw82rhti4t1n9i1d0e`)을 생성한다. 이는 스키마 문서가 정한 네이밍 컨벤션 위반이다.

- 단일 unique 5개 엔티티(Member/Order/Stock/Payment/OutboxEvent)에 `@Table(uniqueConstraints=...)`로
  `uk_<target>_<columns>` 컨벤션 이름을 명시하고 V1과 일치시켰다.
- FK 5개 엔티티(Order/OrderItem/Payment/Stock/StockHistory)에
  `@JoinColumn(... foreignKey=@ForeignKey(name="fk_..."))`로 `fk_<source_table>_<source_columns>`
  컨벤션 이름을 명시하고 V1의 FK 6개도 일치시켰다(예: `fk_order_member_id`). FK 컨벤션 자체를 스키마
  문서에 신규로 추가했다.
- CartItem(`member_id`, `product_id`)·StockHistory(`admin_member_id`)처럼 신규 도메인의 cross-aggregate
  참조를 FK 없이 ID로 두기로 한 기존 결정에 따라 의도적으로 FK를 두지 않은 곳은 그대로 유지했다.
- **효과:** 엔티티에 컨벤션 이름을 박아뒀으니 향후 dump를 다시 떠도 자동 hash로 회귀하지 않는다.

### 8. Flyway 도입 후에도 ENUM→VARCHAR 매핑 규칙이 살아남는 진짜 이유 — H2 parity + validate silent zone

처음엔 "Flyway + validate면 ddl-auto가 안 도니까, ENUM을 `@JdbcTypeCode(SqlTypes.VARCHAR)`로 매핑해온
기존 규칙은 이제 무용한 안전망 아닌가?"로 갔다. 사용자가 두 단계로 정정해 규칙의 역할을 재정의했다.

- **1단계 정정 — *기존 row 묻힘*만 본 것:** 그 매핑 규칙에는 두 갈래가 있다. (a) `ddl-auto: update`가
  기존 row에 첫 번째 enum 값을 묻히는 경로는 validate 전환으로 닫혔지만, (b) *새 INSERT 시 NOT NULL
  컬럼을 첫 번째 값으로 silent fill*하는 건 native ENUM 자체 동작이라 ddl-auto와 무관하다 — test/prod
  모두에서 발생 가능하다.
- **2단계 정정 — test 프로파일 H2 parity:** test는 `jdbc:h2:mem:testdb` pure mode + `create-drop`이라
  H2Dialect가 엔티티로 스키마를 만든다. H2Dialect가 native ENUM으로 매핑하면 prod(MySQL + Flyway
  varchar)와 *INSERT 시 NOT NULL 위반 동작*이 갈린다 — test는 silent fill로 통과, prod는 exception.
  이 *환경별 행위 갈림*이 규칙을 유지하는 진짜 비용이다.
- **validate silent zone 발견:** review에서 Codex가 `AsyncTestEntity.status`에 매핑 규칙이 빠진 걸
  지적한 게 계기였다 — 이 규칙이 운영 엔티티들엔 적용됐지만 테스트 지원 엔티티가 빠져 있었다. dockerTest는
  통과했기에, integrationTest를 `--rerun-tasks`로 강제 재실행해 확인하니 Hibernate SchemaValidator가
  enum 매핑(native ENUM) vs 스키마(varchar)의 sql type 차이를 *strict 비교하지 않는* silent zone이
  드러났다. 즉 이 매핑이 빠지면 validate도 못 잡는 silent drift가 잠재한다.
- **재정의:** 매핑 규칙은 무용한 게 아니라 *H2 ↔ MySQL 프로파일 parity + dialect 변경 안전망*으로
  코드 규칙으로 유지한다. `AsyncTestEntity`에 `@JdbcTypeCode(SqlTypes.VARCHAR)`를 추가해 테스트 지원
  엔티티도 같은 규칙을 따르게 했다.

### 9. V900 test support 스크립트 collation 정리

test classpath의 `V900__test_support.sql`(테스트 지원 테이블 `async_test`/`lock_member`용 — 예전엔 H2가
create-drop으로 자동 생성했지만 Testcontainers MySQL + validate 전환 후엔 스크립트가 필요)이 V1과
collation이 어긋나 있었다(`utf8mb4_unicode_ci` + 컬럼 레벨 COLLATE). V1 기준(`utf8mb4_0900_ai_ci`
테이블 default + 컬럼 레벨 COLLATE 제거)으로 맞췄다. 이건 silent drift는 아니고 표면 일관성 문제다.
V900이라는 높은 번호는 운영 마이그레이션과의 충돌을 피하려는 네임스페이스 선택이다.

## 막힌 점·해결

- **harness AC executor가 multi-line shell을 한 줄씩 실행:** step 2 worker의 AC에서 `for ...; do ...;
  done` 블록이 syntax error로 깨졌다. AC executor가 여러 줄 셸 블록을 한 줄씩 실행하는 게 원인.
  단일 라인 명령으로 고치고, 부팅 검증은 "수동 검증 절차"로 옮겼다.
- **step 2 worker가 범위를 넘어 `application-test.yml`까지 수정:** step 범위 위반이라 revert하고 step
  3로 분리했다.
- **rebase 중 enum CHECK 결정 커밋이 사라짐:** `git rebase --onto 3a8e17e fa9d5fa HEAD`는
  `fa9d5fa..HEAD` 범위만 옮기는데, enum CHECK 미사용 결정(이 PR의 신규 ADR)을 담은 커밋이 `fa9d5fa`의
  *부모*라 범위에 안 들어가 사라졌다. reflog로 찾아 cherry-pick으로 복구했다. `git rebase --onto`의
  upstream이 exclusive라는 걸 다시 새겼다.
- **issue #158 누락:** PR 시작 시점에 issue 검색을 안 한 게 원인이다(이 PR은 결국 #158을 close). plan
  단계에서 GitHub issue를 훑는 걸 컨벤션화할 가치가 있다.

## 배운 것

- **silent drift는 단일 사고가 아니라 패턴이다.** 두 사고(#142, #176)로 시작해 review 단계에서 5건 +
  audit까지 총 7건을 더 찾았다. 공통 원인은 (1) Hibernate 자동 추론(dialect 길이 추론, 자동 hash 이름,
  자동 CHECK 생성)에 암묵 의존, (2) 엔티티가 도메인 의도를 명시하지 않음. 해결 패턴은 *표준 type code
  명시 + 컨벤션 이름 명시 + 의도적 제거(CHECK는 application layer로)*.
- **외부 LLM review가 카테고리 확장 트리거였다.** internal plan만으로는 7건까지 못 갔을 것이다 — 이미
  있던 silent drift 사고 둘(#142, #176)조차 이 PR plan은 그 둘만 인용했다. Gemini의 한 건 지적(outbox
  payload)이 *카테고리 자체를 재검토*하게 만들어 6건을 더 발견하게 했다. 자체 review만 하지 말고 외부
  LLM review를 의식적으로 트리거하는 가치.
- **`@Lob`의 실제 동작 ≠ 직관.** `@Lob String + length 미지정` → `tinytext`. "큰 텍스트"라는 의도와
  정반대 매핑이다. dialect 동작에 silent 의존하면 안 된다.
- **outbox payload는 generic String 통과가 패턴 본질.** JSON 컬럼 + 객체 매핑 검토 끝에 도달했다.
  *generic relay*가 payload schema를 모르고 통과시키는 게 본질이라, 객체 매핑은 sealed interface 추상화를
  강요해 그 일반성을 깬다. 역설적으로 String이 더 확장성 좋다.
- **DEFAULT는 DB 책임, NOT NULL은 엔티티 의도.** JPA `@Column`에 default 속성이 없는 게 그 분리를
  반영한다. version `DEFAULT 0`은 V1만의 DB 쪽 안전망이고, 엔티티는 `@Column(nullable=false)` + JPA
  자동 set에 위임한다.
- **fragile dependency 메모는 plan trigger.** 직전 결정이 그 의존을 *명시했는데도* 이 PR plan에서 활용
  못했다. 향후 ADR의 "fragile dependency", "depends on X", "couples with Y" 같은 메모는 *회고가 아니라
  plan 단계에서* 의존 그래프 trigger로 의식적으로 훑어야 한다.
- **mysqldump raw 결과 ≠ V1 최종 형태.** worker가 도메인 순서·일관 collation·audit NOT NULL·version
  DEFAULT 0 등으로 정리한 결과가 V1이다. *다시 dump 뜨면 회귀*한다 — 엔티티가 의도를 코드에 명시해야
  정합이 유지된다.
- **text vs mediumtext.** 가변 길이라 디스크 비용 차이가 거의 없고, 한도 여유는 silent drift 방지 측면에서
  오히려 안전하다. LONGVARCHAR가 mediumtext일 거라 추정했는데 실제는 text였다 — *dump 떠보기 전엔 모른다*.
- **`git rebase --onto`의 upstream은 exclusive.** `UPSTREAM..HEAD` 범위만 옮기고 UPSTREAM 자체와 그
  부모는 제외한다. UPSTREAM의 *부모* 커밋이 본의 아니게 history에서 사라질 위험이 있다. reflog로 복구는
  되지만 의도하지 않은 사고다.

## 미해결·열린 질문

- **enum CHECK 미사용 결정의 재검토 트리거:** 외부 시스템이 같은 DB에 INSERT하는 시나리오(마이크로서비스
  분리, BI/ETL 도구 직접 접근, 운영자 raw SQL 수정)가 일상화되면 application layer만으로는 안전망이
  부족해질 수 있어 재검토가 필요하다.
- **plan 단계 GitHub issue 훑기 컨벤션화:** issue #158을 시작 시점에 못 잡은 걸 계기로, plan 단계에서
  관련 issue를 훑는 절차를 컨벤션으로 굳힐 가치가 있다.
- **Hibernate 차원의 CHECK 자동 생성 차단:** 향후 `ddl-auto: create` 기반 dump 시 Hibernate가 enum
  CHECK를 또 자동 생성하므로 의식적 제거가 계속 필요하다. 설정 또는 엔티티 어노테이션 차원에서 자동
  생성을 끄는 방법은 후속 task로 남긴다.
