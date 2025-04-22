---
title: Comed Energy Pricing Dashboard
---

<!-- <Details title='How to edit this page'>

  This page can be found in your project at `/pages/index.md`. Make a change to the markdown file and save it to see the change take effect in your browser.
</Details> -->

```sql cheapest
WITH with_ts AS (
  SELECT 
    CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) AS ts,
    price
  FROM comed_pricing.comed_prices
  WHERE millisUTC IS NOT NULL 
    AND millisUTC > 1000000000000
),

last_30_days AS (
  SELECT *
  FROM with_ts
  WHERE ts >= CAST(NOW() AS TIMESTAMP) - INTERVAL 30 DAY
),

avg_by_hour_last_30_days AS (
  SELECT 
    EXTRACT(HOUR FROM ts) AS hour_of_day,
    ROUND(AVG(price), 2) AS avg_price
  FROM last_30_days
  GROUP BY hour_of_day
),

ranked_hours AS (
  SELECT *,
    RANK() OVER (ORDER BY avg_price ASC) AS price_rank
  FROM avg_by_hour_last_30_days
),

cheapest_hour AS (
  SELECT hour_of_day
  FROM ranked_hours
  WHERE price_rank = 1
),

latest_day AS (
  SELECT CAST(MAX(CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP)) AS DATE) AS day
  FROM comed_pricing.comed_prices
),

today_prices AS (
  SELECT 
    CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) AS ts,
    price,
    EXTRACT(HOUR FROM CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP)) AS hour_of_day
  FROM comed_pricing.comed_prices, latest_day
  WHERE CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) >= day
    AND CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) < day + INTERVAL 1 DAY
)

SELECT 
  hour_of_day,
  ROUND(AVG(price), 2) AS avg_price_today
FROM today_prices
WHERE hour_of_day IN (SELECT hour_of_day FROM cheapest_hour)
GROUP BY hour_of_day


```
<BigValue 
  data={cheapest} 
  value=hour_of_day
/>
<BigValue 
  data={cheapest} 
  value=avg_price_today
/>

```sql prices
SELECT 
  TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS ts,
  price
FROM comed_pricing.comed_prices
WHERE millisUTC IS NOT NULL
  AND millisUTC > 1000000000000  -- ~2001-09-09, safe threshold
ORDER BY ts ASC

```

<LineChart 
  data={prices}
  x=ts
  y=price 
  yAxisTitle="Price per timestamp"
/>

# Heatmap of daily/weekly average prices
```sql dailyWeeklyPrices
SELECT
  CASE EXTRACT(DOW FROM TIMESTAMP 'epoch' + (CAST(millisUTC / 1000 AS BIGINT)) * INTERVAL '1 second')
    WHEN 0 THEN 'Sunday'
    WHEN 1 THEN 'Monday'
    WHEN 2 THEN 'Tuesday'
    WHEN 3 THEN 'Wednesday'
    WHEN 4 THEN 'Thursday'
    WHEN 5 THEN 'Friday'
    WHEN 6 THEN 'Saturday'
  END AS day_of_week,
  EXTRACT(HOUR FROM TIMESTAMP 'epoch' + (CAST(millisUTC / 1000 AS BIGINT)) * INTERVAL '1 second') AS hour_of_day,
  AVG(price) AS avg_price
FROM comed_pricing.comed_prices
GROUP BY day_of_week, hour_of_day
ORDER BY 
  CASE day_of_week
    WHEN 'Sunday' THEN 0
    WHEN 'Monday' THEN 1
    WHEN 'Tuesday' THEN 2
    WHEN 'Wednesday' THEN 3
    WHEN 'Thursday' THEN 4
    WHEN 'Friday' THEN 5
    WHEN 'Saturday' THEN 6
  END,
  hour_of_day;
```

<!-- <Heatmap 
    data={dailyWeeklyPrices} 
    x=hour_of_day
    y=day_of_week 
    value=avg_price 
    valueFmt=usd 
/> -->

<CalendarHeatmap 
    data={dailyWeeklyPrices}
    date=hour_of_day
    value=avg_price
    title="Calendar Heatmap"
    subtitle="Daily Sales"
/>