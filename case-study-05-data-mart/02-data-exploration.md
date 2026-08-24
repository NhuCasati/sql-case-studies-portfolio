# Data Exploration

[← Back to Data Mart overview](./README.md)

> Run the data-cleansing query before running this analysis.

---

## 1. What day of the week is used for each `week_date`?

### Query

```sql
SELECT DISTINCT
    TO_CHAR(
        week_date,
        'FMDay'
    ) AS weekday
FROM data_mart.clean_weekly_sales;
```

### Result

| Weekday |
|---|
| Monday |

### Insight

Every sales week begins on a Monday.

---

## 2. Which week numbers are missing?

### Query

```sql
SELECT
    GENERATE_SERIES(
        1,
        52
    ) AS week_number

EXCEPT

SELECT DISTINCT
    week_number
FROM data_mart.clean_weekly_sales

ORDER BY week_number;
```

### Result

The dataset is missing:

```text
Weeks 1–12 and weeks 37–52
```

A total of 28 week numbers are absent.

### Insight

The dataset covers weeks 13 through 36 rather than a complete calendar year.

---

## 3. How many total transactions occurred each year?

### Query

```sql
SELECT
    calendar_year,
    SUM(transactions) AS total_transactions
FROM data_mart.clean_weekly_sales
GROUP BY calendar_year
ORDER BY calendar_year;
```

### Result

| Year | Total transactions |
|---:|---:|
| 2018 | 346,406,460 |
| 2019 | 365,639,285 |
| 2020 | 375,813,651 |

### Insight

Transaction volume increased in each successive year.

---

## 4. What were the total sales by region and month?

### Query

```sql
SELECT
    region,
    month_number,
    SUM(sales) AS total_sales
FROM data_mart.clean_weekly_sales
GROUP BY
    region,
    month_number
ORDER BY
    region,
    month_number;
```

### Sample result

| Region | Month | Total sales |
|---|---:|---:|
| AFRICA | 3 | 567,767,480 |
| AFRICA | 4 | 1,911,783,504 |
| AFRICA | 5 | 1,647,244,738 |
| AFRICA | 6 | 1,767,559,760 |
| AFRICA | 7 | 1,960,219,710 |
| AFRICA | 8 | 1,809,596,890 |
| AFRICA | 9 | 276,320,987 |
| ASIA | 3 | 529,770,793 |
| ASIA | 4 | 1,804,628,707 |
| ASIA | 5 | 1,526,285,399 |

### Insight

The complete result can be used to identify regional seasonality and compare market size.

---

## 5. How many transactions occurred on each platform?

### Query

```sql
SELECT
    platform,
    SUM(transactions) AS total_transactions
FROM data_mart.clean_weekly_sales
GROUP BY platform
ORDER BY total_transactions DESC;
```

### Result

| Platform | Transactions |
|---|---:|
| Retail | 1,081,934,227 |
| Shopify | 5,925,169 |

### Insight

Retail generated substantially more transactions than Shopify.

---

## 6. What percentage of monthly sales came from Retail and Shopify?

### Query

```sql
WITH platform_sales AS (
    SELECT
        calendar_year,
        month_number,
        platform,
        SUM(sales) AS total_sales
    FROM data_mart.clean_weekly_sales
    GROUP BY
        calendar_year,
        month_number,
        platform
)

SELECT
    calendar_year,
    month_number,

    ROUND(
        100.0
        * SUM(total_sales) FILTER (
            WHERE platform = 'Retail'
        )
        / SUM(total_sales),
        2
    ) AS retail_sales_percentage,

    ROUND(
        100.0
        * SUM(total_sales) FILTER (
            WHERE platform = 'Shopify'
        )
        / SUM(total_sales),
        2
    ) AS shopify_sales_percentage

FROM platform_sales
GROUP BY
    calendar_year,
    month_number
ORDER BY
    calendar_year,
    month_number;
```

### Sample result for 2018

| Year | Month | Retail | Shopify |
|---:|---:|---:|---:|
| 2018 | 3 | 97.92% | 2.08% |
| 2018 | 4 | 97.93% | 2.07% |
| 2018 | 5 | 97.73% | 2.27% |
| 2018 | 6 | 97.76% | 2.24% |
| 2018 | 7 | 97.75% | 2.25% |
| 2018 | 8 | 97.71% | 2.29% |
| 2018 | 9 | 97.68% | 2.32% |

### Insight

Retail produced approximately 98% of monthly sales, although Shopify's share gradually increased.

---

## 7. What percentage of yearly sales came from each demographic?

### Query

```sql
SELECT
    calendar_year,
    demographic,
    SUM(sales) AS demographic_sales,

    ROUND(
        100.0
        * SUM(sales)
        / SUM(SUM(sales)) OVER (
            PARTITION BY calendar_year
        ),
        2
    ) AS sales_percentage

FROM data_mart.clean_weekly_sales
GROUP BY
    calendar_year,
    demographic
ORDER BY
    calendar_year,
    sales_percentage DESC;
```

### Result summary

| Year | Couples | Families | Unknown |
|---:|---:|---:|---:|
| 2018 | 26.38% | 31.99% | 41.63% |
| 2019 | 27.28% | 32.47% | 40.25% |
| 2020 | 28.72% | 32.73% | 38.55% |

### Insight

The unknown demographic group represented the largest sales share, although its contribution declined over time.

---

## 8. Which age-band and demographic groups contributed most to Retail sales?

### Query

```sql
SELECT
    age_band,
    demographic,
    SUM(sales) AS retail_sales,

    ROUND(
        100.0
        * SUM(sales)
        / SUM(SUM(sales)) OVER (),
        2
    ) AS contribution_percentage

FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY
    age_band,
    demographic
ORDER BY retail_sales DESC;
```

### Result

| Age band | Demographic | Retail sales | Contribution |
|---|---|---:|---:|
| Unknown | Unknown | 16,067,285,533 | 40.52% |
| Retirees | Families | 6,634,686,916 | 16.73% |
| Retirees | Couples | 6,370,580,014 | 16.07% |
| Middle Aged | Families | 4,354,091,554 | 10.98% |
| Young Adults | Couples | 2,602,922,797 | 6.56% |
| Middle Aged | Couples | 1,854,160,330 | 4.68% |
| Young Adults | Families | 1,770,889,293 | 4.47% |

### Insight

Customers with missing demographic information accounted for the largest share of Retail sales.

Among known groups, retired families contributed the most.

---

## 9. How should average transaction size be calculated?

### Explanation

A simple average of the `avg_transaction` column would give equal weight to every aggregated row, even though rows contain different transaction volumes.

The correct calculation is:

```text
Total sales ÷ Total transactions
```

### Query

```sql
SELECT
    calendar_year,
    platform,

    ROUND(
        SUM(sales)::NUMERIC
        / SUM(transactions),
        2
    ) AS weighted_average_transaction

FROM data_mart.clean_weekly_sales
GROUP BY
    calendar_year,
    platform
ORDER BY
    calendar_year,
    platform;
```

### Result

| Year | Platform | Average transaction |
|---:|---|---:|
| 2018 | Retail | $36 |
| 2018 | Shopify | $192 |
| 2019 | Retail | $36 |
| 2019 | Shopify | $183 |
| 2020 | Retail | $36 |
| 2020 | Shopify | $179 |

### Insight

Shopify had far fewer transactions but a considerably higher average transaction value.

Its average transaction value declined between 2018 and 2020.
