# Product Analysis

[← Back to Balanced Tree overview](./README.md)

## Overview

This section examines product, segment and category performance using gross revenue before discounts.

---

## 1. What were the top three products by gross revenue?

### Query

```sql
SELECT
    pd.product_name,
    SUM(s.qty * s.price) AS gross_revenue
FROM balanced_tree.sales AS s
INNER JOIN balanced_tree.product_details AS pd
    ON s.prod_id = pd.product_id
GROUP BY pd.product_name
ORDER BY gross_revenue DESC
LIMIT 3;
```

### Result

| Product | Gross revenue |
|---|---:|
| Blue Polo Shirt - Mens | $217,683 |
| Grey Fashion Jacket - Womens | $209,304 |
| White Tee Shirt - Mens | $152,000 |

### Insight

Blue Polo Shirt generated the highest gross revenue.

---

## 2. What were the total quantity, revenue and discount for each segment?

### Query

```sql
SELECT
    pd.segment_name,

    SUM(s.qty) AS total_quantity,

    SUM(
        s.qty * s.price
    ) AS gross_revenue,

    ROUND(
        SUM(
            s.qty
            * s.price
            * s.discount::NUMERIC
            / 100
        ),
        2
    ) AS discount_value

FROM balanced_tree.sales AS s
INNER JOIN balanced_tree.product_details AS pd
    ON s.prod_id = pd.product_id

GROUP BY pd.segment_name
ORDER BY gross_revenue DESC;
```

### Result

| Segment | Quantity | Gross revenue | Discount |
|---|---:|---:|---:|
| Shirt | 11,265 | $406,143 | $49,594.27 |
| Jacket | 11,385 | $366,983 | $44,277.46 |
| Socks | 11,217 | $307,977 | $37,013.44 |
| Jeans | 11,349 | $208,350 | $25,343.97 |

### Insight

Shirts generated the highest gross revenue despite Jackets recording a slightly larger sales quantity.

---

## 3. What was the top-selling product for each segment?

### Query

```sql
WITH product_quantity AS (
    SELECT
        pd.segment_name,
        pd.product_name,
        SUM(s.qty) AS total_quantity,

        DENSE_RANK() OVER (
            PARTITION BY pd.segment_name
            ORDER BY SUM(s.qty) DESC
        ) AS product_rank

    FROM balanced_tree.sales AS s
    INNER JOIN balanced_tree.product_details AS pd
        ON s.prod_id = pd.product_id

    GROUP BY
        pd.segment_name,
        pd.product_name
)

SELECT
    segment_name,
    product_name,
    total_quantity
FROM product_quantity
WHERE product_rank = 1
ORDER BY segment_name;
```

### Result

| Segment | Top product | Quantity |
|---|---|---:|
| Jacket | Grey Fashion Jacket - Womens | 3,876 |
| Jeans | Navy Oversized Jeans - Womens | 3,856 |
| Shirt | Blue Polo Shirt - Mens | 3,819 |
| Socks | Navy Solid Socks - Mens | 3,792 |

---

## 4. What were the total quantity, revenue and discount for each category?

### Query

```sql
SELECT
    pd.category_name,

    SUM(s.qty) AS total_quantity,

    SUM(
        s.qty * s.price
    ) AS gross_revenue,

    ROUND(
        SUM(
            s.qty
            * s.price
            * s.discount::NUMERIC
            / 100
        ),
        2
    ) AS discount_value

FROM balanced_tree.sales AS s
INNER JOIN balanced_tree.product_details AS pd
    ON s.prod_id = pd.product_id

GROUP BY pd.category_name
ORDER BY gross_revenue DESC;
```

### Result

| Category | Quantity | Gross revenue | Discount |
|---|---:|---:|---:|
| Mens | 22,482 | $714,120 | $86,607.71 |
| Womens | 22,734 | $575,333 | $69,621.43 |

### Insight

Womens products sold slightly more units, but Mens products generated significantly more revenue.

---

## 5. What was the top-selling product for each category?

### Query

```sql
WITH category_product_quantity AS (
    SELECT
        pd.category_name,
        pd.product_name,
        SUM(s.qty) AS total_quantity,

        DENSE_RANK() OVER (
            PARTITION BY pd.category_name
            ORDER BY SUM(s.qty) DESC
        ) AS product_rank

    FROM balanced_tree.sales AS s
    INNER JOIN balanced_tree.product_details AS pd
        ON s.prod_id = pd.product_id

    GROUP BY
        pd.category_name,
        pd.product_name
)

SELECT
    category_name,
    product_name,
    total_quantity
FROM category_product_quantity
WHERE product_rank = 1
ORDER BY category_name;
```

### Result

| Category | Product | Quantity |
|---|---|---:|
| Mens | Blue Polo Shirt - Mens | 3,819 |
| Womens | Grey Fashion Jacket - Womens | 3,876 |

---

## 6. What was the revenue split by product within each segment?

### Query

```sql
WITH product_revenue AS (
    SELECT
        pd.segment_name,
        pd.product_name,
        SUM(s.qty * s.price) AS gross_revenue

    FROM balanced_tree.sales AS s
    INNER JOIN balanced_tree.product_details AS pd
        ON s.prod_id = pd.product_id

    GROUP BY
        pd.segment_name,
        pd.product_name
)

SELECT
    segment_name,
    product_name,
    gross_revenue,

    ROUND(
        100.0
        * gross_revenue
        / SUM(gross_revenue) OVER (
            PARTITION BY segment_name
        ),
        2
    ) AS segment_revenue_percentage

FROM product_revenue
ORDER BY
    segment_name,
    segment_revenue_percentage DESC;
```

### Result

| Segment | Product | Revenue percentage |
|---|---|---:|
| Jacket | Grey Fashion Jacket - Womens | 57.03% |
| Jacket | Khaki Suit Jacket - Womens | 23.51% |
| Jacket | Indigo Rain Jacket - Womens | 19.45% |
| Jeans | Black Straight Jeans - Womens | 58.15% |
| Jeans | Navy Oversized Jeans - Womens | 24.06% |
| Jeans | Cream Relaxed Jeans - Womens | 17.79% |
| Shirt | Blue Polo Shirt - Mens | 53.60% |
| Shirt | White Tee Shirt - Mens | 37.43% |
| Shirt | Teal Button Up Shirt - Mens | 8.98% |
| Socks | Navy Solid Socks - Mens | 44.33% |
| Socks | Pink Fluro Polkadot Socks - Mens | 35.50% |
| Socks | White Striped Socks - Mens | 20.18% |

---

## 7. What was the revenue split by segment within each category?

### Query

```sql
WITH segment_revenue AS (
    SELECT
        pd.category_name,
        pd.segment_name,
        SUM(s.qty * s.price) AS gross_revenue

    FROM balanced_tree.sales AS s
    INNER JOIN balanced_tree.product_details AS pd
        ON s.prod_id = pd.product_id

    GROUP BY
        pd.category_name,
        pd.segment_name
)

SELECT
    category_name,
    segment_name,
    gross_revenue,

    ROUND(
        100.0
        * gross_revenue
        / SUM(gross_revenue) OVER (
            PARTITION BY category_name
        ),
        2
    ) AS category_revenue_percentage

FROM segment_revenue
ORDER BY
    category_name,
    category_revenue_percentage DESC;
```

### Result

| Category | Segment | Revenue percentage |
|---|---|---:|
| Mens | Shirt | 56.87% |
| Mens | Socks | 43.13% |
| Womens | Jacket | 63.79% |
| Womens | Jeans | 36.21% |

---

## 8. What was the total revenue split by category?

### Query

```sql
WITH category_revenue AS (
    SELECT
        pd.category_name,
        SUM(s.qty * s.price) AS gross_revenue

    FROM balanced_tree.sales AS s
    INNER JOIN balanced_tree.product_details AS pd
        ON s.prod_id = pd.product_id

    GROUP BY pd.category_name
)

SELECT
    category_name,
    gross_revenue,

    ROUND(
        100.0
        * gross_revenue
        / SUM(gross_revenue) OVER (),
        2
    ) AS total_revenue_percentage

FROM category_revenue
ORDER BY total_revenue_percentage DESC;
```

### Result

| Category | Revenue percentage |
|---|---:|
| Mens | 55.38% |
| Womens | 44.62% |

---

## 9. What was the transaction penetration for each product?

### Definition

```text
Transactions containing the product
÷
All transactions
× 100
```

### Query

```sql
WITH total_transactions AS (
    SELECT
        COUNT(DISTINCT txn_id) AS transaction_count
    FROM balanced_tree.sales
),

product_transactions AS (
    SELECT
        prod_id,
        COUNT(DISTINCT txn_id) AS product_transaction_count
    FROM balanced_tree.sales
    GROUP BY prod_id
)

SELECT
    pd.product_name,
    pt.product_transaction_count,

    ROUND(
        100.0
        * pt.product_transaction_count
        / tt.transaction_count,
        2
    ) AS penetration_percentage

FROM product_transactions AS pt
INNER JOIN balanced_tree.product_details AS pd
    ON pt.prod_id = pd.product_id
CROSS JOIN total_transactions AS tt

ORDER BY penetration_percentage DESC;
```

### Result

| Product | Transactions | Penetration |
|---|---:|---:|
| Navy Solid Socks - Mens | 1,281 | 51.24% |
| Grey Fashion Jacket - Womens | 1,275 | 51.00% |
| Navy Oversized Jeans - Womens | 1,274 | 50.96% |
| White Tee Shirt - Mens | 1,268 | 50.72% |
| Blue Polo Shirt - Mens | 1,268 | 50.72% |
| Pink Fluro Polkadot Socks - Mens | 1,258 | 50.32% |
| Indigo Rain Jacket - Womens | 1,250 | 50.00% |
| Khaki Suit Jacket - Womens | 1,247 | 49.88% |
| Black Straight Jeans - Womens | 1,246 | 49.84% |
| Cream Relaxed Jeans - Womens | 1,243 | 49.72% |
| White Striped Socks - Mens | 1,243 | 49.72% |
| Teal Button Up Shirt - Mens | 1,242 | 49.68% |

### Insight

Every product appeared in approximately half of all transactions.

---

## 10. What was the most common three-product combination?

### SQL approach

Create every unique three-product combination within each transaction, then count how frequently each combination occurred.

This approach includes transactions containing three or more products.

### Query

```sql
WITH transaction_products AS (
    SELECT DISTINCT
        txn_id,
        prod_id
    FROM balanced_tree.sales
),

product_combinations AS (
    SELECT
        p1.txn_id,
        p1.prod_id AS product_1,
        p2.prod_id AS product_2,
        p3.prod_id AS product_3

    FROM transaction_products AS p1

    INNER JOIN transaction_products AS p2
        ON p1.txn_id = p2.txn_id
       AND p1.prod_id < p2.prod_id

    INNER JOIN transaction_products AS p3
        ON p2.txn_id = p3.txn_id
       AND p2.prod_id < p3.prod_id
),

combination_counts AS (
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
)

SELECT
    pd1.product_name AS product_1,
    pd2.product_name AS product_2,
    pd3.product_name AS product_3,
    cc.combination_count

FROM combination_counts AS cc

INNER JOIN balanced_tree.product_details AS pd1
    ON cc.product_1 = pd1.product_id

INNER JOIN balanced_tree.product_details AS pd2
    ON cc.product_2 = pd2.product_id

INNER JOIN balanced_tree.product_details AS pd3
    ON cc.product_3 = pd3.product_id

ORDER BY cc.combination_count DESC
LIMIT 1;
```

### Result

Run the query in DB Fiddle and add the returned combination.

> Some published solutions only analyse transactions containing exactly three products. The query above analyses every possible three-product combination from transactions containing at least three products, which more closely follows the wording of the business question.

---

## Product Findings

- Blue Polo Shirt generated the highest gross revenue.
- Grey Fashion Jacket sold the largest number of units.
- Mens products generated the larger share of revenue.
- Shirts were the highest-revenue segment.
- Navy Solid Socks appeared in the largest percentage of transactions.
- Each segment depended heavily on its highest-revenue product.
