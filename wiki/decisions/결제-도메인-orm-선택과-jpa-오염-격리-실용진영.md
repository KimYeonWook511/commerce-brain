---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [jpa, ddd, orm, payment, persistence, convention, flush]
created: 2026-06-08
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-08-jpa-benefits-limits-ddd-friction]]"
---

# 결제 도메인의 ORM 선택과 JPA 오염 격리 — 실용진영 유지, 명시적 save 컨벤션

## 컨텍스트 — flush·예외 복잡함이 DDD 탓인가라는 질문

flush 타이밍·예외 경계 논의([[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]])가 깊어지면서 "JPA는 변수가 너무 많은 것 아닌가, DDD를 지키려니 이렇게 되는 건가?"라는 질문에 부딪혔다. 그 답으로 프로젝트(commerce-backend) 차원의 원칙을 확정했다. 일반 지식 배경(JPA 메커니즘·DDD 괴리)은 [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]]로 분리하고, 이 노트는 결제 도메인에 대한 **결정**만 담는다.

## 결정 — 복잡함의 원인을 JPA 영속성컨텍스트 + 특수 요구로 규정, 결제 한 지점 격리

결제 저장에서 flush 타이밍·예외 번역이 복잡한 건 **DDD 때문이 아니라 JPA의 영속성 컨텍스트(flush 지연) + 우리가 특수한 것을 하기 때문**이다. 그 특수한 것 둘:

1. **unique 위반 즉시 능동 처리** — 결제 승인 저장 전용 경로가 `saveAndFlush`의 조기 flush로 unique 위반을 그 호출 안에서 즉시 확정하고 adapter가 도메인 예외로 매핑한다(정상 흐름을 사전 조회에 맡기지 않는 carve-out).
2. **MySQL NULL 트릭** — "한 주문에 성공 결제 하나"를 강제하는 부분 unique 제약을, MySQL InnoDB가 partial unique index를 지원하지 않아 조건을 만족하지 않는 행의 키 컬럼을 NULL로 두어 unique 대상에서 제외하는 방식으로 구현한 것([[payment-이중결제-reserve따닥-mysql-null트릭-unique]]).

이 두 요구가 결제 승인 저장 한 지점에만 있으므로 복잡함도 거기 가둔다.

## 결정 — 엔티티=도메인 실용진영 유지 + 오염 4가지로 가둠

DDD ↔ JPA 괴리는 "해결"이 아니라 "어느 비용을 택하느냐"다. 순수 도메인 객체와 JPA 엔티티를 분리하는 순수 진영은 매핑 보일러플레이트가 폭발한다. 실무 대부분처럼 한 클래스가 도메인이자 엔티티인 **실용 진영을 기본**으로 두고, 그 오염을 네 가지로 가둔다:

- (a) **도메인 메서드 캡슐화** — 상태 전이를 도메인 메서드 뒤로.
- (b) **명시적 save** — 아래 별도 결정.
- (c) **예외 격리** — 인프라 예외를 adapter 경계에서 도메인 예외로 번역([[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]).
- (d) **FK를 걸지 않고 ID 참조** — [[cross-aggregate-fk-to-id-참조-컨벤션]] / [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]] series 전체와 직접 연결.

## 결정 — 명시적 save를 컨벤션으로

더티 체킹의 암묵성에 기대지 않고 명시적 save를 컨벤션으로 둔다. managed 엔티티에 대한 `repository.save()`는 JPA 내부에선 no-op이지만, "이 시점에 저장한다"는 의도를 코드 표면에 드러내고 응용 계층의 사고 모델을 특정 ORM에서 떼어낸다. **save 계약은 "영속화한다"까지이고 flush 시점은 adapter/트랜잭션 정책**이다. 즉시 검증이 필요한 특수 케이스만 별도 계약 메서드(`saveAndFlush` 기반)로 판다. "DDD 지키니 변수 많다"의 반전이 여기 있다 — 명시 save를 컨벤션으로 박으면 "save를 부를까 말까"라는 더티 체킹의 암묵성이 사라져 오히려 변수가 준다.

## 결정 — 결제는 JPA 이점 약하나 일관성 위해 유지, JDBC/MyBatis 전환은 재판단 열어둠

결제는 append-only + 단순 연관 + 재조회 없음 + 정합성 critical이라 JPA 자랑(1차 캐시·지연로딩·write-behind·batch)이 거의 다 비껴간다. 남는 이득은 "단순 조회의 편함 + 다른 도메인과의 일관성"뿐이다. flush에 예민한 건 그 한 지점(unique 위반 즉시 처리)뿐이라 선택지는 ① JPA + `saveAndFlush`(그 메서드만) ② 그 쿼리만 `JdbcTemplate` ③ 결제 전체를 JDBC/MyBatis로. **지금은 ①.** 다른 도메인(주문/회원)이 JPA를 충분히 활용 중이라 일관성 위해 유지하되, "이득 없고 1차 캐시가 오히려 불편"이 충분히 강해지면 결제만 JDBC로 빼는 것도 정당하다. 도메인마다 ORM이 달라도 된다.

## 트레이드오프 — 실용진영(오염) vs 순수진영(매핑 보일러플레이트 폭발)

| 진영 | 이득 | 비용 |
|---|---|---|
| 실용(엔티티=도메인, 택함) | 단순·생산성 높음, 다른 도메인과 일관 | 도메인 오염(한 클래스에 두 주인) — 4가지로 가둬 완화 |
| 순수(도메인 ≠ 영속성 모델) | 도메인 순수성 | 순수 도메인 + JPA 엔티티 분리 매핑 보일러플레이트 폭발 |

둘 다 비용이 있어 해결이 아니라 트레이드오프다. 실무 중도는 실용 진영 기본 + 오염 가두기다.

## 미해결

- **batch를 위한 IDENTITY→시퀀스/테이블 전환:** batch insert를 누리려면 ID 전략을 IDENTITY에서 바꿔야 하는데, MySQL은 시퀀스가 없어 테이블 전략을 써야 하고 그것도 비용이 있다. 다만 결제는 트랜잭션당 변경이 1~2행이라 batch 실익이 없어 지금 전환할 이유는 없다.
- **결제만 JDBC/MyBatis로 뺄지 여부:** "결제에서 JPA 이득이 거의 없고 1차 캐시가 오히려 불편"이 충분히 강해지는 시점에 재판단으로 남긴다.

## 근거

- [[raw/sessions/backend/2026-06-08-jpa-benefits-limits-ddd-friction]] — 복잡함 원인 규정, 실용진영 + 오염 4가지 가두기, 명시적 save 컨벤션, 결제 ORM 유지·재판단 조건의 출처.
