---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [jpa, ddd, orm, persistence, first-level-cache, batch, knowledge]
created: 2026-06-08
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-08-jpa-benefits-limits-ddd-friction]]"
---

# JPA 메커니즘·이점·한계와 DDD 괴리 — 실무 트레이드오프 정리

## 한 줄 정의

JPA의 본질은 **"DB(테이블/SQL)를 객체(그래프/상태)로 다루게 해주는 것"**이고, 그 이점이 도메인 성격에 좌우된다. 결제 도메인 논의에서 파생된, 플랫폼 무관하게 재사용 가능한 JPA/DDD 일반 지식이다. 이 지식을 근거로 삼은 결정은 [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]](지식↔결정 짝).

## JPA 메커니즘 오해→정정 — 1차캐시≠write-behind≠batch

- **1차 캐시 ≠ write-behind ≠ batch.** 셋은 다른 메커니즘이다.
  - 1차 캐시(영속성 컨텍스트): 같은 트랜잭션 안에서 같은 엔티티 재조회를 메모리로 처리.
  - 쓰기 지연(write-behind): 변경/INSERT를 flush(기본은 커밋 시점) 때 모아 전송.
  - batch: 모인 SQL을 JDBC batch로 묶어 1 RTT로. **별도 설정(`batch_size`) 필요, 기본은 OFF.**
- **batch insert는 `@GeneratedValue(IDENTITY)`면 불가능하다.** INSERT를 해야 ID를 받으니 모을 수가 없다. 우리 엔티티가 IDENTITY 전략이라 INSERT batch 이점은 못 누린다.
- **"flush 한 번" ≠ "1 RTT".** flush 시점에 UPDATE가 3개면 기본은 각각 3 RTT다. 1 RTT로 묶으려면 batching 설정 + 같은 종류로 정렬(`order_updates`)이 필요하다. UPDATE/DELETE는 IDENTITY와 무관하게 batching 가능하지만 역시 설정이 있어야 한다.
- **같은 엔티티를 한 트랜잭션에서 여러 번 수정해도 flush가 한 번이면 UPDATE는 한 번.** 더티 체킹은 "변경할 때마다"가 아니라 flush 시점에 "최종 상태"를 스냅샷과 비교해 SQL을 만든다. 단 그 사이에 flush(`saveAndFlush`, JPQL 실행 등)가 끼면 쪼개진다. 기본 UPDATE는 전체 컬럼, `@DynamicUpdate`면 변경된 컬럼만.

## 1차 캐시 = 트랜잭션 내 동일성 보장 (lost update 방지), 성능 캐시가 아님

- 흔한 불편 지적("같은 엔티티를 재조회하면 그 사이 DB가 바뀌었어도 처음 스냅샷을 준다")은 **맞는 단점**이다. 그러나 본질은 성능이 아니라 동일성이다.
- 같은 트랜잭션의 같은 행 = **같은 객체 인스턴스**(`o1 == o2`). 둘을 다 변경해도 한 객체이므로 **lost update가 방지된다.** 이게 성능보다 중요한 역할이다.
- 게다가 MySQL 기본 격리수준 REPEATABLE READ에선 1차 캐시를 비우고 DB를 재조회해도 어차피 같은 스냅샷이라 같은 값이 나온다 — "이상한 옛날 값"이 아니라 "격리수준과 일관된 값"을 메모리로 빠르게 주는 것이다. 최신값이 꼭 필요하면 `em.refresh()` / `em.clear()` / 새 트랜잭션으로 탈출한다.

## 트랜잭션 내 row 존재 재확인이 안 되는 이유 — MVCC 스냅샷, INSERT+unique에 맡기기

- 1차 캐시를 비워도 **MVCC 스냅샷** 때문에 그 사이 다른 트랜잭션이 커밋한 INSERT는 안 보인다.
- 애초에 "한 트랜잭션 안에서 외부의 실시간 변화를 감지"하려는 것 자체가 트랜잭션(일관된 세계를 봄)의 목적과 충돌한다 → 보통 설계가 어긋난 신호다.
- 해법: "있으면 쓰고 없으면 생성"이면 SELECT로 확인하지 말고 **INSERT 시도 + unique 제약**에 맡긴다(write는 현재 상태와 충돌하므로 정확). 이 원리가 [[find-first-write-not-check-db-unique-멱등]]·[[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]의 S-lock 대기와 직결된다. 정말 최신 조회가 필요하면 짧은 트랜잭션·`FOR UPDATE`(current read)·READ COMMITTED, 생길 때까지 대기해야 하면 앱 레벨 폴링/이벤트(트랜잭션을 열어둔 채 기다리지 않는다).

## JPA가 빛나는 곳 vs 안 먹히는 곳 — 결제 비교표

| 이점 | 빛나는 조건 | 결제(append-only, IDENTITY, FK 없음) |
|---|---|---|
| 객체 그래프 탐색 | 연관이 풍부 | FK 안 걺 / ID 참조 → 탐색 없음 ✕ |
| 더티 체킹 | 부분 수정 잦음 | 상태 전이 몇 개, INSERT 위주 △ |
| 1차 캐시(동일성) | 같은 객체 재조회 | 트랜잭션당 조회 1번 → 이득 0 ✕ |
| 벤더 독립 | DB 바꿀 가능성 | NULL 트릭 등 MySQL 특화 ✕ |
| CRUD 생산성 | 단순 CRUD 많음 | 단순 조회만 누림 △ |
| 생명주기/cascade | 애그리거트 부모-자식 | 단순 구조 △ |

JPA는 연관이 풍부하고 상태 변경·재조회가 잦고 애그리거트 구조인 도메인(회원/주문/카탈로그/일반 CRUD)에서 강력하다. 결제는 append-only + 단순 연관 + 재조회 없음 + 정합성 critical이라 JPA 자랑이 거의 다 비껴가고, 오히려 SQL 명시성(JDBC/MyBatis)이 잘 맞는 도메인이다.

## DDD↔JPA 괴리 — 한 클래스 두 주인, 실용진영 vs 순수진영

- 괴리의 뿌리는 **한 클래스에 두 주인**이다. DDD는 "도메인 객체는 영속화를 몰라야 한다"고 하는데, JPA는 "이 객체는 내가 관리하는 영속 엔티티다"라고 한다. 같은 `Payment`가 순수 도메인이면서 동시에 JPA 엔티티여야 한다.
- 증상이 전부 여기서 나온다: save를 부를까 말까(더티 체킹), `@Entity` 침투, `DataIntegrityViolationException` 번역, FK를 걸까 ID로 참조할까.
- **실용 진영(엔티티=도메인)**: 한 클래스가 둘 다. 단순·생산성 높지만 도메인 오염 감수. 실무 대부분.
- **순수 진영(도메인 ≠ 영속성 모델)**: 순수 도메인 + JPA 엔티티 분리, adapter 매핑. 순수하지만 매핑 보일러플레이트 폭발.
- **둘 다 비용이 있어 "해결"이 아니라 트레이드오프다.** 실무 중도는 실용 진영 기본 + 오염을 도메인 메서드 캡슐화 / 명시적 save / 예외 격리 / FK 안 걸고 ID 참조로 가두는 것 — 이 적용판이 [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]]이다. 명시적 save는 추세(특히 DDD/헥사고날 팀): 반영 의도가 드러나고 detached 사고를 막고 흐름이 바뀌어도 안전하다.

## 열린 질문 / 관련 링크

- **IDENTITY → 시퀀스/테이블 전환:** batch insert를 누리려면 필요하나 MySQL은 시퀀스가 없어 테이블 전략을 써야 하고 비용이 있다. 결제는 트랜잭션당 1~2행이라 batch 실익이 없어 당장 전환 이유 없음.
- 관련: [[cross-aggregate-fk-to-id-참조-컨벤션]] · [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]] — "FK 안 걸고 ID 참조" 오염 가두기 축. [[payment-append-only-원장과-exists-완료판단]] — 결제 append-only 성격. [[payment-도메인-구조-개요]] — 결제 도메인 개요.

## 근거

- [[raw/sessions/backend/2026-06-08-jpa-benefits-limits-ddd-friction]] — JPA 메커니즘 오해 정정, 1차 캐시=동일성, MVCC 재확인 불가, 빛나는 곳/안 먹히는 곳 비교표, DDD↔JPA 두 주인 트레이드오프의 출처.
