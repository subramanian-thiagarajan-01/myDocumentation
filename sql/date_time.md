# 18 Date & Time Functions

### Date/Time Types and Functions

```sql
-- Current date/time functions
SELECT
    CURRENT_DATE AS today,              -- Just date
    CURRENT_TIME AS time,               -- Time with TZ
    CURRENT_TIMESTAMP AS now,           -- Date + time + TZ
    NOW() AS now_alias,                 -- Alias for CURRENT_TIMESTAMP
    LOCALTIME AS local_time,            -- No TZ
    LOCALTIMESTAMP AS local_timestamp;  -- No TZ

-- Extracting components
SELECT
    EXTRACT(YEAR FROM TIMESTAMP '2025-11-09 14:30:00') AS year,      -- 2025
    EXTRACT(MONTH FROM NOW()) AS month,                               -- 11
    EXTRACT(DAY FROM NOW()) AS day,                                   -- 9
    EXTRACT(HOUR FROM NOW()) AS hour,
    EXTRACT(MINUTE FROM NOW()) AS minute,
    EXTRACT(SECOND FROM NOW()) AS second,
    EXTRACT(DOW FROM NOW()) AS day_of_week,                          -- 0=Sunday
    EXTRACT(QUARTER FROM NOW()) AS quarter;                          -- 1-4

-- DATE_PART alternative syntax
SELECT
    DATE_PART('year', hire_date) AS year,
    DATE_PART('month', hire_date) AS month,
    DATE_PART('day', hire_date) AS day;

-- Date truncation
SELECT
    DATE_TRUNC('year', NOW()) AS year_start,      -- 2025-01-01 00:00:00
    DATE_TRUNC('month', NOW()) AS month_start,    -- 2025-11-01 00:00:00
    DATE_TRUNC('day', NOW()) AS day_start,        -- 2025-11-09 00:00:00
    DATE_TRUNC('hour', NOW()) AS hour_start;      -- 2025-11-09 14:00:00

-- Age calculation
SELECT
    AGE(NOW(), hire_date) AS time_employed,
    AGE(hire_date) AS age_from_epoch;  -- From '1900-01-01'

-- Date arithmetic with INTERVAL
SELECT
    NOW() + INTERVAL '1 day' AS tomorrow,
    NOW() - INTERVAL '1 month' AS last_month,
    NOW() + INTERVAL '2 hours 30 minutes' AS in_2_5_hours,
    NOW() + (salary::INTEGER || ' days')::INTERVAL AS future_date;

-- INTERVAL types
SELECT
    INTERVAL '1 day' AS one_day,
    INTERVAL '1 week' AS one_week,
    INTERVAL '1 month' AS one_month,
    INTERVAL '1 year' AS one_year,
    INTERVAL '1 day 02:03:04' AS complex_interval;

-- Convert string to timestamp
SELECT
    TO_TIMESTAMP('2025-11-09', 'YYYY-MM-DD') AS date_converted,
    TO_TIMESTAMP('2025-11-09 14:30:00', 'YYYY-MM-DD HH24:MI:SS') AS full_timestamp,
    TO_DATE('09/11/2025', 'DD/MM/YYYY') AS european_date;

-- Format timestamp to string
SELECT
    TO_CHAR(NOW(), 'YYYY-MM-DD') AS date_format,
    TO_CHAR(NOW(), 'Day, Month DD, YYYY') AS long_format,  -- Saturday, November 09, 2025
    TO_CHAR(NOW(), 'MM/DD/YY HH24:MI:SS') AS time_format;

-- Date comparison
SELECT * FROM events
WHERE event_date >= CURRENT_DATE - INTERVAL '30 days'
  AND event_date < CURRENT_DATE + INTERVAL '1 day';

-- Business day calculation
SELECT
    hire_date,
    hire_date + (working_days || ' days')::INTERVAL AS end_date
FROM employees;

-- Time zone handling
SELECT
    NOW() AT TIME ZONE 'America/New_York' AS ny_time,
    NOW() AT TIME ZONE 'Europe/London' AS london_time,
    NOW() AT TIME ZONE 'Asia/Tokyo' AS tokyo_time;

-- Interview trick: Date edge cases
SELECT
    DATE_TRUNC('month', NOW())::DATE + INTERVAL '1 month' - INTERVAL '1 day' AS last_day_of_month,
    EXTRACT(DAYS FROM INTERVAL '1 month') AS month_length,  -- NULL (variable)
    EXTRACT(DAYS FROM (DATE_TRUNC('month', NOW()) + INTERVAL '1 month' - DATE_TRUNC('month', NOW()))) AS days_in_month;
```

### Common Date/Time Patterns

```sql
-- Last day of month
SELECT DATE_TRUNC('month', NOW())::DATE + INTERVAL '1 month' - INTERVAL '1 day';

-- First day of month
SELECT DATE_TRUNC('month', NOW())::DATE;

-- Last day of quarter
SELECT DATE_TRUNC('quarter', NOW())::DATE + INTERVAL '3 months' - INTERVAL '1 day';

-- Next Monday
SELECT
    DATE_TRUNC('week', NOW() + INTERVAL '1 day')::DATE + INTERVAL '1 day' AS next_monday;

-- Number of days between dates
SELECT DATE_PART('day', end_date - start_date) AS days_difference;

-- Business days (excluding weekends)
SELECT
    COUNT(*) FILTER (WHERE EXTRACT(DOW FROM date) NOT IN (0, 6)) AS business_days
FROM (
    SELECT DATE_TRUNC('day', generate_series(start_date, end_date, '1 day'))::DATE AS date
) t;

-- Month-over-month comparison
SELECT
    EXTRACT(MONTH FROM order_date) AS month,
    COUNT(*) AS orders,
    LAG(COUNT(*)) OVER (ORDER BY EXTRACT(MONTH FROM order_date)) AS prev_month_orders
FROM orders
GROUP BY EXTRACT(MONTH FROM order_date);

-- Year-to-date totals
SELECT
    SUM(order_amount) FILTER (WHERE order_date >= DATE_TRUNC('year', NOW())) AS ytd_sales,
    SUM(order_amount) FILTER (WHERE order_date < DATE_TRUNC('year', NOW())) AS prior_year_sales
FROM orders;
```
