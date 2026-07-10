---
platform: backend
author: KimYeonWook511
created: 2026-06-02
origin:
  - { type: issue, repo: commerce-backend, ref: 195 }
---

# Issue #195 — FK → Long ID 마이그레이션 motivation 정리

## 결정한 것 — 왜 지금, 왜 이걸 먼저

### 멀티모듈은 미루고 FK 제거부터

멀티모듈은 학습이 안 돼서 도입 부담이 있다. 그런데 멀티모듈로 가려면 어차피 모듈 간 JPA 연관을 끊어야 하니까, **FK 제거가 멀티모듈의 선행 작업**이 된다. 학습 부담을 분산하려고 우회로를 찾았는데, 그 우회로 = 선행 작업이라는 인식으로 이번 트랙을 결정.

→ 멀티모듈을 *건너뛰는* 게 아니라 *준비 단계를 먼저 끝내두는* 것. 나중에 멀티모듈로 갈 때 모듈 경계 긋기와 build.gradle 분리 정도만 남는다.

### 3가지 변경의 영향 분리

"FK 제거" 라고 한 덩어리로 부르지만 실제는 셋:

| 변경 | DB 변경 | 코드 수정 |
|---|---|---|
| DB FK 제약 DROP | 스키마 변경 | **JOIN 쿼리 영향 0** (FK 는 무결성용, JOIN 의 전제 아님) |
| `@ManyToOne` → `Long fooId` | 없음 | join fetch / QueryDSL path / `entity.getOther()` 다수 수정 |
| cross-aggregate `@OneToMany` 제거 | **없음** | 호출부만 `findByYyyId(...)` 로 교체 |

이 분리가 PR 단위 결정의 기준. 같은 aggregate 내부 (Order ↔ OrderItem) 는 풀지 않는다.

### 도메인 간 의존 통제 — 패턴 A 채택

FK 끊으면 도메인 간 의존이 늘어나 보이지만, 실제로는 **JPA lazy loading 뒤에 숨어 있던 의존이 드러나는 것**. 총량은 비슷하고 통제 가능성이 올라간다. 이 프레임 없으면 "ID 참조로 가면 코드 더러워진다" 표면 반응에 휘말리기 쉽다.

처음 들었던 예시는 패턴 B (Payment 가 Order 의 port 구현, DIP 강하게 적용) 였는데 **commerce 에는 패턴 A 가 자연스럽다**:

- Order 가 자기 adapter 들고 Payment 의 공개 facade 호출
- 의존 방향: `Order → Payment` (Payment 는 자기를 누가 쓰는지 모름)
- Payment 는 "사실을 제공하는 쪽", Order 는 "그걸 소비하는 쪽"

핵심은 패턴 선택이 아니라 (1) Order 가 Payment 엔티티를 직접 import 안 함, (2) 의존 방향이 한 방향 — 멀티모듈 시 cycle 방지가 본질.

### FK 단점 — ADR-020 한 줄 뒤에 책의 영향이 깔려 있다

ADR-020 본문에 누적 부채를 7가지 정리해뒀지만 (`commerce-backend/docs/ADR.md` 참조), 그중 **"DDD 정통 원칙 위반"** 한 줄이 사실 "도메인 주도 개발 시작하기" (최범균) 3장 3.5 절 전체의 압축이라는 걸 이번에 다시 인식했다.

책의 직접 참조 문제 3가지가 ADR-020 의 여러 항목에 동시에 걸쳐 있음:

- **편한 탐색 오용** → 결합도 증가의 *본질*. PR #192 (보상 정책 판단을 Payment 도메인 사실 조회로 정리) 가 이 문제의 실제 사례. 직접 객체 참조 시 사실 판단이 객체 traversal 뒤로 숨어 책임 경계가 흐려졌다.
- **성능 고민** → N+1 / fetch join 부담
- **확장 어려움** → 미래 마이크로서비스 분리 경로

→ ADR 의 "원칙 위반" 같은 추상 항목은 *구체적 학습 출처* 와 함께 읽어야 motivation 의 깊이가 산다. 5번만 아니라 #2 · #3 · #5 · #7 이 책의 영향 아래 묶여 있었음.

### 추가 통증

**FK 가 정책을 강제로 결정** — `tbl_order_item → tbl_product` FK 때문에 상품 hard delete 가 불가능해 soft delete 가 강제됐다. FK 가 비즈니스 정책을 외부에서 결정해버리는 구조. (`docs/tasks/product-management/adr.md`)

### 진행 방식

도메인 경계 단위 PR 로 점진. 한 번에 풀지 않는다. 첫 PR 단위는 코드 스캔 후 결정 (아래 다음 단계 참조).

- 워밍업 후보: cross-aggregate `@OneToMany` 제거 — DB 안 건드리고 호출부만 교체라 가장 단순
- DB FK 실제 DROP 은 모든 코드 마이그레이션 완료 후 마지막에 일괄
- 운영 DB 적용은 별도 결정 (FK 의 안전망 가치 vs 코드 일관성 가치 재평가 필요)

## 다음 단계

- 첫 PR 단위 결정 — 현재 코드의 `@ManyToOne` / `@OneToMany` 분포 스캔 후 가장 단순한 경계 고르기. cross-aggregate `@OneToMany` 만 있는 곳이 있다면 워밍업으로 적합.
- **deferred 트레이드오프**: 운영 DB 의 FK 안전망 가치 vs 코드 일관성 가치. 사용자가 SQL 로 직접 손대는 환경이 아니라면 application 책임으로 옮기는 게 멀티모듈 미래까지 일관됨. 다만 마지막 결정 시점에 다시 본다.

## 참고

- Issue #195: refactor: 기존 도메인의 cross-aggregate 참조를 Long ID로 통일
- ADR-020: `commerce-backend/docs/ADR.md` "신규 도메인의 cross-aggregate 참조는 ID로 한다"
- PR #192: 보상 정책 판단을 Payment 도메인 사실 조회로 정리 (편한 탐색 오용 사례)
- 도메인 주도 개발 시작하기 (최범균) 3장 애그리거트, 3.5 "애그리거트를 ID로 참조"
