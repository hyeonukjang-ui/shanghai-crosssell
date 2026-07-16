# 크로스셀 분석 쿼리 세트 (도시 교체 재사용용)

> 상하이로 분석한 쿼리를 **다른 도시에도 그대로** 쓸 수 있게 정리했습니다.
> 각 쿼리 맨 위 `DECLARE` 두 줄(**도시명 · 항공 도착 공항코드**)만 바꾸면 됩니다.
> 데이터소스: Redash `mrtdata` (BigQuery). 기간은 `INTERVAL 12 MONTH` 등에서 조정.

## 0. 공통 설정 — 도시별 파라미터

| 도시 | `city` (T&A·숙박 CITY_NM) | `arr_airports` (항공 도착 공항코드) |
|---|---|---|
| 상하이 | `Shanghai` | `'PVG','SHA'` |
| 베이징 | `Beijing` | `'PEK','PKX'` |
| 칭다오 | `Qingdao` | `'TAO'` |
| 시안 | `Xian` | `'XIY'` |
| 다롄 | `Dalian` | `'DLC'` |
| 하얼빈 | `Harbin` | `'HRB'` |
| 청두 | `Chengdu` | `'CTU','TFU'` |
| 항저우 | `Hangzhou` | `'HGH'` |
| 선전 | `Shenzhen` | `'SZX'` |
| 광저우 | `Guangzhou` | `'CAN'` |
| 옌지 | `Yanji` | `'YNJ'` |
| 난징 | `Nanjing` | `'NKG'` |
| 샤먼 | `Xiamen` | `'XMN'` |
| 충칭 | `Chongqing` | `'CKG'` |
| 홍콩 | `Hong Kong` | `'HKG'` (홍콩은 `ARRIVE_COUNTRY_CD='HK'`) |

**핵심 테이블**
- `edw_fpna.MART_FPNA_NONAIR_PROFIT_D` — T&A·숙박 실 CM/GMV. `FPNA_DOMAIN_NM` = `TNA`(투어티켓) / `LODGMENT`(숙박). `CITY_NM`은 영문.
- `edw_mart.MART_AIR_SALE_D` — 항공 발권. 도착 `ARRIVE_AIRPORT_CD`, 발권일 `ISSUE_KST_DATE`, 출발일 `DEPART_KST_DT`.
- `business.TNA_dashboard_category_product_list_v1` — 대시보드 카테고리(`dashboard_category`), 조인키 `PRODUCT_ID`.

**크로스셀 정의**: 동일 `USER_ID`가 항공 발권/숙박 예약 후 **30일 이내** 해당 도시 T&A 구매. (로그인 매칭 구매만 집계)

---

## 0-1. 도시 설정값 찾기 (내 도시 코드 확인)

```sql
-- (A) T&A 도시명(CITY_NM) 확인 — 영문 표기 확인용
SELECT COUNTRY_NM, CITY_NM, COUNT(*) orders
FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE FPNA_DOMAIN_NM='TNA'
  AND BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
GROUP BY 1,2 ORDER BY orders DESC;

-- (B) 항공 도착 공항코드 확인 — 내 도시 도착 공항 찾기 (예: 중국 CN)
SELECT ARRIVE_CITY_CD, ARRIVE_AIRPORT_CD, COUNT(*) tickets
FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
WHERE ARRIVE_COUNTRY_CD='CN'                       -- 홍콩='HK', 대만='TW', 마카오='MO'
  AND ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
  AND COALESCE(VOID_FLAG,'N')!='Y'
GROUP BY 1,2 ORDER BY tickets DESC;
```

---

## 1. 크로스셀 요약 — 항공 → T&A (전체 기간)

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH flt AS (  -- 해당 도시행 항공 발권 유저 (발권일 이벤트)
  SELECT DISTINCT USER_ID, ISSUE_KST_DATE AS flt_date
  FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND ARRIVE_AIRPORT_CD IN UNNEST(arr_airports)
    AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
),
tna AS (  -- 해당 도시 T&A 주문 (1행=주문라인)
  SELECT USER_ID, CREATE_KST_DATE AS tna_date, RESVE_ID, GMV, CM
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
    AND FPNA_DOMAIN_NM='TNA' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
)
SELECT
  (SELECT COUNT(DISTINCT USER_ID) FROM flt) AS flight_cohort_users,  -- 분모: 항공 구매 유저
  COUNT(DISTINCT t.USER_ID)  AS xsell_users,      -- 30일내 T&A 산 유저
  COUNT(DISTINCT t.RESVE_ID) AS xsell_orders,     -- 크로스셀 T&A 주문 수
  ROUND(SUM(t.GMV))          AS xsell_gmv,
  ROUND(SUM(t.CM))           AS xsell_cm
FROM tna t
WHERE EXISTS (
  SELECT 1 FROM flt f
  WHERE f.USER_ID=t.USER_ID
    AND t.tna_date BETWEEN f.flt_date AND DATE_ADD(f.flt_date, INTERVAL 30 DAY)
);
-- 크로스셀률 = xsell_users / flight_cohort_users
```

## 2. 크로스셀 요약 — 숙박 → T&A (전체 기간)

```sql
DECLARE city STRING DEFAULT 'Shanghai';

WITH stay AS (  -- 해당 도시 숙박 예약 유저 (예약일 이벤트)
  SELECT DISTINCT USER_ID, CREATE_KST_DATE AS stay_date
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND FPNA_DOMAIN_NM='LODGMENT' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
),
tna AS (
  SELECT USER_ID, CREATE_KST_DATE AS tna_date, RESVE_ID, GMV, CM
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
    AND FPNA_DOMAIN_NM='TNA' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
)
SELECT
  (SELECT COUNT(DISTINCT USER_ID) FROM stay) AS stay_cohort_users,
  COUNT(DISTINCT t.USER_ID)  AS xsell_users,
  COUNT(DISTINCT t.RESVE_ID) AS xsell_orders,
  ROUND(SUM(t.GMV))          AS xsell_gmv,
  ROUND(SUM(t.CM))           AS xsell_cm
FROM tna t
WHERE EXISTS (
  SELECT 1 FROM stay s
  WHERE s.USER_ID=t.USER_ID
    AND t.tna_date BETWEEN s.stay_date AND DATE_ADD(s.stay_date, INTERVAL 30 DAY)
);
```

## 3. 월별 크로스셀 (발권월 귀속: 코호트·율·GMV·CM) — 항공

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH flt AS (
  SELECT USER_ID, ISSUE_KST_DATE AS flt_date
  FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND ARRIVE_AIRPORT_CD IN UNNEST(arr_airports)
    AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
),
cohort AS (SELECT FORMAT_DATE('%Y-%m', flt_date) ym, COUNT(DISTINCT USER_ID) cohort_users FROM flt GROUP BY 1),
tna AS (
  SELECT USER_ID, CREATE_KST_DATE AS tna_date, RESVE_ID, GMV, CM
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
    AND FPNA_DOMAIN_NM='TNA' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
),
matched AS (  -- 각 T&A 주문을 30일내 가장 이른 발권월에 귀속
  SELECT t.RESVE_ID, t.USER_ID, t.GMV, t.CM,
    (SELECT MIN(f.flt_date) FROM flt f WHERE f.USER_ID=t.USER_ID
       AND t.tna_date BETWEEN f.flt_date AND DATE_ADD(f.flt_date, INTERVAL 30 DAY)) AS anchor
  FROM tna t
),
agg AS (
  SELECT FORMAT_DATE('%Y-%m', anchor) ym, COUNT(DISTINCT USER_ID) xs_users,
         COUNT(DISTINCT RESVE_ID) xs_orders, ROUND(SUM(GMV)) gmv, ROUND(SUM(CM)) cm
  FROM matched WHERE anchor IS NOT NULL GROUP BY 1
)
SELECT c.ym, c.cohort_users, a.xs_users,
  ROUND(a.xs_users/c.cohort_users*100,1) xs_rate_pct,
  a.xs_orders, a.gmv, a.cm, ROUND(a.cm/NULLIF(a.gmv,0)*100,2) cm_pct
FROM cohort c LEFT JOIN agg a USING(ym) ORDER BY c.ym;
-- 숙박 버전: flt→staycoh(FPNA_DOMAIN_NM='LODGMENT', CREATE_KST_DATE), arr_airports 제거
```

## 4. 구매 타이밍 + 데일리 원본 (발권→구매→여행) — 대시보드 카테고리 조인

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH dc AS (  -- 대시보드 카테고리 매핑
  SELECT PRODUCT_ID, ANY_VALUE(dashboard_category) dcat
  FROM `mrtdata`.business.TNA_dashboard_category_product_list_v1 GROUP BY PRODUCT_ID
),
flt AS (
  SELECT USER_ID, ISSUE_KST_DATE evt, DATE(DEPART_KST_DT) trv
  FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
    AND ARRIVE_AIRPORT_CD IN UNNEST(arr_airports)
    AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
),
tna AS (
  SELECT RESVE_ID, ANY_VALUE(USER_ID) USER_ID, ANY_VALUE(CREATE_KST_DATE) tna_date,
    ANY_VALUE(TRAVEL_START_DATE) tstart, ANY_VALUE(PRODUCT_ID) pid,
    ANY_VALUE(PRODUCT_TITLE) title, ROUND(SUM(GMV)) gmv, ROUND(SUM(CM)) cm
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='TNA' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
    AND CREATE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
  GROUP BY RESVE_ID
),
m AS (  -- 각 주문을 30일내 가장 이른 발권에 귀속 (발권일·출발일 획득)
  SELECT t.*, COALESCE(dc.dcat,'기타') dcat, f.evt, f.trv,
    ROW_NUMBER() OVER(PARTITION BY t.RESVE_ID ORDER BY f.evt) rn
  FROM tna t LEFT JOIN dc ON dc.PRODUCT_ID=t.pid
  JOIN flt f ON f.USER_ID=t.USER_ID
    AND t.tna_date BETWEEN f.evt AND DATE_ADD(f.evt, INTERVAL 30 DAY)
)
SELECT dcat, title, tna_date AS 구매일, evt AS 발권일, trv AS 여행일,
  DATE_DIFF(tna_date, evt, DAY)  AS 발권to구매,   -- (음수 없음)
  DATE_DIFF(trv, tna_date, DAY)  AS 구매to여행,   -- 음수=현지 구매
  gmv, cm
FROM m WHERE rn=1
ORDER BY tna_date DESC;
-- 카테고리 요약: GROUP BY dcat, AVG(발권to구매), AVG(구매to여행), SUM(cm)
-- 숙박 버전: flt→staycoh(LODGMENT, evt=CREATE_KST_DATE, trv=TRAVEL_START_DATE)
```

## 5. 월별 최다 크로스셀 상품/카테고리 — 항공

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH dc AS (SELECT PRODUCT_ID, ANY_VALUE(dashboard_category) dcat FROM `mrtdata`.business.TNA_dashboard_category_product_list_v1 GROUP BY PRODUCT_ID),
flt AS (
  SELECT USER_ID, ISSUE_KST_DATE flt_date FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
    AND ARRIVE_AIRPORT_CD IN UNNEST(arr_airports) AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
),
tna AS (
  SELECT RESVE_ID, ANY_VALUE(USER_ID) USER_ID, ANY_VALUE(CREATE_KST_DATE) tna_date,
    ANY_VALUE(PRODUCT_ID) pid, ANY_VALUE(PRODUCT_TITLE) title
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='TNA' AND CITY_NM=city AND USER_ID IS NOT NULL AND USER_ID!=''
    AND CREATE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
  GROUP BY RESVE_ID
),
m AS (
  SELECT FORMAT_DATE('%Y-%m',t.tna_date) ym, COALESCE(dc.dcat,'기타') dcat, t.title, t.RESVE_ID,
    (SELECT MIN(f.flt_date) FROM flt f WHERE f.USER_ID=t.USER_ID
       AND t.tna_date BETWEEN f.flt_date AND DATE_ADD(f.flt_date, INTERVAL 30 DAY)) anchor
  FROM tna t LEFT JOIN dc ON dc.PRODUCT_ID=t.pid
),
r AS (
  SELECT ym, dcat, title, COUNT(*) orders,
    ROW_NUMBER() OVER(PARTITION BY ym ORDER BY COUNT(*) DESC) rk
  FROM m WHERE anchor IS NOT NULL GROUP BY 1,2,3
)
SELECT ym, rk, dcat, title, orders FROM r WHERE rk<=5 ORDER BY ym, rk;
```

## 6. 역방향 — T&A를 미끼로 항공/숙박 유도

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH tna AS (  -- T&A 구매자 + 첫 구매일
  SELECT USER_ID, MIN(CREATE_KST_DATE) f_tna
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='TNA' AND CITY_NM=city
    AND BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND USER_ID IS NOT NULL AND USER_ID!='' GROUP BY USER_ID
),
flt AS (
  SELECT USER_ID, ISSUE_KST_DATE fdt, SALE_TOTAL_PRICE amt
  FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ARRIVE_AIRPORT_CD IN UNNEST(arr_airports) AND USER_ID IS NOT NULL AND USER_ID!=''
    AND COALESCE(VOID_FLAG,'N')!='Y' AND ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH)
),
fltu AS (SELECT USER_ID, MIN(fdt) f_flt FROM flt GROUP BY USER_ID),
bait AS (  -- T&A 먼저 → 30일내 항공 발권 + 그 항공 규모
  SELECT COUNT(DISTINCT t.USER_ID) users, COUNT(*) tickets, ROUND(SUM(f.amt)) gmv
  FROM tna t JOIN flt f ON f.USER_ID=t.USER_ID
    AND f.fdt > t.f_tna AND f.fdt <= DATE_ADD(t.f_tna, INTERVAL 30 DAY)
)
SELECT
  (SELECT COUNT(*) FROM tna) tna_buyers,
  COUNTIF(fu.USER_ID IS NOT NULL) also_flight,        -- T&A구매자 중 항공도 구매
  COUNTIF(fu.f_flt <  t.f_tna) flight_first,           -- 항공 먼저(기존 크로스셀)
  COUNTIF(fu.f_flt >= t.f_tna) tna_first,              -- T&A 먼저(미끼 후보)
  (SELECT users FROM bait) tna_then_flight_30d_users,
  (SELECT tickets FROM bait) bait_tickets,
  (SELECT gmv FROM bait) bait_flight_gmv
FROM tna t LEFT JOIN fltu fu USING(USER_ID);
-- 숙박 버전: flt→stay(LODGMENT, sdt=CREATE_KST_DATE, GMV/CM 사용), 항공 CM은 이 마트에 없음(GMV만)
```

## 7. 미끼로 강한 T&A 카테고리 (첫 T&A → 30일내 항공/숙박)

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH dc AS (SELECT PRODUCT_ID, ANY_VALUE(dashboard_category) dcat FROM `mrtdata`.business.TNA_dashboard_category_product_list_v1 GROUP BY PRODUCT_ID),
ftna AS (
  SELECT USER_ID, CREATE_KST_DATE f_tna, PRODUCT_ID pid,
    ROW_NUMBER() OVER(PARTITION BY USER_ID ORDER BY CREATE_KST_DATE) rn
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='TNA' AND CITY_NM=city
    AND BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND USER_ID IS NOT NULL AND USER_ID!=''
),
first_tna AS (SELECT ft.USER_ID, ft.f_tna, COALESCE(dc.dcat,'기타') dcat FROM ftna ft LEFT JOIN dc ON dc.PRODUCT_ID=ft.pid WHERE ft.rn=1),
flt AS (SELECT USER_ID, ISSUE_KST_DATE fdt FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ARRIVE_AIRPORT_CD IN UNNEST(arr_airports) AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
    AND ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 13 MONTH))
SELECT ft.dcat, COUNT(DISTINCT ft.USER_ID) bait_users
FROM first_tna ft JOIN flt f ON f.USER_ID=ft.USER_ID
  AND f.fdt > ft.f_tna AND f.fdt <= DATE_ADD(ft.f_tna, INTERVAL 30 DAY)
GROUP BY 1 ORDER BY bait_users DESC;
-- 숙박 버전: flt→stay(LODGMENT, sdt=CREATE_KST_DATE)
```

## 8. 세그먼트 — 항공/숙박 구매 조합별 T&A 다건 구매

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH flt AS (SELECT DISTINCT USER_ID FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ARRIVE_AIRPORT_CD IN UNNEST(arr_airports) AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
    AND ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)),
stay AS (SELECT DISTINCT USER_ID FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='LODGMENT' AND CITY_NM=city
    AND BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH) AND USER_ID IS NOT NULL AND USER_ID!=''),
tna AS (SELECT USER_ID, COUNT(DISTINCT RESVE_ID) tna_cnt, ROUND(SUM(CM)) cm
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
  WHERE FPNA_DOMAIN_NM='TNA' AND CITY_NM=city
    AND BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH) AND USER_ID IS NOT NULL AND USER_ID!='' GROUP BY USER_ID),
seg AS (  -- 유저별 세그먼트 태깅
  SELECT COALESCE(f.USER_ID,s.USER_ID,tn.USER_ID) uid,
    (f.USER_ID IS NOT NULL) has_air, (s.USER_ID IS NOT NULL) has_stay, IFNULL(tn.tna_cnt,0) tcnt, IFNULL(tn.cm,0) cm
  FROM flt f FULL JOIN stay s USING(USER_ID) FULL JOIN tna tn USING(USER_ID)
)
SELECT
  CASE WHEN has_air AND has_stay THEN '항공+숙박'
       WHEN has_air THEN '항공만' WHEN has_stay THEN '숙박만' ELSE 'T&A만' END seg,
  COUNT(*) users,
  COUNTIF(tcnt>=1) with_tna, COUNTIF(tcnt=1) tna_1,
  COUNTIF(tcnt=2) tna_2, COUNTIF(tcnt>=3) tna_3plus,
  COUNTIF(tcnt>=2) tna_2plus, ROUND(SUM(cm)) tna_cm
FROM seg GROUP BY 1 ORDER BY users DESC;
```

## 9. 장바구니 — 함께 사는 T&A 카테고리 조합 (항공 크로스셀러)

```sql
DECLARE city STRING DEFAULT 'Shanghai';
DECLARE arr_airports ARRAY<STRING> DEFAULT ['PVG','SHA'];

WITH dc AS (SELECT PRODUCT_ID, ANY_VALUE(dashboard_category) dcat FROM `mrtdata`.business.TNA_dashboard_category_product_list_v1 GROUP BY PRODUCT_ID),
flt AS (SELECT DISTINCT USER_ID FROM `mrtdata`.edw_mart.MART_AIR_SALE_D
  WHERE ARRIVE_AIRPORT_CD IN UNNEST(arr_airports) AND USER_ID IS NOT NULL AND USER_ID!='' AND COALESCE(VOID_FLAG,'N')!='Y'
    AND ISSUE_KST_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)),
uc AS (  -- 항공 구매자 × 구매한 T&A 카테고리(중복 제거)
  SELECT DISTINCT n.USER_ID, COALESCE(dc.dcat,'기타') cat
  FROM `mrtdata`.edw_fpna.MART_FPNA_NONAIR_PROFIT_D n
  JOIN flt f ON f.USER_ID=n.USER_ID
  LEFT JOIN dc ON dc.PRODUCT_ID=n.PRODUCT_ID
  WHERE n.FPNA_DOMAIN_NM='TNA' AND n.CITY_NM=city
    AND n.BASIS_DATE >= DATE_SUB(CURRENT_DATE('Asia/Seoul'), INTERVAL 12 MONTH)
    AND n.USER_ID IS NOT NULL AND n.USER_ID!=''
),
multi AS (SELECT USER_ID FROM uc GROUP BY USER_ID HAVING COUNT(*)>=2)  -- 2개 카테고리+ 구매자
SELECT a.cat AS cat_a, b.cat AS cat_b, COUNT(*) AS users
FROM uc a JOIN uc b ON a.USER_ID=b.USER_ID AND a.cat < b.cat
WHERE a.USER_ID IN (SELECT USER_ID FROM multi)
GROUP BY 1,2 ORDER BY users DESC LIMIT 20;
-- 숙박 버전: flt→stay(LODGMENT)
```

---

## 참고 / 주의
- **CM은 순액**: `MART_FPNA_NONAIR_PROFIT_D.CM`은 취소·환불 반영 실 공헌이익. 디즈니 등 마이너스 CM 상품 존재.
- **항공 CM 미포함**: 항공 마진은 이 마트에 없어 역방향은 항공 GMV(`SALE_TOTAL_PRICE`)만 집계. 항공 실 CM 필요 시 항공 수익 마트 별도 조인.
- **로그인 매칭만**: `USER_ID` 매칭되는 구매만 잡혀 실제 크로스셀은 더 높을 수 있음.
- **BASIS_DATE 여유**: T&A는 발권/예약보다 뒤에 발생하므로 `INTERVAL 13 MONTH`로 넉넉히 로드.
- **DECLARE 미지원 환경**: `city`/`arr_airports` 를 쿼리 내 `'Shanghai'`, `IN ('PVG','SHA')`로 직접 치환해서 사용.
