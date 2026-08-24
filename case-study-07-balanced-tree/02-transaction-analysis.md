# Transaction Analysis

[← Back to Balanced Tree overview](./README.md)

## Overview

This section analyses transaction volume, product variety, transaction revenue, discounts and membership behaviour.

---

## 1. How many unique transactions were there?

### Query

```sql
SELECT
    COUNT(DISTINCT txn_id) AS unique_transactions
FROM balanced_tree.sales;
```

### Result

| Unique transactions |
|---:|
| 2,500 |

---

## 2. What was the average number of unique products purchased per transaction?

### Query

```sql
WITH products_per_transaction AS (
    SELECT
        txn_id,
        COUNT(DISTINCT prod_id) AS unique_products
    FROM balanced_tree.sales
    GROUP BY txn_id
)

SELECT
    ROUND(
        AVG(unique_products),
        2
    ) AS average_unique_products
FROM products_per_transaction;
```

### Result

| Average unique products |
|---:|
| 6.04 |

### Insight

A typical transaction contained approximately six different products.

This metric counts unique products, not the total number of units.

---

## 3. What were the revenue percentiles per transaction?

### SQL approach

First calculate gross revenue for each transaction, then use `PERCENTILE_CONT()`.

### Query

```sql
WITH transaction_revenue AS (
    SELECT
        txn_id,
        SUM(qty * price) AS gross_revenue
    FROM balanced_tree.sales
    GROUP BY txn_id
)

SELECT
    PERCENTILE_CONT(0.25)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS percentile_25,

    PERCENTILE_CONT(0.50)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS percentile_50,

    PERCENTILE_CONT(0.75)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS percentile_75

FROM transaction_revenue;
```

### Result

| 25th percentile | Median | 75th percentile |
|---:|---:|---:|
| $375.75 | $509.50 | $647.00 |

### Insight

Half of all transactions generated gross revenue of $509.50 or less.

---

## 4. What was the average discount value per transaction?

### Query

```sql
WITH transaction_discounts AS (
    SELECT
        txn_id,

        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ) AS discount_value

    FROM balanced_tree.sales
    GROUP BY txn_id
)

SELECT
    ROUND(
        AVG(discount_value),
        2
    ) AS average_discount
FROM transaction_discounts;
```

### Result

| Average discount |
|---:|
| $62.49 |

---

## 5. What was the transaction split between members and non-members?

### Query

```sql
WITH transaction_membership AS (
    SELECT
        txn_id,
        BOOL_OR(member) AS member
    FROM balanced_tree.sales
    GROUP BY txn_id
)

SELECT
    CASE
        WHEN member THEN 'Member'
        ELSE 'Non-member'
    END AS customer_type,

    COUNT(*) AS transactions,

    ROUND(
        100.0
        * COUNT(*)
        / SUM(COUNT(*)) OVER (),
        2
    ) AS transaction_percentage

FROM transaction_membership
GROUP BY member
ORDER BY member DESC;
```

### Result

| Customer type | Transactions | Percentage |
|---|---:|---:|
| Member | 1,505 | 60.20% |
| Non-member | 995 | 39.80% |

### Insight

Members generated approximately three-fifths of all transactions.

---

## 6. What was the average gross revenue for member and non-member transactions?

### Query

```sql
WITH transaction_revenue AS (
    SELECT
        txn_id,
        BOOL_OR(member) AS member,
        SUM(qty * price) AS gross_revenue
    FROM balanced_tree.sales
    GROUP BY txn_id
)

SELECT
    CASE
        WHEN member THEN 'Member'
        ELSE 'Non-member'
    END AS customer_type,

    ROUND(
        AVG(gross_revenue),
        2
    ) AS average_gross_revenue

FROM transaction_revenue
GROUP BY member
ORDER BY member DESC;
```

### Result

| Customer type | Average gross revenue |
|---|---:|
| Member | $516.27 |
| Non-member | $515.04 |

### Insight

The average transaction value was almost identical for members and non-members.

The membership programme appears to influence transaction frequency more than transaction value.
