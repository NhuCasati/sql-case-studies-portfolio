# Payments Challenge

[← Back to Foodie-Fi overview](./README.md)

## Business Requirement

Create a `payments` table for 2020 containing:

- Customer ID
- Plan ID
- Plan name
- Payment date
- Payment amount
- Payment order

The following rules must be applied:

1. Monthly payments occur on the same day each month as the plan start date.
2. Basic-to-pro upgrades take effect immediately.
3. The amount already paid for the current basic billing period is deducted from the upgrade payment.
4. Pro-monthly-to-annual upgrades occur at the end of the monthly billing period.
5. Annual plans generate one payment.
6. Customers stop making payments after churning.
7. Only payments made during 2020 are included.

---

## SQL Approach

The solution:

1. Uses `LEAD()` and `LAG()` to identify previous and next plans.
2. Uses `GENERATE_SERIES()` to create recurring monthly payments.
3. Creates one-time annual payments.
4. Adjusts basic-to-pro upgrade charges.
5. Numbers each customer's payments chronologically.

---

## Query

```sql
DROP TABLE IF EXISTS foodie_fi.payments;

CREATE TABLE foodie_fi.payments AS

WITH subscription_periods AS (
    SELECT
        s.customer_id,
        s.plan_id,
        s.start_date,
        LAG(s.plan_id) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date
        ) AS previous_plan_id,
        LEAD(s.start_date) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date
        ) AS next_start_date
    FROM foodie_fi.subscriptions AS s
),

monthly_payments AS (
    SELECT
        sp.customer_id,
        sp.plan_id,
        p.plan_name,
        gs.payment_date::DATE AS payment_date,
        p.price AS amount
    FROM subscription_periods AS sp
    INNER JOIN foodie_fi.plans AS p
        ON sp.plan_id = p.plan_id
    CROSS JOIN LATERAL GENERATE_SERIES(
        sp.start_date::TIMESTAMP,
        (
            LEAST(
                COALESCE(
                    sp.next_start_date,
                    DATE '2021-01-01'
                ),
                DATE '2021-01-01'
            ) - 1
        )::TIMESTAMP,
        INTERVAL '1 month'
    ) AS gs(payment_date)
    WHERE sp.plan_id IN (1, 2)
      AND sp.start_date < DATE '2021-01-01'
),

annual_payments AS (
    SELECT
        sp.customer_id,
        sp.plan_id,
        p.plan_name,
        sp.start_date AS payment_date,
        p.price AS amount
    FROM subscription_periods AS sp
    INNER JOIN foodie_fi.plans AS p
        ON sp.plan_id = p.plan_id
    WHERE sp.plan_id = 3
      AND sp.start_date >= DATE '2020-01-01'
      AND sp.start_date < DATE '2021-01-01'
),

raw_payments AS (
    SELECT * FROM monthly_payments

    UNION ALL

    SELECT * FROM annual_payments
),

ordered_raw_payments AS (
    SELECT
        customer_id,
        plan_id,
        plan_name,
        payment_date,
        amount,
        LAG(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY payment_date, plan_id
        ) AS previous_payment_plan,
        LAG(payment_date) OVER (
            PARTITION BY customer_id
            ORDER BY payment_date, plan_id
        ) AS previous_payment_date,
        LAG(amount) OVER (
            PARTITION BY customer_id
            ORDER BY payment_date, plan_id
        ) AS previous_payment_amount
    FROM raw_payments
),

adjusted_payments AS (
    SELECT
        customer_id,
        plan_id,
        plan_name,
        payment_date,
        CASE
            WHEN plan_id IN (2, 3)
                 AND previous_payment_plan = 1
                 AND payment_date
                     < previous_payment_date
                       + INTERVAL '1 month'
                THEN amount - previous_payment_amount
            ELSE amount
        END AS amount
    FROM ordered_raw_payments
)

SELECT
    customer_id,
    plan_id,
    plan_name,
    payment_date,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY payment_date, plan_id
    ) AS payment_order
FROM adjusted_payments
WHERE payment_date >= DATE '2020-01-01'
  AND payment_date < DATE '2021-01-01'
ORDER BY
    customer_id,
    payment_date;
```

---

## Validate the Table

```sql
SELECT *
FROM foodie_fi.payments
WHERE customer_id IN (
    1, 2, 13, 15, 16, 18, 19
)
ORDER BY
    customer_id,
    payment_date;
```

---

## Example Results

| Customer | Plan | Payment date | Amount | Order |
|---:|---|---|---:|---:|
| 1 | Basic monthly | 2020-08-08 | 9.90 | 1 |
| 1 | Basic monthly | 2020-09-08 | 9.90 | 2 |
| 1 | Basic monthly | 2020-10-08 | 9.90 | 3 |
| 1 | Basic monthly | 2020-11-08 | 9.90 | 4 |
| 1 | Basic monthly | 2020-12-08 | 9.90 | 5 |
| 2 | Pro annual | 2020-09-27 | 199.00 | 1 |
| 13 | Basic monthly | 2020-12-22 | 9.90 | 1 |
| 15 | Pro monthly | 2020-03-24 | 19.90 | 1 |
| 15 | Pro monthly | 2020-04-24 | 19.90 | 2 |
| 16 | Pro annual | 2020-10-21 | 189.10 | 6 |
| 19 | Pro annual | 2020-08-29 | 199.00 | 3 |

---

## Validation Queries

### Total number of payments

```sql
SELECT
    COUNT(*) AS payment_count
FROM foodie_fi.payments;
```

### Total 2020 revenue

```sql
SELECT
    ROUND(
        SUM(amount),
        2
    ) AS total_revenue
FROM foodie_fi.payments;
```

### Revenue by plan

```sql
SELECT
    plan_name,
    COUNT(*) AS payments,
    ROUND(
        SUM(amount),
        2
    ) AS revenue
FROM foodie_fi.payments
GROUP BY plan_name
ORDER BY revenue DESC;
```

---

## Insight

Creating the payments table converts subscription events into a transaction-level revenue dataset.

This table can support:

- Monthly recurring revenue analysis
- Revenue forecasting
- Customer lifetime-value calculations
- Payment reconciliation
- Revenue analysis by subscription plan
