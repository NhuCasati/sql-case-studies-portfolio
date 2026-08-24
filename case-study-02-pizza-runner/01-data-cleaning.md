# Data Cleaning and Transformation

[← Back to Pizza Runner overview](./README.md)

## Overview

The original Pizza Runner dataset contains inconsistent null values, text inside numeric columns and measurements with mixed formats.

The data must be cleaned before completing the analysis.

---

## 1. Cleaning `customer_orders`

### Data-quality issues

The `exclusions` and `extras` columns contain:

- SQL `NULL` values
- Text values containing `'null'`
- Empty values
- Comma-separated topping identifiers

### Cleaning approach

- Standardise missing values.
- Preserve valid topping identifiers.
- Create a cleaned temporary table for analysis.

### Query

```sql
CREATE TEMP TABLE customer_orders_clean AS
SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
        WHEN exclusions IS NULL
             OR exclusions = 'null'
             OR TRIM(exclusions) = ''
            THEN NULL
        ELSE exclusions
    END AS exclusions,
    CASE
        WHEN extras IS NULL
             OR extras = 'null'
             OR TRIM(extras) = ''
            THEN NULL
        ELSE extras
    END AS extras,
    order_time
FROM pizza_runner.customer_orders;
```

---

## 2. Cleaning `runner_orders`

### Data-quality issues

The `runner_orders` table contains:

- Text values containing `'null'`
- Distances containing `km`
- Durations containing `minutes`, `minute` or `mins`
- Blank cancellation values
- Numeric and timestamp values stored as text

### Cleaning approach

- Convert invalid text values to SQL `NULL`.
- Remove units from distance and duration.
- Convert cleaned values to appropriate data types.

### Query

```sql
CREATE TEMP TABLE runner_orders_clean AS
SELECT
    order_id,
    runner_id,
    CASE
        WHEN pickup_time = 'null'
            THEN NULL
        ELSE pickup_time::TIMESTAMP
    END AS pickup_time,
    CASE
        WHEN distance = 'null'
            THEN NULL
        ELSE REGEXP_REPLACE(distance, '[^0-9.]', '', 'g')::NUMERIC
    END AS distance_km,
    CASE
        WHEN duration = 'null'
            THEN NULL
        ELSE REGEXP_REPLACE(duration, '[^0-9.]', '', 'g')::INTEGER
    END AS duration_minutes,
    CASE
        WHEN cancellation IS NULL
             OR cancellation = 'null'
             OR TRIM(cancellation) = ''
            THEN NULL
        ELSE cancellation
    END AS cancellation
FROM pizza_runner.runner_orders;
```

---

## Validation

After cleaning, verify the transformed tables:

```sql
SELECT *
FROM customer_orders_clean;

SELECT *
FROM runner_orders_clean;
```

## Cleaning Summary

The cleaned tables provide:

- Consistent missing values
- Numeric distance values
- Numeric duration values
- Valid timestamps
- Standardised cancellation information
