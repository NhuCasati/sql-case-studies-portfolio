# Monthly Reporting Challenge

[← Back to Balanced Tree overview](./README.md)

## Objective

Create a reusable SQL report that can run at the beginning of every month.

The report should analyse the previous month without requiring changes throughout every query.

Only the `report_month` parameter must be updated.

---

## Step 1: Set the reporting month

For January:

```sql
DATE '2021-01-01'
```

For February:

```sql
DATE '2021-02-01'
```

---

## Step 2: Create the monthly reporting scope

```sql
DROP TABLE IF EXISTS monthly_sales_scope;

CREATE TEMP TABLE monthly_sales_scope AS

WITH parameters AS (
    SELECT
        DATE '2021-01-01' AS report_month
)

SELECT
    s.*,
    pd.product_name,
    pd.category_name,
    pd.segment_name,
    pd.style_name

FROM balanced_tree.sales AS s

INNER JOIN balanced_tree.product_details AS pd
    ON s.prod_id = pd.product_id

CROSS JOIN parameters AS p

WHERE s.start_txn_time >= p.report_month
  AND s.start_txn_time
      < p.report_month + INTERVAL '1 month';
```

To run the February report, change only:

```sql
DATE '2021-01-01'
```

to:

```sql
DATE '2021-02-01'
```

---

## Report 1: High-Level Summary

This output supports:

- Total quantity
- Gross revenue
- Discount value
- Net revenue
- Unique transactions

```sql
SELECT
    SUM(qty) AS total_quantity,

    SUM(
        qty * price
    ) AS gross_revenue,

    ROUND(
        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ),
        2
    ) AS discount_value,

    ROUND(
        SUM(
            qty
            * price
            * (
                1 - discount::NUMERIC / 100
            )
        ),
        2
    ) AS net_revenue,

    COUNT(
        DISTINCT txn_id
    ) AS unique_transactions

FROM monthly_sales_scope;
```

---

## Report 2: Transaction Metrics

This output supports:

- Average unique products
- Revenue percentiles
- Average discount per transaction

```sql
WITH transaction_metrics AS (
    SELECT
        txn_id,

        COUNT(
            DISTINCT prod_id
        ) AS unique_products,

        SUM(
            qty * price
        ) AS gross_revenue,

        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ) AS discount_value

    FROM monthly_sales_scope
    GROUP BY txn_id
)

SELECT
    ROUND(
        AVG(unique_products),
        2
    ) AS average_unique_products,

    PERCENTILE_CONT(0.25)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS revenue_percentile_25,

    PERCENTILE_CONT(0.50)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS revenue_median,

    PERCENTILE_CONT(0.75)
        WITHIN GROUP (
            ORDER BY gross_revenue
        ) AS revenue_percentile_75,

    ROUND(
        AVG(discount_value),
        2
    ) AS average_discount

FROM transaction_metrics;
```

---

## Report 3: Member Performance

```sql
WITH transaction_revenue AS (
    SELECT
        txn_id,
        BOOL_OR(member) AS member,
        SUM(qty * price) AS gross_revenue

    FROM monthly_sales_scope
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
    ) AS transaction_percentage,

    ROUND(
        AVG(gross_revenue),
        2
    ) AS average_transaction_revenue,

    SUM(
        gross_revenue
    ) AS total_gross_revenue

FROM transaction_revenue
GROUP BY member
ORDER BY member DESC;
```

---

## Report 4: Product Performance

```sql
WITH product_performance AS (
    SELECT
        prod_id,
        product_name,
        category_name,
        segment_name,

        SUM(qty) AS total_quantity,

        SUM(
            qty * price
        ) AS gross_revenue,

        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ) AS discount_value,

        COUNT(
            DISTINCT txn_id
        ) AS product_transactions

    FROM monthly_sales_scope

    GROUP BY
        prod_id,
        product_name,
        category_name,
        segment_name
),

monthly_transaction_count AS (
    SELECT
        COUNT(
            DISTINCT txn_id
        ) AS total_transactions

    FROM monthly_sales_scope
)

SELECT
    pp.*,

    ROUND(
        100.0
        * pp.product_transactions
        / mtc.total_transactions,
        2
    ) AS transaction_penetration,

    DENSE_RANK() OVER (
        ORDER BY pp.gross_revenue DESC
    ) AS revenue_rank,

    DENSE_RANK() OVER (
        PARTITION BY pp.segment_name
        ORDER BY pp.total_quantity DESC
    ) AS segment_quantity_rank,

    DENSE_RANK() OVER (
        PARTITION BY pp.category_name
        ORDER BY pp.total_quantity DESC
    ) AS category_quantity_rank

FROM product_performance AS pp
CROSS JOIN monthly_transaction_count AS mtc

ORDER BY revenue_rank;
```

---

## Report 5: Segment Performance

```sql
SELECT
    category_name,
    segment_name,

    SUM(qty) AS total_quantity,

    SUM(
        qty * price
    ) AS gross_revenue,

    ROUND(
        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ),
        2
    ) AS discount_value

FROM monthly_sales_scope

GROUP BY
    category_name,
    segment_name

ORDER BY gross_revenue DESC;
```

---

## Report 6: Category Performance

```sql
WITH category_performance AS (
    SELECT
        category_name,

        SUM(qty) AS total_quantity,

        SUM(
            qty * price
        ) AS gross_revenue,

        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ) AS discount_value

    FROM monthly_sales_scope
    GROUP BY category_name
)

SELECT
    category_name,
    total_quantity,
    gross_revenue,

    ROUND(
        discount_value,
        2
    ) AS discount_value,

    ROUND(
        100.0
        * gross_revenue
        / SUM(gross_revenue) OVER (),
        2
    ) AS revenue_percentage

FROM category_performance
ORDER BY gross_revenue DESC;
```

---

## Report 7: Three-Product Combinations

```sql
WITH transaction_products AS (
    SELECT DISTINCT
        txn_id,
        prod_id,
        product_name
    FROM monthly_sales_scope
),

product_combinations AS (
    SELECT
        p1.txn_id,

        p1.product_name AS product_1,
        p2.product_name AS product_2,
        p3.product_name AS product_3

    FROM transaction_products AS p1

    INNER JOIN transaction_products AS p2
        ON p1.txn_id = p2.txn_id
       AND p1.prod_id < p2.prod_id

    INNER JOIN transaction_products AS p3
        ON p2.txn_id = p3.txn_id
       AND p2.prod_id < p3.prod_id
)

SELECT
    product_1,
    product_2,
    product_3,
    COUNT(*) AS combination_count

FROM product_combinations

GROUP BY
    product_1,
    product_2,
    product_3

ORDER BY combination_count DESC
LIMIT 1;
```

---

## Scheduling Recommendation

In a production database, the reporting month could be calculated automatically:

```sql
DATE_TRUNC(
    'month',
    CURRENT_DATE
    - INTERVAL '1 month'
)::DATE
```

This would automatically select the previous calendar month whenever the report runs.

---

## Reporting Benefits

The monthly reporting structure:

- Changes the date in only one place
- Produces consistent definitions
- Reduces manual work
- Supports scheduled execution
- Makes January and February directly comparable
- Can later feed a Power BI or Tableau dashboard
