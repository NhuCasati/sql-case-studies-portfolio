# Data Cleansing

[← Back to Data Mart overview](./README.md)

## Objective

Create a new table named `data_mart.clean_weekly_sales` with:

- A valid date column
- Week, month and year fields
- Standardised segment values
- Age-band and demographic fields
- Average transaction values

---

## Transformation Rules

### Age bands

| Segment number | Age band |
|---:|---|
| 1 | Young Adults |
| 2 | Middle Aged |
| 3 or 4 | Retirees |

### Demographics

| Segment letter | Demographic |
|---|---|
| C | Couples |
| F | Families |

Missing or text-based `null` values are replaced with `unknown`.

---

## Query

```sql
DROP TABLE IF EXISTS data_mart.clean_weekly_sales;

CREATE TABLE data_mart.clean_weekly_sales AS

WITH cleaned_source AS (
    SELECT
        TO_DATE(
            week_date,
            'DD/MM/YY'
        ) AS parsed_week_date,
        region,
        platform,
        CASE
            WHEN segment IS NULL
              OR LOWER(TRIM(segment)) = 'null'
                THEN 'unknown'
            ELSE TRIM(segment)
        END AS cleaned_segment,
        customer_type,
        transactions,
        sales
    FROM data_mart.weekly_sales
)

SELECT
    parsed_week_date AS week_date,

    EXTRACT(
        WEEK FROM parsed_week_date
    )::INTEGER AS week_number,

    EXTRACT(
        MONTH FROM parsed_week_date
    )::INTEGER AS month_number,

    EXTRACT(
        YEAR FROM parsed_week_date
    )::INTEGER AS calendar_year,

    region,
    platform,
    cleaned_segment AS segment,

    CASE
        WHEN RIGHT(cleaned_segment, 1) = '1'
            THEN 'Young Adults'
        WHEN RIGHT(cleaned_segment, 1) = '2'
            THEN 'Middle Aged'
        WHEN RIGHT(cleaned_segment, 1) IN ('3', '4')
            THEN 'Retirees'
        ELSE 'unknown'
    END AS age_band,

    CASE
        WHEN LEFT(cleaned_segment, 1) = 'C'
            THEN 'Couples'
        WHEN LEFT(cleaned_segment, 1) = 'F'
            THEN 'Families'
        ELSE 'unknown'
    END AS demographic,

    customer_type,
    transactions,
    sales,

    ROUND(
        sales::NUMERIC
        / NULLIF(transactions, 0),
        2
    ) AS avg_transaction

FROM cleaned_source;
```

---

## Validation

### Preview the cleaned table

```sql
SELECT *
FROM data_mart.clean_weekly_sales
LIMIT 20;
```

### Check the column types

```sql
SELECT
    column_name,
    data_type
FROM information_schema.columns
WHERE table_schema = 'data_mart'
  AND table_name = 'clean_weekly_sales'
ORDER BY ordinal_position;
```

### Check missing values

```sql
SELECT
    COUNT(*) FILTER (
        WHERE segment = 'unknown'
    ) AS unknown_segments,

    COUNT(*) FILTER (
        WHERE age_band = 'unknown'
    ) AS unknown_age_bands,

    COUNT(*) FILTER (
        WHERE demographic = 'unknown'
    ) AS unknown_demographics

FROM data_mart.clean_weekly_sales;
```

### Check the years

```sql
SELECT DISTINCT
    calendar_year
FROM data_mart.clean_weekly_sales
ORDER BY calendar_year;
```

### Expected years

| Calendar year |
|---:|
| 2018 |
| 2019 |
| 2020 |

---

## Cleaning Summary

The cleaned table provides:

- Consistent date values
- New calendar attributes
- Standardised missing values
- Derived age and demographic groups
- Row-level average transaction values
