# Customer Transactions

[← Back to Data Bank overview](./README.md)

## Transaction Rules

- Deposits are positive cash flows.
- Purchases are negative cash flows.
- Withdrawals are negative cash flows.

---

## 1. What are the count and total amount for each transaction type?

### Query

```sql
SELECT
    txn_type,
    COUNT(*) AS transaction_count,
    SUM(txn_amount) AS total_amount
FROM data_bank.customer_transactions
GROUP BY txn_type
ORDER BY transaction_count DESC;
```

### Result

| Transaction type | Count | Total amount |
|---|---:|---:|
| Deposit | 2,671 | 1,359,168 |
| Purchase | 1,617 | 806,537 |
| Withdrawal | 1,580 | 793,003 |

### Insight

Deposits were the most frequent transaction type and also represented the largest total monetary value.

---

## 2. What were the average historical deposit count and amount?

### Query

```sql
WITH customer_deposits AS (
    SELECT
        customer_id,
        COUNT(*) AS deposit_count,
        AVG(txn_amount) AS average_deposit_amount
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY customer_id
)

SELECT
    ROUND(
        AVG(deposit_count),
        2
    ) AS average_deposit_count,
    ROUND(
        AVG(average_deposit_amount),
        2
    ) AS average_deposit_amount
FROM customer_deposits;
```

### Result

| Average deposit count | Average amount |
|---:|---:|
| Approximately 5 | $508.61 |

### Insight

A typical customer made approximately five deposits, with an average deposit value of about $509.

---

## 3. How many customers met the monthly transaction criteria?

The required customers must make:

- More than one deposit
- At least one purchase or one withdrawal
- All within the same calendar month

### Query

```sql
WITH monthly_activity AS (
    SELECT
        customer_id,
        DATE_TRUNC(
            'month',
            txn_date
        )::DATE AS transaction_month,
        COUNT(*) FILTER (
            WHERE txn_type = 'deposit'
        ) AS deposit_count,
        COUNT(*) FILTER (
            WHERE txn_type = 'purchase'
        ) AS purchase_count,
        COUNT(*) FILTER (
            WHERE txn_type = 'withdrawal'
        ) AS withdrawal_count
    FROM data_bank.customer_transactions
    GROUP BY
        customer_id,
        DATE_TRUNC('month', txn_date)
)

SELECT
    transaction_month,
    COUNT(*) AS customer_count
FROM monthly_activity
WHERE deposit_count > 1
  AND (
      purchase_count >= 1
      OR withdrawal_count >= 1
  )
GROUP BY transaction_month
ORDER BY transaction_month;
```

### Result

Add the monthly counts returned by DB Fiddle.

### Insight

This metric identifies customers with both active funding and spending behaviour.

---

## 4. What was each customer's closing balance at the end of every month?

### SQL approach

1. Calculate each customer's net monthly change.
2. Generate every month for every customer.
3. Fill months without transactions with zero.
4. Calculate a cumulative closing balance.

### Query

```sql
WITH date_bounds AS (
    SELECT
        DATE_TRUNC(
            'month',
            MIN(txn_date)
        )::DATE AS first_month,
        DATE_TRUNC(
            'month',
            MAX(txn_date)
        )::DATE AS last_month
    FROM data_bank.customer_transactions
),

customers AS (
    SELECT DISTINCT
        customer_id
    FROM data_bank.customer_transactions
),

months AS (
    SELECT
        GENERATE_SERIES(
            first_month,
            last_month,
            INTERVAL '1 month'
        )::DATE AS month_start
    FROM date_bounds
),

customer_months AS (
    SELECT
        c.customer_id,
        m.month_start
    FROM customers AS c
    CROSS JOIN months AS m
),

monthly_changes AS (
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
        cm.customer_id,
        cm.month_start,
        COALESCE(
            mc.monthly_change,
            0
        ) AS monthly_change,
        SUM(
            COALESCE(
                mc.monthly_change,
                0
            )
        ) OVER (
            PARTITION BY cm.customer_id
            ORDER BY cm.month_start
            ROWS BETWEEN UNBOUNDED PRECEDING
                     AND CURRENT ROW
        ) AS closing_balance
    FROM customer_months AS cm
    LEFT JOIN monthly_changes AS mc
        ON cm.customer_id = mc.customer_id
       AND cm.month_start = mc.month_start
)

SELECT
    customer_id,
    month_start,
    monthly_change,
    closing_balance
FROM monthly_balances
ORDER BY
    customer_id,
    month_start;
```

### Result

The complete query returns one row for every customer and calendar month.

Add a short sample to the portfolio:

| Customer | Month | Monthly change | Closing balance |
|---:|---|---:|---:|
| 1 | 2020-01-01 | Add result | Add result |
| 1 | 2020-02-01 | Add result | Add result |
| 1 | 2020-03-01 | Add result | Add result |

### Insight

The closing balance carries forward between months, including months in which a customer makes no transactions.

---

## 5. What percentage of customers increased their closing balance by more than 5%?

### Assumption

A customer qualifies when their closing balance increases by more than 5% between any two consecutive months.

The absolute previous balance is used as the denominator because some customers have negative balances.

### Query

```sql
WITH date_bounds AS (
    SELECT
        DATE_TRUNC(
            'month',
            MIN(txn_date)
        )::DATE AS first_month,
        DATE_TRUNC(
            'month',
            MAX(txn_date)
        )::DATE AS last_month
    FROM data_bank.customer_transactions
),

customers AS (
    SELECT DISTINCT customer_id
    FROM data_bank.customer_transactions
),

months AS (
    SELECT
        GENERATE_SERIES(
            first_month,
            last_month,
            INTERVAL '1 month'
        )::DATE AS month_start
    FROM date_bounds
),

customer_months AS (
    SELECT
        c.customer_id,
        m.month_start
    FROM customers AS c
    CROSS JOIN months AS m
),

monthly_changes AS (
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
        cm.customer_id,
        cm.month_start,
        SUM(
            COALESCE(
                mc.monthly_change,
                0
            )
        ) OVER (
            PARTITION BY cm.customer_id
            ORDER BY cm.month_start
        ) AS closing_balance
    FROM customer_months AS cm
    LEFT JOIN monthly_changes AS mc
        ON cm.customer_id = mc.customer_id
       AND cm.month_start = mc.month_start
),

balance_comparison AS (
    SELECT
        customer_id,
        month_start,
        closing_balance,
        LAG(closing_balance) OVER (
            PARTITION BY customer_id
            ORDER BY month_start
        ) AS previous_balance
    FROM monthly_balances
),

customer_growth AS (
    SELECT
        customer_id,
        MAX(
            CASE
                WHEN previous_balance <> 0
                 AND (
                     closing_balance
                     - previous_balance
                 ) / ABS(
                     previous_balance::NUMERIC
                 ) > 0.05
                    THEN 1
                ELSE 0
            END
        ) AS increased_over_five_percent
    FROM balance_comparison
    GROUP BY customer_id
)

SELECT
    ROUND(
        100.0
        * SUM(increased_over_five_percent)
        / COUNT(*),
        1
    ) AS customer_percentage
FROM customer_growth;
```

### Result

Run the query and add the resulting percentage.

### Insight

The metric shows how many customers experienced meaningful balance growth during the analysis period.
