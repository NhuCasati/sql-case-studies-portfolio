# Data Allocation Challenge

[← Back to Data Bank overview](./README.md)

## Business Requirement

Data Bank wants to estimate storage requirements using three possible allocation models:

1. Previous month-end balance
2. Average balance during the previous 30 days
3. Real-time account balance

Negative balances are treated as requiring zero data allocation.

---

## 1. Running balance after each transaction

### Query

```sql
SELECT
    customer_id,
    txn_date,
    txn_type,
    txn_amount,
    CASE
        WHEN txn_type = 'deposit'
            THEN txn_amount
        ELSE -txn_amount
    END AS transaction_impact,
    SUM(
        CASE
            WHEN txn_type = 'deposit'
                THEN txn_amount
            ELSE -txn_amount
        END
    ) OVER (
        PARTITION BY customer_id
        ORDER BY
            txn_date,
            txn_type,
            txn_amount
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS running_balance
FROM data_bank.customer_transactions
ORDER BY
    customer_id,
    txn_date,
    txn_type,
    txn_amount;
```

### Limitation

The dataset does not contain a transaction ID or timestamp. When several transactions occur on the same date, their exact intraday sequence cannot be confirmed.

---

## 2. Customer balance at the end of each month

```sql
WITH monthly_changes AS (
    SELECT
        customer_id,
        DATE_TRUNC(
            'month',
            txn_date
        )::DATE AS month_start,
        SUM(
            CASE
                WHEN txn_type = 'deposit'
                    THEN txn_amount
                ELSE -txn_amount
            END
        ) AS monthly_change
    FROM data_bank.customer_transactions
    GROUP BY
        customer_id,
        DATE_TRUNC('month', txn_date)
)

SELECT
    customer_id,
    month_start,
    monthly_change,
    SUM(monthly_change) OVER (
        PARTITION BY customer_id
        ORDER BY month_start
    ) AS closing_balance
FROM monthly_changes
ORDER BY
    customer_id,
    month_start;
```

---

## 3. Minimum, average and maximum running balance

```sql
WITH transaction_balances AS (
    SELECT
        customer_id,
        txn_date,
        SUM(
            CASE
                WHEN txn_type = 'deposit'
                    THEN txn_amount
                ELSE -txn_amount
            END
        ) OVER (
            PARTITION BY customer_id
            ORDER BY
                txn_date,
                txn_type,
                txn_amount
            ROWS BETWEEN UNBOUNDED PRECEDING
                     AND CURRENT ROW
        ) AS running_balance
    FROM data_bank.customer_transactions
)

SELECT
    customer_id,
    MIN(running_balance) AS minimum_balance,
    ROUND(
        AVG(running_balance),
        2
    ) AS average_balance,
    MAX(running_balance) AS maximum_balance
FROM transaction_balances
GROUP BY customer_id
ORDER BY customer_id;
```

---

# Option 1: Previous Month-End Balance

## Assumption

The closing balance from one month determines the storage allocation for the following month.

### Query

```sql
WITH monthly_changes AS (
    SELECT
        customer_id,
        DATE_TRUNC(
            'month',
            txn_date
        )::DATE AS month_start,
        SUM(
            CASE
                WHEN txn_type = 'deposit'
                    THEN txn_amount
                ELSE -txn_amount
            END
        ) AS monthly_change
    FROM data_bank.customer_transactions
    GROUP BY
        customer_id,
        DATE_TRUNC('month', txn_date)
),

monthly_balances AS (
    SELECT
        customer_id,
        month_start,
        SUM(monthly_change) OVER (
            PARTITION BY customer_id
            ORDER BY month_start
        ) AS closing_balance
    FROM monthly_changes
)

SELECT
    (
        month_start
        + INTERVAL '1 month'
    )::DATE AS allocation_month,
    SUM(
        GREATEST(
            closing_balance,
            0
        )
    ) AS required_data
FROM monthly_balances
GROUP BY month_start
ORDER BY allocation_month;
```

### Interpretation

This is the simplest model to implement, but storage allocations are updated only once per month.

---

# Option 2: Previous 30-Day Average Balance

## Assumption

The trailing 30-day average on the final day of a month determines allocation for the following month.

### Query

```sql
WITH date_bounds AS (
    SELECT
        MIN(txn_date) AS first_date,
        MAX(txn_date) AS last_date
    FROM data_bank.customer_transactions
),

customers AS (
    SELECT DISTINCT customer_id
    FROM data_bank.customer_transactions
),

calendar AS (
    SELECT
        c.customer_id,
        gs.balance_date::DATE AS balance_date
    FROM customers AS c
    CROSS JOIN date_bounds AS db
    CROSS JOIN LATERAL GENERATE_SERIES(
        db.first_date,
        db.last_date,
        INTERVAL '1 day'
    ) AS gs(balance_date)
),

daily_changes AS (
    SELECT
        customer_id,
        txn_date AS balance_date,
        SUM(
            CASE
                WHEN txn_type = 'deposit'
                    THEN txn_amount
                ELSE -txn_amount
            END
        ) AS daily_change
    FROM data_bank.customer_transactions
    GROUP BY
        customer_id,
        txn_date
),

daily_balances AS (
    SELECT
        c.customer_id,
        c.balance_date,
        SUM(
            COALESCE(
                dc.daily_change,
                0
            )
        ) OVER (
            PARTITION BY c.customer_id
            ORDER BY c.balance_date
        ) AS daily_balance
    FROM calendar AS c
    LEFT JOIN daily_changes AS dc
        ON c.customer_id = dc.customer_id
       AND c.balance_date = dc.balance_date
),

trailing_averages AS (
    SELECT
        customer_id,
        balance_date,
        AVG(
            GREATEST(
                daily_balance,
                0
            )
        ) OVER (
            PARTITION BY customer_id
            ORDER BY balance_date
            ROWS BETWEEN 29 PRECEDING
                     AND CURRENT ROW
        ) AS trailing_30_day_average,
        ROW_NUMBER() OVER (
            PARTITION BY
                customer_id,
                DATE_TRUNC(
                    'month',
                    balance_date
                )
            ORDER BY balance_date DESC
        ) AS month_end_rank
    FROM daily_balances
)

SELECT
    (
        DATE_TRUNC(
            'month',
            balance_date
        )
        + INTERVAL '1 month'
    )::DATE AS allocation_month,
    ROUND(
        SUM(trailing_30_day_average),
        2
    ) AS required_data
FROM trailing_averages
WHERE month_end_rank = 1
GROUP BY DATE_TRUNC(
    'month',
    balance_date
)
ORDER BY allocation_month;
```

### Interpretation

This model smooths short-term balance movements and produces more stable allocation levels.

---

# Option 3: Real-Time Balance

## Assumption

Storage allocation changes every day with each customer's latest account balance.

The monthly requirement is the maximum total allocation recorded during that month.

### Query

```sql
WITH date_bounds AS (
    SELECT
        MIN(txn_date) AS first_date,
        MAX(txn_date) AS last_date
    FROM data_bank.customer_transactions
),

customers AS (
    SELECT DISTINCT customer_id
    FROM data_bank.customer_transactions
),

calendar AS (
    SELECT
        c.customer_id,
        gs.balance_date::DATE AS balance_date
    FROM customers AS c
    CROSS JOIN date_bounds AS db
    CROSS JOIN LATERAL GENERATE_SERIES(
        db.first_date,
        db.last_date,
        INTERVAL '1 day'
    ) AS gs(balance_date)
),

daily_changes AS (
    SELECT
        customer_id,
        txn_date AS balance_date,
        SUM(
            CASE
                WHEN txn_type = 'deposit'
                    THEN txn_amount
                ELSE -txn_amount
            END
        ) AS daily_change
    FROM data_bank.customer_transactions
    GROUP BY
        customer_id,
        txn_date
),

daily_balances AS (
    SELECT
        c.customer_id,
        c.balance_date,
        SUM(
            COALESCE(
                dc.daily_change,
                0
            )
        ) OVER (
            PARTITION BY c.customer_id
            ORDER BY c.balance_date
        ) AS daily_balance
    FROM calendar AS c
    LEFT JOIN daily_changes AS dc
        ON c.customer_id = dc.customer_id
       AND c.balance_date = dc.balance_date
),

daily_requirements AS (
    SELECT
        balance_date,
        SUM(
            GREATEST(
                daily_balance,
                0
            )
        ) AS required_data
    FROM daily_balances
    GROUP BY balance_date
)

SELECT
    DATE_TRUNC(
        'month',
        balance_date
    )::DATE AS allocation_month,
    MAX(required_data) AS peak_required_data
FROM daily_requirements
GROUP BY DATE_TRUNC(
    'month',
    balance_date
)
ORDER BY allocation_month;
```

---

## Model Comparison

| Option | Advantage | Limitation |
|---|---|---|
| Previous month-end | Simple and predictable | Responds slowly to balance changes |
| Previous 30-day average | Stable and less volatile | More computationally complex |
| Real-time | Closely reflects account value | Highest technical and operational complexity |

## Recommendation

The trailing 30-day average provides a practical balance between accuracy and stability.

Real-time allocation is more responsive, but it requires significantly more frequent calculations and infrastructure updates.
