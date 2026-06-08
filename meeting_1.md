extends: PRISM_SKILL_public.md

# BEENY 브랜드 미팅 자동화 SKILL (meeting.md)

> 오늘의집 브랜드 미팅 자료 자동 생성 스킬 (워크플로우 by 비니/beeny_lee)
> **버전 v2.1 (PRISM 검증 마트 정합판 + 파트너 미팅 어젠다 B12)** | 갱신 2026-06-06
> 📌 **데이터 범위 = 오늘의집 전 브랜드.** 비니 담당 브랜드로 한정하지 않는다. `brand_id`만 넣으면 어느 브랜드든(예: LG전자·한샘 등 타 MD 담당 포함) 추출된다. KAM/MD 필터는 절대 걸지 않는다.
> v1.0 → v2.0 변경점: 손으로 쓴 물리테이블 SQL을 **PRISM 검증 마트(화이트리스트 intent)** 라우팅으로 전면 교체. 컬럼명/필터/단독판정 정정.

---

## 이 파일의 목적

`brand_id` · `brand_name` · `shop_id` **3개만** 입력하면, Claude가 아래 절차대로
PRISM 검증 마트에서 데이터를 조회 → 분석 → **HTML 대시보드 + 미팅 리포트**를 자동 생성한다.

이 파일은 PRISM(`PRISM_SKILL_public.md`)을 **확장(extends)** 한다.
PRISM의 페르소나·SQL 가드·답변 포맷·So What 규칙은 모두 그대로 유지하고,
이 파일은 "브랜드 미팅"이라는 **워크플로우와 산출물 양식**만 추가한다.

---

## 사용법

Claude에게 아래처럼 입력한다 (PRISM 프로젝트 안에서 실행):

```
meeting.md 참고해서 브랜드 미팅 자료 뽑아줘
brand_name: 레미제이
brand_id: 50585
shop_id: 22799044
```

`shop_id`를 모르면 `brand_name`만 줘도 STEP 0에서 자동 확인한다.

---

## ⚠️ 데이터 정합 원칙 (이 스킬의 핵심 · 반드시 준수)

> 이 섹션은 Claude에게 전달되는 규칙이다.

1. **물리 테이블 직접 쿼리 금지 · PRISM 검증 마트만 사용한다.**
   `comm_product_perform`, `commerce_gross_profit`, `comm_plp_def` 등을 직접 짜지 않는다.
   PRISM 화이트리스트의 검증 마트(intent)를 Step 2 SQL 가드 절차로 매칭·실행한다.
   이유: 검증 마트는 BI 대시보드와 산식·단독판정·KAM 계위가 일치하도록 검증되어 있다.
   직접 짠 SQL은 취소액 부호·단독 기준·KAM 컬럼명 등에서 어긋날 위험이 크다.

2. **브랜드 필터 규칙**
   - 기본: `brand_id = {brand_id}` 로 필터한다. (`LIKE '%브랜드명%'` 절대 금지 — 동명 샵 오염)
   - **동명/복수 샵 위험 브랜드**(예: HAY 등)는 `brand_id = {brand_id} AND shop_id = {shop_id}` **동시** 적용.
   - STEP 0에서 항상 정확한 `brand_name`을 먼저 확인한 뒤 본 분석을 진행한다.

3. **KAM/MD 필터 금지 · 전 브랜드 적용** — 이 리포트는 오직 `brand_id`(필요시 `+shop_id`) 단위다.
   `md_kam_latest`·`kam_d1_latest`로 **거르지 않는다**(비니 포트폴리오 한정 아님). 오늘의집 모든 브랜드에 동일하게 동작한다.
   KAM 계위는 커버에 **표시용**으로만 보여줘 맥락을 준다: `kam_d1_latest`(GP/CP/IB), `md_kam_latest`(담당 MD).

4. **시점·요일은 SQL로 직접 조회** (PRISM §1 캘린더 위임 룰). 머릿속 계산 금지.
   YoY = 작년 동월/동기간, MoM = 직전월, MTD/YTD = 당월/당년 base_dt 범위 누적.

5. **이월 마감 주의**: 진행 중인 달(예: 당월 말) 수치는 미마감일 수 있다. 리포트에 주석 표기.

---

## 분석 블록 → PRISM 마트 라우팅 표

| # | 분석 블록 | 사용 PRISM 마트 (intent) | 핵심 컬럼 |
|---|----------|--------------------------|-----------|
| B0 | 브랜드 기본정보 | `prism_comm_sales_raw` | brand_id/name, shop_id/name, kam_d1_latest, md_kam_latest |
| B1 | YoY·MoM·MTD·YTD 실적 | `prism_comm_sales_raw` | base_dt, gmv_net, gmv_payment, gmv_cancel, option_quantity |
| B2 | 카테고리별 분석 | `prism_comm_sales_raw` | admin_cate_d2/d3, gmv_net, qty, selling_cost, review_cnt |
| B3 | 상품별 분석 + 단독 추이 | `prism_comm_sales_raw` + `prism_comm_exclusive_product` | product_id/name, gmv_net, gmv_cancel, is_excv_ssot / excv_type, excv_brand, excv_duration |
| B4 | PLP 지면별 퍼널 | `prism_comm_funnel_stats_byproduct_att` | page_d1_kr, imp_total, pv_total, order_total, gmv_total |
| B5 | PDP 유입·전환 | `prism_comm_funnel_stats_byproduct` | impression_count_join, pdpview_count_join, purchase_count_join |
| B6 | 유입채널(organic/paid/nonpaid) | `prism_comm_funnel_stats_byproduct_att` | imp/pv/order/gmv × _organic/_paid/_nonpaid |
| B7 | 카테고리 평균 가격대 분포 | `prism_comm_sales_raw` | admin_cate_d2, selling_cost, status_kr |
| B8 | 비슷한 가격대 경쟁 브랜드 추이 | `prism_comm_sales_raw` | admin_cate_d2, brand_name, selling_cost, gmv_net (월별) |
| B9 | 추천 키워드 | `prism_comm_product_searchkeywords` | search_keyword, qc, click_qc, order_qc / production_id, user_cnt |
| B10 | 광고 효율(오하우스애즈) | `prism_ads_stats_byproduct` (+ `prism_ads_stats_byproduct_plp_period`) | campaign_objective, spending_ads, d14_payments_ads_direct, click_ads / ads_type |
| B11 | 하반기 주요 액션 | (B1~B10 종합 인사이트) | — |
| **B12 파트너 미팅 어젠다 (4종)** | | | |
| B12-1 | 프로모션 슬롯 성과 | `prism_mkt_sm_market_stats_byproduct` (+ `prism_mkt_sm_exhibition_stats`) | market_d1/d2, deal_id, gmv_net, gross_profit, impression_count_join, pdpview_count_join, orders_total |
| B12-2 | 목표 예실 대비 | `prism_comm_etc_target_mgmt` | base_mth, usage(목표/가마감/마감), data_sort, index_2/3, gmv, gross_profit |
| B12-3 | 재고/품절·취소 | `prism_comm_product_selection` (+ `prism_comm_sales_raw`) | is_onsale, is_outofstock, is_stopped, is_new / status_kr, gmv_cancel |
| B12-4 | 리뷰·시딩 효과 | `prism_comm_sales_raw` (+ `prism_comm_product_reviews`) | review_cnt, review_avg_stars, stylingshot_cnt, scrap_cnt / pos_rate, neg_rate, aspect, keyword |

> 마트 정식 intent명은 PRISM 화이트리스트에 등록되어 있다. Step 2 SQL 가드가 위 마트명을 키로 매칭한다.

---

## STEP 0 — 브랜드 확정 (항상 가장 먼저)

`prism_comm_sales_raw`에서 최근 1개월 범위로 아래를 확인한다.

- 정확한 `brand_name`, `shop_id`/`shop_name`, 복수 샵 여부
- `kam_d1_latest`(GP/CP/IB), `md_kam_latest`(담당 MD)
- 주력 카테고리(`admin_cate_d2`) 상위, 판매중 상품 수(`status_kr = '판매중'`)

복수 shop_id가 잡히면 사용자에게 "합산할지 / 특정 shop만 볼지" 1회 확인.

---

## STEP 1 — 본 분석 (B1 ~ B11)

각 블록은 PRISM Step 2 SQL 가드(intent 매칭 → schema_doc 적용 → 출처 명시)를 거쳐 실행한다.
모든 금액은 **억 단위**, 증감은 **+/- 기호**, 비율은 **%/%p**로 표기(PRISM 답변 포맷 준수).

### B1 — YoY · MoM · MTD · YTD 실적

- 마트: `prism_comm_sales_raw` (필터 `brand_id`, 필요시 `+shop_id`)
- 순거래액 = `SUM(gmv_net)` / 취소율 = `ABS(SUM(gmv_cancel)) / NULLIF(SUM(gmv_payment), 0)` / 평균단가(AOV) = `SUM(gmv_net) / NULLIF(SUM(option_quantity), 0)`
- 기간 grain: `DATE_TRUNC('month', base_dt)` (월별 추이), `DATE_TRUNC('week', base_dt)` (WoW)
- **YoY**: 올해 vs 작년 동월/동기간 / **MoM**: 직전월 / **MTD**: 당월 1일~기준일 / **YTD**: 당년 1/1~기준일 누적
- 표기: `+12.3억 (+4.1%)`, 취소율은 `%p`
- ⚠️ 취소액(`gmv_cancel`)은 음수로 적재됨 → 취소율 계산 시 `ABS()` 또는 분자에 `-` 처리

### B2 — 카테고리별 분석 (다중 카테고리 보유 시)

- 마트: `prism_comm_sales_raw`, `GROUP BY admin_cate_d2` (필요시 `admin_cate_d3`까지 드릴다운)
- 지표: `SUM(gmv_net)`, `SUM(option_quantity)`, `AVG(selling_cost)`, 상품수 `COUNT(DISTINCT product_id)`
- 해석 포인트: 카테고리별 GMV 비중·성장률, 단일 카테고리 의존도(80%+면 리스크), 신규 카테고리 진입 여부

### B3 — 상품별 분석 (+ 단독 상품 추이)

- 마트: `prism_comm_sales_raw` — 상품별 `SUM(gmv_net)` 상위/하위, `SUM(gmv_cancel)`(취소 원인), `review_cnt`, `review_avg_stars`, `selling_cost`, `is_excv_ssot`(단독 여부)
- **잘 팔리는 이유 진단**: B5(PDP 퍼널)·B4(노출 지면)와 결합 — 노출↑ + 조회→구매 CVR↑ + 리뷰 다수면 "검색·노출+신뢰" 구조
- **빠지는 이유 진단**: 노출 감소? 취소율 급등(`gmv_cancel` 비중)? 품절(`status_kr`)? 경쟁 가격(B8)?
- **단독 상품 추이**: `prism_comm_exclusive_product` (SSOT = `exclusive_product_list_v2`, **25.11.3 이후만 신뢰**)
  - `excv_type`(신규<90일/기존), `excv_brand`(연합/세컨), `excv_duration`(운영일수), `is_edition`(에디션)
  - 레이어 split 필요 시 `is_layer`(brand_id=100651)로 포함/제외 동시 제시
  - 단독 비중(vs 전체)은 `prism_comm_sales_raw`의 `is_excv_ssot`로 `SUM(IF(is_excv_ssot=1, gmv_net,0)) / NULLIF(SUM(gmv_net),0)`

### B4 — PLP 지면별 퍼널 (어디서 전환이 나오나)

- 마트: `prism_comm_funnel_stats_byproduct_att` (월 파티션 `yyyymm` 6자리 문자열 필수, 예 `'202605'`)
- `GROUP BY page_d1_kr` (한글 지면명 · 내장 컬럼, `comm_plp_def` 별도 조인 불필요)
- 지표: `SUM(imp_total)`, `SUM(pv_total)`, `SUM(order_total)`, `SUM(gmv_total)`
- 전환율: 조회→구매 CVR = `order_total / NULLIF(pv_total,0)`, 노출→조회 = `pv_total / NULLIF(imp_total,0)`
- ⚠️ attribution 모델이 지표별로 다름: 노출=PLP raw / 조회=referrer / 주문·거래액=last-click. `gmv_total`은 sales_raw의 `gmv_net`과 단위·기준이 달라 직접 합산 비교 금지(별도 SSOT)
- 해석: **노출 多 + CVR 低 지면 = 낭비 구간**(기획전이 흔함), 묶음배송·브랜드페이지가 보통 CVR 상위

### B5 — PDP 유입·전환 (상품별 퍼널)

- 마트: `prism_comm_funnel_stats_byproduct` (grain base_dt × product_id, 회원 이벤트수 기준)
- 지표: `impression_count_join`(노출), `pdpview_count_join`(PDP 조회), `purchase_count_join`(구매, 당일취소 제외)
- 퍼널: 노출→조회 = `pdpview/impression`, 조회→구매 = `purchase/pdpview`
- 상품별로 "노출은 되는데 조회 안 됨(썸네일/가격)" vs "조회는 되는데 구매 안 됨(상세/리뷰/배송)" 구분
- 참고: 유저수(조회자수·구매자수) 기반은 `prism_mkt_funnel_stats`이나 브랜드 필터 미지원 → 브랜드 단위는 본 이벤트수 마트 사용

### B6 — 유입채널 (organic / paid / nonpaid)

- 마트: `prism_comm_funnel_stats_byproduct_att` (organic/paid/nonpaid가 컬럼으로 **내장**)
- `SUM(gmv_organic)`, `SUM(gmv_paid)`, `SUM(gmv_nonpaid)` → 채널 비중
- 채널별 CVR = `order_paid / NULLIF(pv_paid,0)` 등으로 organic vs paid 전환 효율 비교
- 해석: organic 비중 70%+면 건강한 자연 수요 / paid CVR이 organic의 50% 미만이면 유료 효율 점검

### B7 — 카테고리 평균 가격대 분포

- 마트: `prism_comm_sales_raw`, 브랜드 주력 `admin_cate_d2`로 필터 (`status_kr='판매중'`, `selling_cost>0`)
- 가격 밴드(`selling_cost`): `<1만 / 1~2만 / 2~3만 / 3~5만 / 5~10만 / 10만+`
- 카테고리 전체 상품수 대비 **이 브랜드 상품의 가격 밴드별 분포** → 가격 포지셔닝 파악

### B8 — 비슷한 가격대 경쟁 브랜드 추이

- 마트: `prism_comm_sales_raw`, 동일 `admin_cate_d2` + 브랜드 주력가 `selling_cost ± 30%` 밴드
- 해당 밴드 GMV 상위 브랜드(`brand_name`) Top 5~7 추출 → 월별 `SUM(gmv_net)` 시계열
- 해석: 우리 브랜드의 카테고리 내 순위·점유 추이, 급성장 경쟁사 vs 정체 브랜드

### B9 — 추천 키워드

- 마트: `prism_comm_product_searchkeywords`
  - Query B(키워드 × 상품)로 **이 브랜드 product_id를 클릭으로 끌어온 키워드** 역추적 → 현재 작동 키워드
  - Query A(키워드 퍼널 `qc → click_qc → order_qc`)로 검색량 大 + 주문전환 高 키워드 식별 → 기회 키워드
- 상품명(`product_name`)에 없는데 검색 수요 있는 키워드 = 상품명 최적화 후보
- 시즌 키워드: 이사철(3~5월) / 추석(8~9월) / 연말(11~12월). 패브릭 예: `냉감 이불`, `극세사 이불`, `여름 침구`

### B10 — 광고 효율 (오하우스애즈)

- 마트: `prism_ads_stats_byproduct` (원천 `ads_kr.ads_report`)
- 집행비 = `SUM(spending_ads)` / 노출 `impression_ads` / 클릭 `click_ads` / CTR = `click_ads/impression_ads`
- **상품 광고 ROAS(기본)** = `SUM(d14_payments_ads_direct) / NULLIF(SUM(spending_ads),0)` · 필터 `campaign_objective='PRODUCT'` 필수
- 광고주 전체 효율(예외) = `(direct + indirect) / spending`
- `ad_platform`: OHOUSEADS(현재)/MOLOCO(레거시) 통합 SUM, 둘 비교 무의미
- 온사이트 광고 인벤토리(주간/월간 추이)는 `prism_ads_stats_byproduct_plp_period`의 `ads_type`(ad_production/ad_stylingshot/ad_banner_production vs NULL=오가닉)로 보강
- 해석 기준(상품 ROAS): 양호 ≥ 3.0 / 점검 < 1.0 (브랜드·카테고리 평균과 함께 제시)

### B11 — 하반기 주요 액션 (인사이트 기반)

B1~B10 결과를 종합해 즉시 / 7월 / 8월로 구분한 실행안을 도출한다.
각 액션은 근거 블록 번호를 괄호로 단다. 예: "취소율 높은 SKU 상세 보강 (B3·B1)".
PRISM 규칙대로 "그래서 무엇을(So What) + 다음 단계" 반드시 포함.

---

## STEP 2 — 파트너 미팅 어젠다 (B12 · 미팅 자리에서 합의할 레이어)

> B1~B11이 "현황 진단(과거~현재)"이라면, B12는 미팅에서 파트너와 **합의·협상·계획**할 레이어다.
> 파트너 미팅 자료를 뽑을 때 아래 4종을 함께 산출한다(요청 시).

### B12-1 — 프로모션 슬롯 성과 (어느 구좌가 잘 됐나)

- 마트: `prism_mkt_sm_market_stats_byproduct` (매장 × 모음전 × 상품 grain, 필터 `brand_id`)
- `GROUP BY market_d1`(오늘의딜/바이너리샵/라이브/빅프로모션 등) 또는 `market_d2`(초스페셜/일반, 집요한세일 주차 등)
- 지표: `SUM(gmv_net)`, `SUM(gross_profit)`, `SUM(impression_count_join)`(노출), `SUM(pdpview_count_join)`(조회), `SUM(orders_total)`(주문), CVR = `orders_total / NULLIF(pdpview_count_join,0)`
- 기획전 자체 퍼널(방문→조회→구매)은 `prism_mkt_sm_exhibition_stats` (`page_type`별, 비율컬럼 SUM 금지 → UV 재집계)
- ⚠️ 일 약 17K행 평탄화 → **반드시 `base_dt` 범위 좁히고 `brand_id` 필터**. 14일 초과는 주차 분할
- ⚠️ 같은 상품이 여러 매장/모음전에 노출되면 행 중복 → 상품 유니크는 `COUNT(DISTINCT product_id)`
- 해석/미팅 포인트: 구좌별 GMV·CVR 비교 → "다음 슬롯은 효율 좋은 OO 구좌로, 비효율 구좌는 SKU 교체" 합의. 집요한세일 만원특가/반값딜 후보 SKU 선정 근거

### B12-2 — 목표 예실 대비

- 마트: `prism_comm_etc_target_mgmt` (원천 `target_mgmt_cost_sales`)
- ⚠️ **자동 활성화 금지** — 사용자가 "목표/예실/마감" 명시 요청 시에만 (PRISM §5)
- `data_sort = '01_거래액/매출총이익액'`, 헤더행(`data_sort <> 'data_sort'`) 제외
- **closing status fallback**: `③ 마감 > ② 가마감 > ① 목표` 순으로 best available 값 사용(`usage` 컬럼)
- ⚠️ 값 컬럼(`gmv`, `gross_profit` 등)이 **STRING** → `CAST(REPLACE(gmv, ',', '') AS BIGINT)` 후 연산
- ⚠️ **grain 한계**: 이 목표 마트는 **카테고리·비용항목 단위**라 단일 브랜드 목표값은 직접 없음. 브랜드 실적을 **소속 카테고리(admin_cate_d1) 목표 라인 대비**로 위치시키는 맥락 용도로 사용. 브랜드 개별 목표가 필요하면 별도 시트/합의 라인으로 보완
- 미팅 포인트: 카테고리 목표 진척률 안에서 이 브랜드의 기여·갭 → 반기 commit 재확인

### B12-3 — 재고/품절·취소 (운영 신뢰도)

- 마트: `prism_comm_product_selection` (일 × 상품 상태 플래그, 필터 `brand_id`)
- 상태 플래그: `is_onsale`(판매중), `is_outofstock`(일시품절), `is_stopped`(판매정지), `is_new`(신규 승인), `is_exposed/is_viewed/is_sold`(노출·조회·구매 발생)
- 품절 규모: `COUNT(DISTINCT IF(is_outofstock=1, product_id, NULL))` / 신규 등록 추이: `SUM(is_new)`
- 취소: `prism_comm_sales_raw`의 `gmv_cancel`(음수 적재 → `ABS`) → 상품별 취소율 = `ABS(SUM(gmv_cancel)) / NULLIF(SUM(gmv_payment),0)`, `status_kr`로 품절·정지 동반 확인
- 미팅 포인트: 품절 SKU 재고 보충 요청, 취소율 높은 SKU 원인(배송 지연·상세 불일치 등) 공동 점검. "기회손실 = 노출 있는데 품절"인 SKU 우선순위화

### B12-4 — 리뷰·시딩 효과 (전환 자산)

- 마트: `prism_comm_sales_raw` (스냅샷 반응 지표, 필터 `brand_id`)
  - `review_cnt`(리뷰수), `review_avg_stars`(별점), `stylingshot_cnt`(스타일링샷 = 시딩/콘텐츠 자산), `scrap_cnt`(스크랩 = 관심도)
  - 시계열(`DATE_TRUNC('week'/'month', base_dt)`)로 리뷰·스타일링샷·스크랩 증가 추이 → 시딩/리뷰 이벤트 효과 대리 지표
- 리뷰 품질 심화: `prism_comm_product_reviews` (감성 `pos_rate`/`neg_rate`, `aspect`·`keyword`·대표표현)
  - ⚠️ **상품 단위 전용 · product_id 최대 10개** → 주력/단독 SKU 중심으로만 호출
- 미팅 포인트: 리뷰 적은 주력 SKU에 리뷰 이벤트·시딩 집중, 부정 키워드(`neg`/aspect) 상세페이지 보강, 스타일링샷 부족 SKU 콘텐츠 제작 제안
- ⚠️ "시딩"은 별도 전용 테이블이 없어, 위 지표(스타일링샷·스크랩·리뷰 속도)로 **간접 측정**한다. 광고/콘텐츠 기여 GMV가 필요하면 PRISM 광고·콘텐츠 intent로 별도 보강

---

## 산출물

### 산출물 1 — HTML 대시보드 (`{브랜드명}_브랜드미팅_{YYYYMMDD}.html`)

- 슬라이드 9~12개: 커버(KPI 카드) → YoY/MoM → 거래액 추이(막대+취소율) → PLP 퍼널(B4) → 유입채널 도넛(B6) → 카테고리+상품(B2·B3) → 경쟁 브랜드(B8) → 단독 추이(B3) → 개선 진단 → 액션(B11)
- 외부 의존성 최소(Chart.js CDN 1개 또는 순수 Canvas), 로컬에서 바로 열림, "HTML 저장" 버튼, ←/→ 키보드 이동
- 컬러: 다크 네이비 `#0F2044` + 블루 `#1B4FBE` + 그린 `#00C896`

### 산출물 2 — 미팅 리포트 (`{브랜드명}_미팅리포트_{YYYYMMDD}.md`)

요청 항목 11개를 순서대로 포함: 카테고리별 / 상품별(+단독) / PLP 퍼널 / YoY·MoM(MTD·YTD) / PDP 유입·전환 / 가격대 분포 / 경쟁 브랜드 / 유입채널 / 추천 키워드 / 광고 효율 / 하반기 액션.
+ 파트너 미팅용이면 **B12 어젠다 4종**(프로모션 슬롯 성과 / 목표 예실 대비 / 재고·품절·취소 / 리뷰·시딩 효과)을 "미팅 어젠다" 챕터로 덧붙인다.

### 산출물 3 — 미팅 대본 (선택 · 요청 시 docx)

슬라이드별 스크립트 + 파트너에게 던질 질문 + 오늘 합의할 액션 테이블.

---

## 인사이트 판정 기준 (비니 MD 기준)

**✅ 긍정**: YoY 순거래액 +30%↑ · organic 비중 70%↑ · 취소율 10% 미만 · 묶음배송 CVR 3%↑ · 주력 SKU 리뷰 50건↑
**⚠️ 주의**: 취소율 15~20% · paid CVR < organic CVR의 50% · 기획전 노출 多 + CVR 0.2% 미만 · 신상품 3개월 공백
**🚨 위험**: 노출 YoY -50%↓ · 취소율 20%↑ · 단일 상품 GMV 80%↑ 집중 · 기획전 CVR 0.1% 미만(낭비)

---

## 주의사항 (정합성 체크)

1. **shop_id 복수 확인** — 합산/분리 여부 STEP 0에서 확정
2. **취소액 부호** — `gmv_cancel` 음수 적재 → 취소율은 `ABS()` 처리
3. **att 거래액 ≠ sales 거래액** — `byproduct_att.gmv_total`(last-click)과 `sales_raw.gmv_net`은 기준이 달라 직접 합산 비교 금지
4. **단독 SSOT** — `is_excv_ssot`(=exclusive_product_list_v2), 25.11.3 이전은 0. legacy `is_exclusive`와 다를 수 있음
5. **att 월 파티션** — `yyyymm` 6자리 문자열(`'202605'`)
6. **당월 미마감** — 진행 중인 달 수치는 잠정치 주석
7. **출처 1줄 명시** — PRISM Step 2 규칙대로 사용 마트 친화명+풀네임 답변 말미 표기

---

*meeting.md | 오늘의집 비니(beeny_lee) 브랜드 미팅 자동화 | v2.1 | 2026-06-06 | extends PRISM_SKILL_public.md*
