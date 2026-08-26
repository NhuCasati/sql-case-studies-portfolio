# Interest Analysis

[← Back to Fresh Segments overview](./README.md)

> Run the queries in `01-data-exploration-cleansing.md` before this section.

## Objective

Determine:

- Which interests appear consistently
- How frequently interests occur
- An appropriate minimum number of months for analysis
- The effect of excluding short-lived interests

---

## 1. Which interests appeared in every month?

### Count the reporting months

```sql
SELECT
    COUNT(DISTINCT month_year) AS reporting_months
FROM fresh_segments.interest_metrics_clean;
```

### Result

| Reporting months |
|---:|
| 14 |

### Find interests appearing in every month

```sql
WITH reporting_period AS (
    SELECT
        COUNT(DISTINCT month_year) AS total_months
    FROM fresh_segments.interest_metrics_clean
)

SELECT
    m.interest_id,
    im.interest_name,
    COUNT(DISTINCT m.month_year) AS months_present

FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

CROSS JOIN reporting_period AS rp

GROUP BY
    m.interest_id,
    im.interest_name,
    rp.total_months

HAVING COUNT(DISTINCT m.month_year)
       = rp.total_months

ORDER BY m.interest_id;
```

### Count the consistent interests

```sql
WITH interest_months AS (
    SELECT
        interest_id,
        COUNT(DISTINCT month_year) AS total_months
    FROM fresh_segments.interest_metrics_clean
    GROUP BY interest_id
)

SELECT
    COUNT(*) AS interests_in_every_month
FROM interest_months
WHERE total_months = 14;
```

### Result

| Interests appearing in all months |
|---:|
| 480 |

### Insight

Approximately 40% of the 1,202 measured interests appeared throughout the complete 14-month reporting period.

---

## 2. At which `total_months` value does the cumulative percentage pass 90%?

### Query

```sql
WITH interest_months AS (
    SELECT
        interest_id,
        COUNT(DISTINCT month_year) AS total_months
    FROM fresh_segments.interest_metrics_clean
    GROUP BY interest_id
),

month_distribution AS (
    SELECT
        total_months,
        COUNT(*) AS interest_count
    FROM interest_months
    GROUP BY total_months
)

SELECT
    total_months,
    interest_count,

    ROUND(
        100.0
        * SUM(interest_count) OVER (
            ORDER BY total_months DESC
            ROWS BETWEEN UNBOUNDED PRECEDING
                     AND CURRENT ROW
        )
        / SUM(interest_count) OVER (),
        2
    ) AS cumulative_percentage

FROM month_distribution
ORDER BY total_months DESC;
```

### Result

| Months present | Interests | Cumulative percentage |
|---:|---:|---:|
| 14 | 480 | 39.93% |
| 13 | 82 | 46.76% |
| 12 | 65 | 52.16% |
| 11 | 94 | 59.98% |
| 10 | 86 | 67.14% |
| 9 | 95 | 75.04% |
| 8 | 67 | 80.62% |
| 7 | 90 | 88.10% |
| 6 | 33 | 90.85% |
| 5 | 38 | 94.01% |
| 4 | 32 | 96.67% |
| 3 | 15 | 97.92% |
| 2 | 12 | 98.92% |
| 1 | 13 | 100.00% |

### Insight

The cumulative percentage first exceeds 90% at six months.

Six months is therefore used as the minimum coverage threshold for the filtered analysis.

---

## 3. How many data points would be removed?

### Query

```sql
WITH interest_months AS (
    SELECT
        interest_id,
        COUNT(DISTINCT month_year) AS total_months
    FROM fresh_segments.interest_metrics_clean
    GROUP BY interest_id
),

excluded_interests AS (
    SELECT
        interest_id
    FROM interest_months
    WHERE total_months < 6
)

SELECT
    COUNT(*) AS removed_records,
    COUNT(DISTINCT m.interest_id) AS removed_interests

FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN excluded_interests AS e
    ON m.interest_id = e.interest_id;
```

### Result

| Removed records | Removed interests |
|---:|---:|
| 400 | 110 |

### Insight

Filtering out interests appearing in fewer than six months removes:

- 110 unique interests
- 400 monthly metric records

---

## 4. Does removing these interests make business sense?

Removing interests with limited history improves the consistency of comparisons because every retained interest has at least six months of data.

However, short-lived interests should not be permanently deleted.

They may represent:

- Newly created customer interests
- Seasonal events
- Emerging trends
- Short marketing campaigns
- Declining interests
- Data-collection problems

For example, an interest appearing for all 14 months supports stable trend analysis.

An interest appearing in only one month may still represent a meaningful event or newly emerging customer behaviour, but it cannot support reliable long-term comparisons.

### Recommended approach

- Retain all source records.
- Use interests with six or more months for stable trend analysis.
- Analyse short-lived interests separately.
- Investigate whether missing months are expected or caused by data-quality issues.

---

## 5. Create the filtered analytical dataset

```sql
DROP TABLE IF EXISTS
    fresh_segments.interest_metrics_filtered;

CREATE TABLE
    fresh_segments.interest_metrics_filtered AS

WITH interest_months AS (
    SELECT
        interest_id,
        COUNT(DISTINCT month_year) AS total_months
    FROM fresh_segments.interest_metrics_clean
    GROUP BY interest_id
)

SELECT
    m.*
FROM fresh_segments.interest_metrics_clean AS m

INNER JOIN interest_months AS im
    ON m.interest_id = im.interest_id

WHERE im.total_months >= 6;
```

### Validate the filtered dataset

```sql
SELECT
    COUNT(*) AS total_records,
    COUNT(DISTINCT interest_id) AS unique_interests
FROM fresh_segments.interest_metrics_filtered;
```

### Result

| Records | Unique interests |
|---:|---:|
| 12,680 | 1,092 |

---

## Unique interests by month

```sql
SELECT
    month_year,
    COUNT(DISTINCT interest_id) AS unique_interests

FROM fresh_segments.interest_metrics_filtered

GROUP BY month_year
ORDER BY month_year;
```

### Result

| Month | Unique interests |
|---|---:|
| 2018-07-01 | 709 |
| 2018-08-01 | 752 |
| 2018-09-01 | 774 |
| 2018-10-01 | 853 |
| 2018-11-01 | 925 |
| 2018-12-01 | 986 |
| 2019-01-01 | 966 |
| 2019-02-01 | 1,072 |
| 2019-03-01 | 1,078 |
| 2019-04-01 | 1,035 |
| 2019-05-01 | 827 |
| 2019-06-01 | 804 |
| 2019-07-01 | 836 |
| 2019-08-01 | 1,062 |

### Insight

Interest coverage increased substantially between July 2018 and March 2019, declined from April through June and recovered by August 2019.
