# Before-and-After Analysis

[← Back to Data Mart overview](./README.md)

## Objective

Measure the effect of Data Mart's sustainable-packaging change.

The change was introduced during the week beginning:

```text
15 June 2020
```

That week is included in the **after** period.

---

## Reusable Analysis Query

This query calculates both the four-week and twelve-week comparison periods for every year.

```sql
WITH event_week AS (
    SELECT
        week_number AS baseline_week
    FROM data_mart.clean_weekly_sales
    WHERE week_date = DATE '2020-06-15'
    LIMIT 1
),

period_sales AS (
    SELECT
        calendar_year,

        SUM(sales) FILTER (
            WHERE week_number BETWEEN
                baseline_week - 4
                AND baseline_week - 1
        ) AS before_4_weeks,

        SUM(sales) FILTER (
            WHERE week_number BETWEEN
                baseline_week
                AND baseline_week + 3
        ) AS after_4_weeks,

        SUM(sales) FILTER (
            WHERE week_number BETWEEN
                baseline_week - 12
                AND baseline_week - 1
        ) AS before_12_weeks,

        SUM(sales) FILTER (
            WHERE week_number BETWEEN
                baseline_week
                AND baseline_week + 11
        ) AS after_12_weeks

    FROM data_mart.clean_weekly_sales
    CROSS JOIN event_week
    GROUP BY calendar_year
)

SELECT
    calendar_year,

    before_4_weeks,
    after_4_weeks,

    after_4_weeks
    - before_4_weeks AS change_4_weeks,

    ROUND(
        100.0
        * (
            after_4_weeks
            - before_4_weeks
        )
        / before_4_weeks,
        2
    ) AS percentage_change_4_weeks,

    before_12_weeks,
    after_12_weeks,

    after_12_weeks
    - before_12_weeks AS change_12_weeks,

    ROUND(
        100.0
        * (
            after_12_weeks
            - before_12_weeks
        )
        / before_12_weeks,
        2
    ) AS percentage_change_12_weeks

FROM period_sales
ORDER BY calendar_year;
```

---

## 1. Four-week impact in 2020

### Result

| Before | After | Change | Percentage |
|---:|---:|---:|---:|
| 2,345,878,357 | 2,318,994,169 | -26,884,188 | -1.15% |

### Insight

Sales declined by approximately $26.9 million during the four-week period following the change.

---

## 2. Twelve-week impact in 2020

### Result

| Before | After | Change | Percentage |
|---:|---:|---:|---:|
| 7,126,273,147 | 6,973,947,753 | -152,325,394 | -2.14% |

### Insight

The decline became larger across the twelve-week period, suggesting that the sales impact was not limited to the initial implementation weeks.

---

## 3. Comparison with 2018 and 2019

### Result

| Year | 4-week before | 4-week after | 4-week change | 12-week before | 12-week after | 12-week change |
|---:|---:|---:|---:|---:|---:|---:|
| 2018 | 2,125,140,809 | 2,129,242,914 | +0.19% | 6,396,562,317 | 6,500,818,510 | +1.63% |
| 2019 | 2,249,989,796 | 2,252,326,390 | +0.10% | 6,883,386,397 | 6,862,646,103 | -0.30% |
| 2020 | 2,345,878,357 | 2,318,994,169 | -1.15% | 7,126,273,147 | 6,973,947,753 | -2.14% |

### Insight

The four-week comparison was slightly positive in 2018 and 2019 but negative in 2020.

The twelve-week decline in 2020 was also substantially larger than the small decline recorded in 2019.

This provides evidence that the sustainable-packaging change may have contributed to weaker performance, although other factors may also have affected sales.

---

## Analytical Limitation

Before-and-after analysis identifies an association, not definite causation.

A stronger evaluation would also include:

- Product-level data
- Pricing changes
- Promotion activity
- Store availability
- Customer feedback
- Competitor activity
- A control group unaffected by the packaging change
