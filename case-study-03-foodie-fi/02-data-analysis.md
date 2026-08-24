# Data Analysis

[← Back to Foodie-Fi overview](./README.md)

---

## 1. How many customers has Foodie-Fi ever had?

### Query

```sql
SELECT
    COUNT(DISTINCT customer_id) AS total_customers
FROM foodie_fi.subscriptions;
```

### Result

| Total customers |
|---:|
| 1,000 |

### Insight

Foodie-Fi acquired 1,000 unique customers in the available dataset.

---

## 2. What was the monthly distribution of trial-plan start dates?

### SQL approach

Filter for the trial plan and group trial start dates by the beginning of each month.

### Query

```sql
SELECT
    DATE_TRUNC(
        'month',
        start_date
    )::DATE AS trial_month,
    COUNT(*) AS trial_starts
FROM foodie_fi.subscriptions
WHERE plan_id = 0
GROUP BY trial_month
ORDER BY trial_month;
```

### Result

| Trial month | Trial starts |
|---|---:|
| 2020-01-01 | 88 |
| 2020-02-01 | 68 |
| 2020-03-01 | 94 |
| 2020-04-01 | 81 |
| 2020-05-01 | 88 |
| 2020-06-01 | 79 |
| 2020-07-01 | 89 |
| 2020-08-01 | 88 |
| 2020-09-01 | 87 |
| 2020-10-01 | 79 |
| 2020-11-01 | 75 |
| 2020-12-01 | 84 |

### Insight

March recorded the highest number of new trials, while February recorded the lowest.

---

## 3. Which plan events occurred after 2020?

### Query

```sql
SELECT
    p.plan_id,
    p.plan_name,
    COUNT(*) AS event_count
FROM foodie_fi.subscriptions AS s
INNER JOIN foodie_fi.plans AS p
    ON s.plan_id = p.plan_id
WHERE s.start_date >= DATE '2021-01-01'
GROUP BY
    p.plan_id,
    p.plan_name
ORDER BY p.plan_id;
```

### Result

| Plan ID | Plan | Events |
|---:|---|---:|
| 1 | Basic monthly | 8 |
| 2 | Pro monthly | 60 |
| 3 | Pro annual | 63 |
| 4 | Churn | 71 |

### Insight

No new trials began after 2020 because the provided customer cohort joined during 2020. Plan changes continued into 2021.

---

## 4. How many customers churned?

### Query

```sql
WITH customer_totals AS (
    SELECT
        COUNT(DISTINCT customer_id) AS total_customers
    FROM foodie_fi.subscriptions
)

SELECT
    COUNT(DISTINCT s.customer_id) AS churned_customers,
    ROUND(
        100.0
        * COUNT(DISTINCT s.customer_id)
        / ct.total_customers,
        1
    ) AS churn_percentage
FROM foodie_fi.subscriptions AS s
CROSS JOIN customer_totals AS ct
WHERE s.plan_id = 4
GROUP BY ct.total_customers;
```

### Result

| Churned customers | Churn percentage |
|---:|---:|
| 307 | 30.7% |

### Insight

Almost one-third of Foodie-Fi's customers eventually churned.

---

## 5. How many customers churned immediately after the trial?

### SQL approach

Use `LEAD()` to identify the plan immediately following each customer's trial.

### Query

```sql
WITH plan_transitions AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        LEAD(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS next_plan_id
    FROM foodie_fi.subscriptions
),

customer_totals AS (
    SELECT
        COUNT(DISTINCT customer_id) AS total_customers
    FROM foodie_fi.subscriptions
)

SELECT
    COUNT(*) AS immediate_trial_churns,
    ROUND(
        100.0
        * COUNT(*)
        / ct.total_customers
    ) AS churn_percentage
FROM plan_transitions
CROSS JOIN customer_totals AS ct
WHERE plan_id = 0
  AND next_plan_id = 4
GROUP BY ct.total_customers;
```

### Result

| Immediate trial churns | Percentage |
|---:|---:|
| 92 | 9% |

### Insight

Approximately 9% of all customers left Foodie-Fi immediately after completing the free trial.

---

## 6. Which plans did customers select after the trial?

### Query

```sql
WITH plan_transitions AS (
    SELECT
        customer_id,
        plan_id,
        LEAD(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS next_plan_id
    FROM foodie_fi.subscriptions
),

customer_totals AS (
    SELECT
        COUNT(DISTINCT customer_id) AS total_customers
    FROM foodie_fi.subscriptions
)

SELECT
    pt.next_plan_id AS plan_id,
    p.plan_name,
    COUNT(*) AS customer_count,
    ROUND(
        100.0
        * COUNT(*)
        / ct.total_customers,
        1
    ) AS customer_percentage
FROM plan_transitions AS pt
INNER JOIN foodie_fi.plans AS p
    ON pt.next_plan_id = p.plan_id
CROSS JOIN customer_totals AS ct
WHERE pt.plan_id = 0
GROUP BY
    pt.next_plan_id,
    p.plan_name,
    ct.total_customers
ORDER BY pt.next_plan_id;
```

### Result

| Plan ID | Plan | Customers | Percentage |
|---:|---|---:|---:|
| 1 | Basic monthly | 546 | 54.6% |
| 2 | Pro monthly | 325 | 32.5% |
| 3 | Pro annual | 37 | 3.7% |
| 4 | Churn | 92 | 9.2% |

### Insight

Most customers selected the basic monthly plan after the trial. Only 3.7% moved directly to the annual plan.

---

## 7. What was the plan breakdown on 31 December 2020?

### SQL approach

For each customer, rank their plans up to the reporting date and retain the most recent plan.

### Query

```sql
WITH plans_at_reporting_date AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY start_date DESC
        ) AS plan_rank
    FROM foodie_fi.subscriptions
    WHERE start_date <= DATE '2020-12-31'
),

customer_totals AS (
    SELECT
        COUNT(DISTINCT customer_id) AS total_customers
    FROM foodie_fi.subscriptions
    WHERE start_date <= DATE '2020-12-31'
)

SELECT
    p.plan_id,
    p.plan_name,
    COUNT(*) AS customer_count,
    ROUND(
        100.0
        * COUNT(*)
        / ct.total_customers,
        1
    ) AS customer_percentage
FROM plans_at_reporting_date AS r
INNER JOIN foodie_fi.plans AS p
    ON r.plan_id = p.plan_id
CROSS JOIN customer_totals AS ct
WHERE r.plan_rank = 1
GROUP BY
    p.plan_id,
    p.plan_name,
    ct.total_customers
ORDER BY p.plan_id;
```

### Result

| Plan ID | Plan | Customers | Percentage |
|---:|---|---:|---:|
| 0 | Trial | 19 | 1.9% |
| 1 | Basic monthly | 224 | 22.4% |
| 2 | Pro monthly | 326 | 32.6% |
| 3 | Pro annual | 195 | 19.5% |
| 4 | Churn | 236 | 23.6% |

### Insight

The pro monthly plan was the largest active plan at the end of 2020. However, the churn group was larger than either the basic monthly or annual group.

---

## 8. How many customers upgraded to the annual plan in 2020?

### Query

```sql
SELECT
    COUNT(DISTINCT customer_id) AS annual_customers
FROM foodie_fi.subscriptions
WHERE plan_id = 3
  AND start_date >= DATE '2020-01-01'
  AND start_date < DATE '2021-01-01';
```

### Result

| Annual customers |
|---:|
| 195 |

### Insight

A total of 195 customers moved to the annual plan during 2020.

---

## 9. How long did customers take to move to the annual plan?

### SQL approach

Compare each customer's trial start date with the date they began the annual plan.

### Query

```sql
WITH trial_dates AS (
    SELECT
        customer_id,
        start_date AS trial_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 0
),

annual_dates AS (
    SELECT
        customer_id,
        start_date AS annual_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 3
)

SELECT
    ROUND(
        AVG(
            a.annual_date - t.trial_date
        ),
        2
    ) AS average_days_to_annual
FROM trial_dates AS t
INNER JOIN annual_dates AS a
    ON t.customer_id = a.customer_id;
```

### Result

| Average days |
|---:|
| 104.62 |

### Insight

Customers who selected the annual plan took approximately 105 days on average to move from joining Foodie-Fi to the annual subscription.

---

## 10. How were annual upgrades distributed across 30-day periods?

### Query

```sql
WITH trial_dates AS (
    SELECT
        customer_id,
        start_date AS trial_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 0
),

annual_dates AS (
    SELECT
        customer_id,
        start_date AS annual_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 3
),

upgrade_times AS (
    SELECT
        t.customer_id,
        a.annual_date - t.trial_date AS days_to_upgrade
    FROM trial_dates AS t
    INNER JOIN annual_dates AS a
        ON t.customer_id = a.customer_id
),

upgrade_buckets AS (
    SELECT
        customer_id,
        days_to_upgrade,
        FLOOR(
            (days_to_upgrade - 1) / 30.0
        )::INTEGER AS bucket_number
    FROM upgrade_times
)

SELECT
    CASE
        WHEN bucket_number = 0
            THEN '0-30 days'
        ELSE
            (
                bucket_number * 30 + 1
            )::TEXT
            || '-'
            || (
                (bucket_number + 1) * 30
            )::TEXT
            || ' days'
    END AS upgrade_period,
    COUNT(*) AS customer_count
FROM upgrade_buckets
GROUP BY bucket_number
ORDER BY bucket_number;
```

### Result

| Upgrade period | Customers |
|---|---:|
| 0-30 days | 48 |
| 31-60 days | 25 |
| 61-90 days | 33 |
| 91-120 days | 35 |
| 121-150 days | 43 |
| 151-180 days | 35 |
| 181-210 days | 27 |
| 211-240 days | 4 |
| 241-270 days | 5 |
| 271-300 days | 1 |
| 301-330 days | 1 |
| 331-360 days | 1 |

### Insight

Annual upgrades were concentrated within the first 210 days. Very few customers waited longer than nine months to select the annual plan.

---

## 11. How many customers downgraded from pro monthly to basic monthly in 2020?

### Query

```sql
WITH plan_transitions AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        LEAD(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS next_plan_id,
        LEAD(start_date) OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS next_plan_date
    FROM foodie_fi.subscriptions
)

SELECT
    COUNT(*) AS downgraded_customers
FROM plan_transitions
WHERE plan_id = 2
  AND next_plan_id = 1
  AND next_plan_date >= DATE '2020-01-01'
  AND next_plan_date < DATE '2021-01-01';
```

### Result

| Downgraded customers |
|---:|
| 0 |

### Insight

No customer moved from the pro monthly plan back to the basic monthly plan during 2020.
