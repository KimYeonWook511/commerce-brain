---
type: meta
updated: 2026-08-17
---

# commerce-brain — wiki 카탈로그

wiki 페이지를 타입별로 한 줄 요약한다. **ingest 마다 갱신**한다. 분류의 진실은 각 노트 frontmatter이고, 이 카탈로그는 탐색 보조다.

총 154개 노트 · MOC 32개.

## decisions (118)

### 결제·환불 모델 (2026-08 재설계)

- [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]] — 환불은 독립 aggregate, 한도 판정은 결제가. 합계 조회 대신 누적액 컬럼
- [[한도-기준은-결제사가-실제로-승인한-금액]] — 우리가 기억하는 금액이 아니라 상대가 실제로 처리한 금액
- [[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]] — 예약 테이블 폐지, 구조를 합칠 때 사라지는 방어를 센다
- [[배타점유-슬롯-미리잡기-vs-성공시-감지·되돌리기]] — 돈이 걸린 자리는 막을 수 있으면 막는다
- [[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]] — 비우는 갱신이 삽입보다 먼저 닿아야 한다
- [[전용-종결상태-돈이-나갔다-돌아오는-중]] — 상태 하나가 셋(승인 났다·금액 있다·환불 딸렸다)을 함께 말한다
- [[환불사유-목록값-두-경로-한-사건-회원노출-필터]] — 두 경로를 한 사건으로, 사유로 가르고 회원 노출은 걸러낸다
- [[외부-호출기록-aggregate-밖-낙관락-없는-쌓기전용]] — 판정에 안 쓰이는 기록은 aggregate 밖
- [[돈정합성-우선-경계넘는-트랜잭션-셋-허용과-셀-수-있게]] — 원칙과 부딪히면 돈이 이긴다. 대신 어긴 자리를 파일로 센다
- [[응용계층-서비스-분할-기준-다른-도메인까지-바꿀-때만]] — 기준을 세워도 대상이 안 갈리면 그 기준은 틀린 것
- [[재고복구-트랜잭션-맨뒤-배치와-비관락-경합-계측갈래]] — 락 보유 시간을 줄이려 맨 뒤. 부모 예외 핸들러는 모르는 것만
- [[자식-환불-자기-낙관락-부모버전은-불변식이-바뀔때만]] — 배치가 자식을 만질 때마다 부모 버전이 오르면 사용자를 밀어낸다
- [[도메인-정책-빈-등록-도메인이-설정을-모르게]] — 금지되지 않았는데 안 고른 것은 그 사실을 적어 둔다
- [[결제사-연동타입-인프라-격리와-나가는-호출-읽기제한시간]] — 어댑터 하나, 예외 대신 정해진 결과, 배치도 제한 시간
- [[무결성위반-도메인예외-번역을-제약이름으로-가른다]] — 타입이 아니라 제약 이름. 존재와 생성은 다르다

### 결과 불명·회수 (2026-08)

- [[외부-돈-호출-결과어휘-넷과-전송계층-판정-우선]] — 어휘 넷, 5xx는 "모른다", 응답 코드보다 전송 계층 먼저
- [[결과불명-재호출은-같은-멱등키로-새키-결론-뒤집힘]] — 새 키는 상대의 보호를 스스로 포기하는 것
- [[결과회수-상한-폐지와-백오프-표-통지-반복]] — 멈추면 아무도 안 푼다. 상한 폐지 + 회차별 간격
- [[승인은-다시-물어-확정-환불에는-실패-종착이-없다]] — 환불 실패는 정상 종료가 아니다
- [[잔액대조-옵션-미사용-공급자-권장의-전제-확인]] — 동기 전제 방어를 비동기 구조에 붙이면 지연이 오류가 된다
- [[회수배치가-주문상태를-묻지-않는다-확인대신-전제를-지킨다]] — 확인보다 확인이 필요 없게 만드는 쪽
- [[네이버페이-환불-이력-지목근거-보내기전-우리가-정하는-값]] — 응답에서만 얻는 식별자는 결과를 모를 때 못 쓴다
- [[이중환불-최종방어선-잔액대조-도입]] `superseded` — 잔액 대조를 최종 방어선으로 (기각됨)

### 멱등·유일 제약 (2026-08)

- [[멱등키-세-값-분리와-요청멱등키는-호출자가-발급]] — 멱등키는 하나가 아니다. 유일 범위 = 조회 범위
- [[검증-전에-채우는-외부값에-유일제약-금지]] — 검증 전 외부 값에 제약을 걸면 그 제약이 무기가 된다
- [[동시도착은-선점층이-받고-처리중-전용응답]] — 커밋 전 중복에게 성공으로 답할 수 없다
- [[조회-또는-생성-메서드-해체-조회·생성·기존요청대조-셋]] — 둘로 읽는 순간 세 번째(기존 요청 대조)가 증발한다

### 부분취소 (2026-07 ~ 08)

- [[부분취소-스코프-배송전-품목수량-기반-금액입력-기각]] — 재고 복구 단위가 품목이라 금액 입력이 성립하지 않는다
- [[부분취소-잔액-정본-수량기준-상태무관-누적]] — 사건 집계는 상태에 딸려 있고, 취소수량은 상태와 무관하다
- [[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]] — 필드 수를 세니 근거가 사라졌다. 원인별 중복 규칙
- [[취소요청키-유일범위-주문단위-와-같은키-다른내용-거부]] — 일관성 근거로 삼기 전에 그 흐름이 왜 그런지 확인한다
- [[취소사건에-발생원인·품목내역-기록-대사는-주문을-안-읽는다]] — 답을 알던 쪽이 안 적어놔서 모르는 쪽이 찾아다닌다
- [[부분취소-동시성-주문행-단일잠금과-캐시겹-미도입]] — 최적화 겹은 "돈이 걸렸으니 더 두자"의 대상이 아니다
- [[잔여전부취소-별도주소-미신설과-전액취소-명명-기각]] — 이미 일부를 취소한 주문에서는 전액이 아니다
- [[취소접수-트랜잭션경계-구현을-결정에-맞춘다-전파속성-잔재]] — 결정을 현실에 맞게 개정하지 않는다
- [[부분취소-외부취소와-우리-기록-사이-대응관계-부재]] `decided` — 1:1이라서 성립하던 안전망이 1:N에서 사라졌다
- [[결제-부분환불-도입-현행한계-4가지와-테이블분리]] `superseded` — 현행 한계 진단(유효)과 테이블 분리(기각)
- [[payment-부분취소-모델만-열고-구현-보류]] `decided` — 모델만 열어두고(amount+SUM) 로직은 전액취소만 구현

### 주문·금액

- [[주문-금액-모델-도출·기록·스냅샷-3분류와-청구액-승인액-분리]] — 저장할지 도출할지는 금액의 성격이 정한다
- [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]] — LEFT JOIN + COALESCE backfill
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]] — Redis SETNX 1차 + RDB 복합 unique 2차 이중방어
- [[주문-멱등-캐시-inflight-차단-전용]] — 결과 캐싱 제거
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]] — @TransactionalEventListener(AFTER_COMMIT)로 Redis 분리
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]] — 주문 이중결제 앞단 진입 차단 + 예약 조회 단일화·미발견 코드 분리
- [[주문만료-spring-batch-chunk-retry-skip]] — Spring Batch(chunk 100·retry/skip 차등·keyset 페이징)
- [[order-version-낙관락-비관락-공존]] — 만료/결제 race를 @Version이 잡는다
- [[order-concurrency-service-학습코드-격리]] — production과 격리한 동시성 학습 코드

### 결제 (2026-05 ~ 06)

- [[payment-order-도메인분리와-pg격리]] — 재설계 동기와 식별자 재배치
- [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]] — 외부 호출은 밖, order·payment 두 aggregate 쓰기는 한 트랜잭션
- [[payment-order-facade-결합끊기-tell-dont-ask]] — Tell-Don't-Ask와 facade 조율
- [[payment-append-only-원장과-exists-완료판단]] — 결제 시도 append-only 원장과 EXISTS 기반 완료 판단
- [[payment-완료여부-사실조회-hascompletedpayment-srp]] — hasCompletedPayment
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]] — MySQL NULL 트릭 partial unique
- [[payment-동시성-unique-vs-lock-gap-lock회피]] — 왜 lock이 아니라 unique 제약인가 (gap lock 회피)
- [[payment-낙관적-락-도입-왜-비관-아님]] — 왜 비관 락이 아닌가
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]] — 흡수를 트랜잭션 밖으로 빼는 3계층 구조
- [[payment-amount-mismatch-이중검증-409-vs-400-분리]] — 재요청 mismatch(409)와 PG 검증(400) 두 검증 분리
- [[payment-unknown-결과불명-처리와-예외분류]] — 세 번째 상태와 예외 분류
- [[payment-status-사실만-분류는-정책계산-manual-review-철회]] — MANUAL_REVIEW 새 상태 철회
- [[payment-attempt-네이밍-정리와-refactor-경계]] — 옛 엔티티 잔재만, 진짜 '시도'는 보존
- [[paymentattempt-상태전이-도메인-검증-defensive]] — 멱등 자기 전이까지 거부
- [[paymentattempt-호출정책-문서-javadoc-archunit-보류]] — ArchUnit 대신 문서·JavaDoc·단일 호출처
- [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]] — 실용진영 유지, 명시적 save 컨벤션
- [[payment-reserve-예약테이블-분리-a안-b안]] `superseded` — 단일 결제 테이블(A안)에서 예약/사건 두 테이블(B안)로
- [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]] `superseded` — RESERVED 예약 행 생성·재사용·만료
- [[예약-동시소비-가드-version-vs-cas]] — @Version vs CAS, 정확성 동등 시 도메인 표현력 우선
- [[간편결제-직접연동-vs-결제대행사-하이브리드-전략-검토]] `open` — 소규모에선 직접 연동이 오히려 비쌀 수 있다

### 취소·환불·보상·대사 (2026-06)

- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]] — 단일 tx 환불 의도 + standalone CANCEL 대사로 보장
- [[보상판단-payment-존재-lock-대신-db-unique]] — 락 대신 DB unique 통일 원칙
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]] — 정책은 payment.application, PG는 cancel 콜백, 의미별 dispatcher 분리
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]] — 완료 우선 + 이중결제 adapter 매핑
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]] — 보상 대상 pgPaymentId 무조건 취소로 전환
- [[requires-new-격리-제거-보상판단-트랜잭션정책]] — 근거 사라진 방어벨트와 잘못 박힌 정책 문구 정정
- [[대사-확정-검증보상-대칭-재승인없음]] — 실시간과 대칭 검증·중복결제 환불 누락 수정
- [[결제-후처리-대상식별-status중심-재설계]] — 후처리 대상 식별을 실패코드 열거에서 status 중심으로 재설계
- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]] — 결과 불명 시 UNKNOWN 보존을 AlreadyComplete 이력 재확인·cancel 경로로 확장
- [[미확정차단-대사스캔-정합성-starvation-escalation]] — over-blocking·starvation은 escalation 종착 한 곳에서 풀린다
- [[대사-keep-waiting-backoff-next-reconcile-at]] `superseded` — status-직교 next_reconcile_at 필드로 재조회를 미룬다
- [[결제-escalation-종착통지-escalatedAt-직교필드]] `superseded` — 새 상태 대신 escalatedAt 직교 필드
- [[pg-승인-예외-경계-요청전송시점]] — "요청 전송 시점"을 경계로 버그 전파와 UNKNOWN 보존을 가른다
- [[sql-translator-빈-제거-제약명-이중결제-식별]] — 이중결제 식별을 제약명 기반으로 전환

### 재고·상품·장바구니

- [[재고차감-동시성-비관락과-productid-정렬]] — 낙관락·Redis락·atomic UPDATE 기각
- [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]] — 사용자취소는 동기, 만료는 Outbox+Kafka 비동기
- [[관리자-재고조작-별도api-이력-감사-분리]] — 관리자 재고 조작 별도 API + 변경만 이력에 남긴다
- [[product-공개query-관리자command-서비스-분리]] — Product 공개 query 서비스와 관리자 command 서비스 분리
- [[product-상세조회-stock-의존-재고누락-0-정규화]] — 상품 상세조회 stock 의존 + 재고 누락 시 stockQuantity=0 정규화
- [[product-mvp-범위-imageurl-카테고리-페이지네이션-제외]] — imageUrl 문자열 저장 + 카테고리·페이지네이션·초기재고 제외
- [[product-soft-delete-deletedat-주문이력-보존]] — 주문 이력 보존이 결정을 강제
- [[productstatus-3상태-공개노출-정책]] — ProductStatus 3상태(ON_SALE/SOLD_OUT/STOPPED)와 공개 노출 정책
- [[cart-도메인-골격-cartitem-단일-aggregate]] — CartItem 단일 entity aggregate
- [[cart-동시성-낙관락-processor-분리-retry]] — 낙관락 + Processor 분리 + retry
- [[cart-회원당-row-상한-미도입]] `open` — 이 phase 미도입
- [[cart-add-product-존재-상태-검증]] — 장바구니 추가 시 Product 존재·상태 검증
- [[cart-delete-미존재-4xx-entity-경유-삭제]] — DELETE 미존재 4xx + entity 경유 삭제
- [[cart-path-id-검증-spec을-코드에-맞춤]] — spec을 코드에 맞춤

### 인증·인가

- [[인증-패키지-경계-auth-member-security-분리]] — auth/member/security 3분할과 의존 방향
- [[security-common-leaf-재편과-토큰-포트-네이밍]] — 인증/인가 진입점을 common.security leaf로 재편 + 토큰 포트/어댑터 네이밍
- [[jwt-redis-하이브리드-rtr-ttl-근거]] — TTL 근거와 한 회원 한 활성 refresh
- [[refreshtokenstore-delete-제거-로그아웃-미구현]] — 로그아웃 미구현의 흔적
- [[auth-redis-장애-도메인예외-매핑-도메인전용-어드바이스]] — Auth Redis 장애를 인프라 도메인 예외 매핑 + 도메인 전용 어드바이스로 통일
- [[회원가입-not-supported-트랜잭션-분리]] — REQUIRED/AFTER_COMMIT/NOT_SUPPORTED 3자 비교

### 예외 처리·에러 코드

- [[도메인-예외-httpstatus-제거-errorcategory]] — HttpStatus 대신 의미 분류(ErrorCategory)
- [[persistence-exception-노출-경계-추상수준]] — 영속성 예외 노출 경계를 예외의 추상 수준으로 다시 긋기
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]] — infra adapter 경계와 flush 타이밍
- [[인프라-일시장애-503-예외-매핑-판단축]] — 예외 상속 전략의 두 직교 판단축
- [[redis-장애-strict-정책-soft-fail-기각]] — soft fail의 함정
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]] — boolean을 도메인 예외로 분리

### 스키마·마이그레이션·컨벤션

- [[flyway-도입-ddl-auto-validate-전환]] — 단일 DB에서도 silent drift는 코드 내부 요인으로 일어난다
- [[cross-aggregate-fk-to-id-참조-컨벤션]] — 신설 도메인 컨벤션
- [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]] — 멀티모듈 선행작업으로서의 동기와 전략
- [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]] — 단일 원칙 대신 사용처별 분석 + 응답 DTO 외부 주입
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]] — 잔류 index·UNIQUE 유지와 InnoDB FK-index 비대칭
- [[multi-column-unique-length-명시-컨벤션]] — enum length 미명시 컨벤션의 좁은 예외
- [[enum-db-check-미사용-application-layer-위임]] — 유효성 보장을 application layer로 위임
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]] — decouple series의 메타원칙과 scope 규율
- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]] — 부가효과로 드러난 정책과 fixture 침투
- [[mdc-정리-스코프-오너십-2규칙]] — memberId 필터 간 릴레이 제거
- [[테스트-태그-2축-모델-ci-잡-분리]] — 테스트 @Tag 2축 모델 + CI 잡 단위슬라이스/통합배치 분리

## contracts

_(없음)_

## topics (11)

- [[payment-도메인-구조-개요]] `draft` — 두 Aggregate와 PG 연동 경계 (2026-06·08 재설계로 스냅샷)
- [[order-도메인-구조-개요]] — 엔티티·상태·서비스 경계 지도
- [[product-도메인-구조-개요]] — 공개조회/관리자관리 분리와 기반 도메인 위치
- [[stock-도메인-구조-개요]] — 차감·복구·이력·동시성 격리의 세 축
- [[auth-member-security-도메인-구조-개요]] — 3패키지 분리와 인증 데이터 흐름
- [[결제사-간편결제-구분과-세-층-역할-결과불명-재시도-모델]] — 결제대행사 vs 간편결제, 세 층, 결과 불명과 재시도 두 종류
- [[네이버페이-환불-api-실측-기록]] — 명세가 말하지 않는 것 6건 (재현 비용이 커서 남긴 기록)
- [[동시성-테스트-작성-규칙과-단언-전략]] — "타이밍을 운에 맡기지 않는다"
- [[서블릿-필터-예외-처리와-에러-디스패치]] — 전역 핸들러 경계와 OncePerRequestFilter
- [[jwt-예외-catch-footgun-bare-securityexception]] — bare SecurityException이 java.lang으로 조용히 바인딩
- [[silent-schema-drift-패턴]] — Hibernate 자동 추론 의존 + 엔티티 의도 미명시

## incidents

_(없음)_

## knowledge (25)

### 설계·검증 방법론

- [[설계단계-검증-정본·선례-확인실패와-전제의-적대적검증]] `draft` — 기존 결정·컨벤션·선례 확인 실패 패턴과 전제의 적대적 검증
- [[명세-자기검토의-한계와-인용한-규약의-실재성-검증]] — 자기가 쓴 문서는 자기가 못 본다. 인용한 규약은 열어 확인한다
- [[명세-반복검사-건수도-심각도도-수렴지표가-아니다]] — 0은 "없다"가 아니라 "이번 축이 못 봤다"일 수 있다
- [[설계문서의-근거-선택-사라질코드·문서밖-사실·층-분리]] — 없어질 것을 새 규칙의 발판으로 쓰지 않는다
- [[하네스-ac-검증범위-격리테스트와-위험경로-결정적강제]] `draft` — 격리 테스트를 AC에 넣고 위험 경로를 결정적으로 강제
- [[하네스-프로세스-무게와-문서-동작-정합]] — 규칙은 기억이 아니라 정본으로 확인
- [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]] — 상반돼 보이면 층위로 통합, blind accept 금지
- [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]] — 발견한 인접 부채는 즉석 패치 말고 처리처로 분리
- [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]] — 미사용 API·방어가드·자동리뷰 판단
- [[문서작성-병렬분담의-경계-구멍과-담지-못한-것-보고]] — 병렬 작업의 실패는 "경계가 빈 것"에서 나온다

### 테스트

- [[도달불가분기-방어금지-불변식테스트-돈정합성-통합테스트]] — 돈 정합성은 통합 테스트로
- [[회귀테스트는-판별력있는-단언과-도달지점을-함께-못박는다]] — "없음"은 여러 이유로 성립한다. 어디까지 갔다가 실패했나를 함께 단언
- [[예외타입-실측과-테스트가-명세를-넘어-단언하는-것]] — 존재와 생성은 다르다. 테스트가 명세를 넘으면 그게 명세가 된다

### 문서·기록 운영

- [[문서-코드-정합성-개념정본-심볼최소화]] — 문서는 개념·정책·왜의 정본, 현재 코드 형상·심볼은 코드가 단일 출처
- [[backend-완료된-task-문서-불변-원칙]] — commerce-backend 문서 운영 원칙
- [[작업중-결정을-영구-adr로-승격하는-단위-개정사슬-접기]] — 승격 단위는 "한 질문에 대한 최종 답"
- [[코드-주석에-문서-내부번호를-박지-않는다-재사용이-더-나쁘다]] — 링크 깨짐보다 번호 재사용이 나쁘다
- [[대규모-기계적-일괄변경-diff-검증의-보장범위와-커밋분할]] — 삭제 줄만 보는 검사는 절반만 증명한다

### 구현 패턴·함정

- [[find-first-write-not-check-db-unique-멱등]] — 사전 find로 정상 멱등 흡수, unique 위반 race만 안전망 500
- [[방어-겹은-정합성겹과-최적화겹으로-가른다]] — 장애 정책은 구현체가 아니라 유스케이스가 정한다
- [[종류·테이블-분리시-조용한-회귀와-전수조사-대상]] — 배치 스캔은 대상이 사라져도 예외가 안 난다
- [[애너테이션-잔재-판별과-전파변경의-osiv-파급]] — 오래 살아남았다고 의도된 것은 아니다
- [[보상-catch-2차예외-평탄화-원칙]] — 예외 안 던지는 의도 캡슐화 메서드
- [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]] — 실무 트레이드오프 정리
- [[ddd-이관-컨벤션-adapter-command-query-네이밍]] — adapter·command/query·의도기반 메서드명·legacy 분리

## features (MOC, 32)

도메인·기능·기법 축의 태그만 MOC로 만든다(5개+). 프로세스·메타 축(`convention` 21·`process` 12·`verification` 8 등)은 `knowledge/`가 담당한다.

- [[payment]] (79) · [[refund]] (30) · [[order]] (27) · [[partial-cancel]] (18)
- [[idempotency]] (32) · [[concurrency]] (26) · [[unique-constraint]] (21) · [[double-payment]] (10)
- [[reconciliation]] (23) · [[escalation]] (12) · [[unknown-status]] (8) · [[compensation]] (15) · [[saga]] (6)
- [[exception-handling]] (21) · [[error-code]] (11)
- [[optimistic-lock]] (14) · [[pessimistic-lock]] (8) · [[transaction-boundary]] (11)
- [[ddd]] (13) · [[aggregate]] (10) · [[adapter]] (10)
- [[naverpay]] (19) · [[pg-gateway]] (15) · [[reservation]] (9)
- [[jpa]] (15) · [[mysql]] (13) · [[redis]] (12) · [[migration]] (8)
- [[stock]] (11) · [[product]] (7) · [[auth]] (7) · [[cart]] (6)
