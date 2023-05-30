# Monitoring Business Performance
Defined and visualized KPIs for different areas of the businees

In this project, the KPIs for different areas of the business were defined and also different dashboards were designed to monitor in real-time the business performance.

The areas that are monitored include marketing, customer, funnel and product.

This project was done using SQL in GCP BigQuery and Microsoft Power BI.

## 1. Marketing SQL Code:

For Marketing KPIs use both of the tables generated from these two codes, joining them on Power BI.

### Step 1.1:
```
WITH temp AS
  (
  SELECT
    DISTINCT user_cookie_id as user_cookie_id
  FROM `Prism_Main.sessions`
  ),
temp2 AS
  (
  SELECT
    user_cookie_id,
    SUM(transaction_revenue) AS total_revenue,
    'Registered' AS registration_status
  FROM `Prism_Main.transactions`
  GROUP BY user_cookie_id
  )
SELECT
  temp.user_cookie_id,
  COALESCE(temp2.total_revenue, 0) AS total_revenue,
  CASE
    WHEN temp2.registration_status IS NULL THEN 'Not Registered'
    ELSE temp2.registration_status
  END AS registration_status
FROM temp
LEFT JOIN temp2
  ON temp.user_cookie_id = temp2.user_cookie_id
```
### Step 1.2:
```
SELECT
  date,
  session_id,
  user_cookie_id,
  CASE
    WHEN traffic_source = 'lm.facebook.com' OR traffic_source = 'l.facebook.com' OR traffic_source = 'facebook.com' OR traffic_source = 'm.facebook.com' THEN 'facebook'
    WHEN traffic_source = 'system_mail' THEN 'email'
    WHEN traffic_source = 'youtube.com' THEN 'youtube'
    WHEN traffic_source = 'instagram.com' OR traffic_source = 'l.instagram.com' THEN 'instagram'
    WHEN traffic_source = 'tiktok.com' THEN 'tiktok'
    WHEN traffic_source = 'twitter.com' THEN 'twitter'
    WHEN traffic_source = '(direct)' THEN 'directly'
    ELSE traffic_source
  END AS traffic_source
  FROM `Prism_Main.sessions`
```

## 2. Customer SQL Code:

### Step 2.1:
Saved the result in a table called prism_test.Indigo-SC-W3-Basis
```
WITH temp2 AS
(
  SELECT date, user_cookie_id, SUM(transaction_revenue) AS total_revenue, COUNT(transaction_revenue) AS no_transactions
  FROM `Prism_Main.transactions`
GROUP BY date, user_cookie_id
),
temp3 AS
(
  SELECT date, user_cookie_id, COUNT(DISTINCT session_id) AS total_visits
  FROM `Prism_Main.funnelevents`
  GROUP BY date, user_cookie_id
),
temp4 AS
(
  SELECT date, user_cookie_id, COUNT(DISTINCT session_id) AS total_purchase
  FROM `Prism_Main.funnelevents`
  WHERE event_name = 'purchase'
  GROUP BY date, user_cookie_id
)
SELECT temp2.date, temp2.user_cookie_id, temp2.total_revenue, temp2.no_transactions, temp3.total_visits, temp4.total_purchase
FROM temp2
LEFT JOIN temp3
  ON temp2.user_cookie_id = temp3.user_cookie_id AND temp2.date = temp3.date
LEFT JOIN temp4
  ON temp3.user_cookie_id = temp4.user_cookie_id AND temp3.date = temp4.date
```


### Step 2.2:
Saved the result in a table called prism_test.Indigo-SC-W3-Seg
```
WITH temp AS
(
  SELECT format_date('%Y', date) AS year, user_cookie_id, SUM(total_revenue) as total_revenue, SUM(no_transactions) AS no_transactions, SUM(total_visits) AS total_visits, SUM(total_purchase) AS total_purchases
FROM `prism_test.Indigo-SC-W3-Basis`
GROUP BY year, user_cookie_id
),
temp2 AS
(
  SELECT user_cookie_id, date, MAX(date) OVER (PARTITION BY user_cookie_id, format_date('%Y', date)) AS last_purchase_of_year
FROM `prism_test.Indigo-SC-W3-Basis`
GROUP BY user_cookie_id, date
),
temp3 AS
(
SELECT user_cookie_id, format_date('%Y', date) AS year, last_purchase_of_year
FROM temp2
GROUP BY year, user_cookie_id, last_purchase_of_year
ORDER BY user_cookie_id
)
SELECT temp.*, temp3. last_purchase_of_year
FROM temp
JOIN temp3
  ON temp.user_cookie_id = temp3.user_cookie_id AND temp.year = temp3.year
ORDER BY user_cookie_id
```



### Step 2.3:
Saved the result in a table called prism_test.Indigo-SC-W3-Final

```
with t1 as (select *,
  sum(no_transactions) over (partition by user_cookie_id order by last_purchase_of_year) as running_transactions,
  sum(total_revenue) over (partition by user_cookie_id order by last_purchase_of_year) as running_revenue,
from `prism_test.Indigo-SC-W3-Seg`),
t2 as (select *,
  NTILE(5) OVER (PARTITION BY year ORDER BY last_purchase_of_year ASC) AS R,
  ntile(100) over (partition by year order by running_transactions) as f_precentile,
  ntile(100) over (partition by year order by running_revenue) as m_precentile
from t1),
## giving f and m scores based on percentile
t3 as (
select *,
  case
    when f_precentile >= 95 then 5
    when f_precentile >= 80 then 4
    when f_precentile >= 70 then 3
    when f_precentile >= 40 then 2
    else 1
  end as F,
    case
    when m_precentile >= 95 then 5
    when m_precentile >= 80 then 4
    when m_precentile >= 70 then 3
    when m_precentile >= 40 then 2
    else 1
  end as M
from t2),
## concatinating the rfm scores to one value and converting it into an integer
t4 as (
SELECT *,
      CAST(CONCAT(R,F,M) AS INT64) AS rfm
FROM t3),
## applying the segments
t5 AS (
select *,
  case
    when rfm in (555, 554, 544, 545, 454, 455, 445) then "champions"
    when rfm in (543, 444, 435, 355, 354, 345, 344, 335)  then "loyal customer"
    when rfm in (553, 551, 552, 541, 542, 533, 532, 531, 452, 451, 442, 441, 431, 453, 433, 432, 423, 353, 352, 351, 342, 341, 333, 323)  then "potential loyalist"
    when rfm in (512, 511, 422, 421, 412, 411, 311)  then "new customer"
    when rfm in (525, 524, 523, 522, 521, 515, 514, 513, 425, 424, 413, 414, 415, 315, 314, 313)  then "promising"
    when rfm in (535, 534, 443, 434, 343, 334, 325, 324)  then "need attention"
    when rfm in (331, 321, 312, 221, 213) then "about to sleep"
    when rfm in (255, 254, 245, 244, 253, 252, 243, 242, 235, 234, 225, 224, 153, 152, 145, 143, 142, 135, 134, 133, 125, 124)  then "at risk"
    when rfm in (155, 154, 144, 214, 215, 115, 114, 113)  then "can't lose them"
    when rfm in (332, 322, 231, 241, 251, 233, 232, 223, 222, 132, 123, 122, 212, 211)  then "hibernating"
    when rfm in (111, 112, 121, 131, 141, 151)  then "lost"
  end as segment
from t4)
SELECT *
FROM t5
ORDER BY user_cookie_id, year
```

## 3. Exporting To Microsoft Power BI
In the final step, all the tables were exported to Microsoft Power BI and four different dashboards were made.
