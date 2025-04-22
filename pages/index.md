---
title: Comed Energy Pricing Dashboard
---

<!-- <Details title='How to edit this page'>

  This page can be found in your project at `/pages/index.md`. Make a change to the markdown file and save it to see the change take effect in your browser.
</Details> -->

```sql current_hour_price 
WITH latest_price AS (
  SELECT
    STRFTIME('%Y-%m-%d %-I:%M %p', (CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) - INTERVAL '5 hours')) AS last_price_update,
    price
  FROM comed_pricing.comed_pricing
  WHERE millisUTC IS NOT NULL
    AND millisUTC > 1000000000000
  ORDER BY millisUTC DESC
  LIMIT 1
),
avg_last_1_hour AS (
  SELECT
    ROUND(AVG(price), 3) AS avg_price_last_1h
  FROM comed_pricing.comed_pricing24
  WHERE millisUTC >= CAST(EPOCH(CAST(NOW() AS TIMESTAMP)) * 1000 AS BIGINT) - (1 * 60 * 60 * 1000)  -- last 1 hour
)
SELECT
  latest_price.last_price_update,
  latest_price.price,
  avg_last_1_hour.avg_price_last_1h
FROM latest_price
CROSS JOIN avg_last_1_hour

```
<BigValue 
  data={current_hour_price} 
  value=last_price_update
/>
<BigValue 
  data={current_hour_price} 
  value=price
/>


<!-- <BigValue 
  data={orders_with_comparisons} 
  value=num_orders
  comparison=order_growth
  comparisonFmt=pct1
  comparisonTitle="MoM"
/> -->
```sql last_24_hours
WITH base AS (
  SELECT
    (CAST(TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000) AS TIMESTAMP) - INTERVAL '5 hours') AS ts_local,
    price,
    AVG(price) OVER (
      ORDER BY TO_TIMESTAMP(CAST(millisUTC AS BIGINT) / 1000)
      ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
    ) AS rolling_average_price
  FROM comed_pricing.comed_pricing24
  WHERE millisUTC IS NOT NULL
    AND millisUTC >= CAST(EPOCH(CAST(NOW() AS TIMESTAMP)) * 1000 AS BIGINT) - 86400000
)
SELECT
  ts_local,
  price,
  rolling_average_price,
  STRFTIME('%Y-%m-%d %-I:%M %p', ts_local) AS full_timestamp,
  STRFTIME('%-I:%M %p', ts_local) AS hour_labels
FROM base
ORDER BY ts_local ASC

```
<LineChart  
  data={last_24_hours}
  x=full_timestamp
  y={['price', 'rolling_average_price']} 
  yAxisTitle="Price per timestamp"
/>

<!-- 
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
``` -->

<!-- <Heatmap 
    data={dailyWeeklyPrices} 
    x=hour_of_day
    y=day_of_week 
    value=avg_price 
    valueFmt=usd 
/> -->

<!-- <CalendarHeatmap 
    data={dailyWeeklyPrices}
    date=hour_of_day
    value=avg_price
    title="Calendar Heatmap"
    subtitle="Daily Sales"
/> -->