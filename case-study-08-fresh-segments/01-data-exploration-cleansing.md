# Data Exploration and Cleansing

[← Back to Fresh Segments overview](./README.md)

## Objective

Prepare the Fresh Segments data for analysis by:

- Converting `month_year` to a valid date
- Investigating missing records
- Checking interest identifiers
- Selecting an appropriate table join
- Validating metric dates against interest creation dates

---

## 1. Convert `month_year` to a date

The original `month_year` values use the format:

```text
07-2018
```

The column should contain the first day of each month as a PostgreSQL `DATE`.

### Query

```sql
ALTER TABLE fresh_segments.interest_metrics
ALTER COLUMN month_year TYPE DATE
USING TO_DATE(
    month_year,
    'MM-YYYY'
);
```

### Validate the conversion

```sql
SELECT
    month_year,
    pg_typeof(month_year) AS data_type
FROM fresh_segments.interest_metrics
WHERE month_year IS NOT NULL
LIMIT 10;
```

---

## 2. Count records for each month

### Query

```sql
SELECT
    month_year,
    COUNT(*) AS record_count
FROM fresh_segments.interest_metrics
GROUP BY month_year
ORDER BY month_year NULLS FIRST;
```

### Result

| Month | Records |
|---|---:|
| NULL | 1,194 |
| 2018-07-01 | 729 |
| 2018-08-01 | 767 |
| 2018-09-01 | 780 |
| 2018-10-01 | 857 |
| 2018-11-01 | 928 |
| 2018-12-01 | 995 |
| 2019-01-01 | 973 |
| 2019-02-01 | 1,121 |
| 2019-03-01 | 1,136 |
| 2019-04-01 | 1,099 |
| 2019-05-01 | 857 |
| 2019-06-01 | 824 |
| 2019-07-01 | 864 |
| 2019-08-01 | 1,149 |

### Insight

The monthly number of recorded interests varies considerably.

The dataset contains 1,194 records that cannot be assigned to a reporting month.

---

## 3. Investigate and handle missing values

### Calculate the missing-value percentage

```sql
SELECT
    COUNT(*) FILTER (
        WHERE month_year IS NULL
           OR interest_id IS NULL
    ) AS incomplete_records,

    COUNT(*) AS total_records,

    ROUND(
        100.0
        * COUNT(*) FILTER (
            WHERE month_year IS NULL
               OR interest_id IS NULL
        )
        / COUNT(*),
        2
    ) AS incomplete_percentage

FROM fresh_segments.interest_metrics;
```

### Result

| Incomplete records | Percentage |
|---:|---:|
| 1,194 | 8.36% |

### Decision

The incomplete rows should not be used in the analysis because their metrics cannot be connected to:

- A reporting month
- A specific interest
- A record in the interest mapping table

The original data should remain unchanged. A separate cleaned table is created instead of deleting records permanently.

### Create the cleaned table

```sql
DROP TABLE IF EXISTS fresh_segments.interest_metrics_clean;

CREATE TABLE fresh_segments.interest_metrics_clean AS

SELECT *
FROM fresh_segments.interest_metrics
WHERE month_year IS NOT NULL
  AND interest_id IS NOT NULL;
```

### Validate the cleaned table

```sql
SELECT
    COUNT(*) FILTER (
        WHERE month_year IS NULL
           OR interest_id IS NULL
    ) AS incomplete_records
FROM fresh_segments.interest_metrics_clean;
```

### Expected result

```text
0
```

---

## 4. Compare identifiers between the two tables

### Interests in metrics but not in the map

```sql
SELECT
    COUNT(DISTINCT m.interest_id) AS metrics_without_map
FROM fresh_segments.interest_metrics_clean AS m
LEFT JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id
WHERE im.id IS NULL;
```

### Interests in the map but not in metrics

```sql
SELECT
    COUNT(DISTINCT im.id) AS map_without_metrics
FROM fresh_segments.interest_map AS im
LEFT JOIN fresh_segments.interest_metrics_clean AS m
    ON im.id = m.interest_id
WHERE m.interest_id IS NULL;
```

### Result

| Comparison | Count |
|---|---:|
| Metric interests without a map record | 0 |
| Mapped interests without metric records | 7 |

### Additional summary

```sql
SELECT
    (
        SELECT COUNT(DISTINCT interest_id)
        FROM fresh_segments.interest_metrics_clean
    ) AS metric_interest_count,

    (
        SELECT COUNT(DISTINCT id)
        FROM fresh_segments.interest_map
    ) AS mapped_interest_count;
```

| Metric interests | Mapped interests |
|---:|---:|
| 1,202 | 1,209 |

### Insight

Every interest used in the metrics table has valid mapping information.

Seven mapped interests have no recorded customer interactions.

---

## 5. Check the uniqueness of `interest_map.id`

### Query

```sql
WITH id_counts AS (
    SELECT
        id,
        COUNT(*) AS id_frequency
    FROM fresh_segments.interest_map
    GROUP BY id
)

SELECT
    id_frequency,
    COUNT(*) AS id_count
FROM id_counts
GROUP BY id_frequency
ORDER BY id_frequency;
```

### Result

| ID frequency | Number of IDs |
|---:|---:|
| 1 | 1,209 |

### Insight

Every `interest_map.id` is unique, so the column can safely act as the mapping table's primary key.

---

## 6. Select the correct table join

All interest identifiers in the cleaned metrics table exist in the mapping table.

An `INNER JOIN` is therefore appropriate when analysing metric records.

### Query for interest `21246`

```sql
SELECT
    m.*,
    im.interest_name,
    im.interest_summary,
    im.created_at,
    im.last_modified

FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

WHERE m.interest_id = 21246

ORDER BY m.month_year;
```

### Result summary

The query returns 10 monthly records for:

```text
Readers of El Salvadoran Content
```

### Insight

An `INNER JOIN` preserves every usable metric record while adding the interest name, description and metadata.

---

## 7. Validate metric dates against creation dates

### Records where the monthly date precedes the exact creation date

```sql
SELECT
    COUNT(*) AS records_before_creation_timestamp
FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

WHERE m.month_year < im.created_at::DATE;
```

### Result

| Records |
|---:|
| 188 |

The result occurs because `month_year` is stored as the first day of the month, while `created_at` contains an exact timestamp.

### Check whether any record predates the creation month

```sql
SELECT
    COUNT(*) AS records_before_creation_month
FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

WHERE m.month_year
      < DATE_TRUNC(
            'month',
            im.created_at
        )::DATE;
```

### Result

| Records |
|---:|
| 0 |

### Insight

The 188 records are valid because the interest and its metric record belong to the same calendar month.

---

## Cleaning Summary

The cleaned analytical dataset provides:

- Valid monthly dates
- No incomplete metric records
- Valid interest mappings
- Unique mapping identifiers
- Confirmed creation-date consistency
- A reproducible alternative to deleting source data
