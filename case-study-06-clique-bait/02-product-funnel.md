# Product Funnel Analysis

[← Back to Clique Bait overview](./README.md)

## Objective

Create a product-level funnel containing:

- Product views
- Cart additions
- Abandoned cart additions
- Purchases

A cart addition is treated as purchased when the corresponding visit contains a purchase event.

---

## 1. Create the product-funnel table

```sql
DROP TABLE IF EXISTS product_funnel;

CREATE TEMP TABLE product_funnel AS

WITH visit_purchase AS (
    SELECT
        visit_id,

        MAX(
            CASE
                WHEN event_type = 3 THEN 1
                ELSE 0
            END
        ) AS purchased

    FROM clique_bait.events
    GROUP BY visit_id
),

product_visit_events AS (
    SELECT
        e.visit_id,
        ph.product_id,
        ph.page_name AS product_name,
        ph.product_category,

        COUNT(*) FILTER (
            WHERE e.event_type = 1
        ) AS views,

        COUNT(*) FILTER (
            WHERE e.event_type = 2
        ) AS cart_adds

    FROM clique_bait.events AS e
    INNER JOIN clique_bait.page_hierarchy AS ph
        ON e.page_id = ph.page_id

    WHERE ph.product_id IS NOT NULL

    GROUP BY
        e.visit_id,
        ph.product_id,
        ph.page_name,
        ph.product_category
)

SELECT
    pve.product_id,
    pve.product_name,
    pve.product_category,

    SUM(pve.views) AS views,

    SUM(pve.cart_adds) AS cart_adds,

    SUM(
        CASE
            WHEN vp.purchased = 0
                THEN pve.cart_adds
            ELSE 0
        END
    ) AS abandoned,

    SUM(
        CASE
            WHEN vp.purchased = 1
                THEN pve.cart_adds
            ELSE 0
        END
    ) AS purchased

FROM product_visit_events AS pve
INNER JOIN visit_purchase AS vp
    ON pve.visit_id = vp.visit_id

GROUP BY
    pve.product_id,
    pve.product_name,
    pve.product_category

ORDER BY pve.product_id;
```

### Validate the table

```sql
SELECT *
FROM product_funnel
ORDER BY product_id;
```

### Result

| Product | Category | Views | Cart adds | Abandoned | Purchased |
|---|---|---:|---:|---:|---:|
| Salmon | Fish | 1,559 | 938 | 227 | 711 |
| Kingfish | Fish | 1,559 | 920 | 213 | 707 |
| Tuna | Fish | 1,515 | 931 | 234 | 697 |
| Russian Caviar | Luxury | 1,563 | 946 | 249 | 697 |
| Black Truffle | Luxury | 1,469 | 924 | 217 | 707 |
| Abalone | Shellfish | 1,525 | 932 | 233 | 699 |
| Lobster | Shellfish | 1,547 | 968 | 214 | 754 |
| Crab | Shellfish | 1,564 | 949 | 230 | 719 |
| Oyster | Shellfish | 1,568 | 943 | 217 | 726 |

---

## 2. Create the category-funnel table

```sql
DROP TABLE IF EXISTS category_funnel;

CREATE TEMP TABLE category_funnel AS

SELECT
    product_category,
    SUM(views) AS views,
    SUM(cart_adds) AS cart_adds,
    SUM(abandoned) AS abandoned,
    SUM(purchased) AS purchased
FROM product_funnel
GROUP BY product_category
ORDER BY product_category;
```

### Validate the table

```sql
SELECT *
FROM category_funnel;
```

### Result

| Category | Views | Cart adds | Abandoned | Purchased |
|---|---:|---:|---:|---:|
| Fish | 4,633 | 2,789 | 674 | 2,115 |
| Luxury | 3,032 | 1,870 | 466 | 1,404 |
| Shellfish | 6,204 | 3,792 | 894 | 2,898 |

---

## 3. Which product had the most views?

```sql
SELECT
    product_name,
    views
FROM product_funnel
ORDER BY views DESC
LIMIT 1;
```

### Result

| Product | Views |
|---|---:|
| Oyster | 1,568 |

---

## 4. Which product had the most cart additions?

```sql
SELECT
    product_name,
    cart_adds
FROM product_funnel
ORDER BY cart_adds DESC
LIMIT 1;
```

### Result

| Product | Cart additions |
|---|---:|
| Lobster | 968 |

---

## 5. Which product had the most purchases?

```sql
SELECT
    product_name,
    purchased
FROM product_funnel
ORDER BY purchased DESC
LIMIT 1;
```

### Result

| Product | Purchases |
|---|---:|
| Lobster | 754 |

---

## 6. Which product was most frequently abandoned?

```sql
SELECT
    product_name,
    abandoned
FROM product_funnel
ORDER BY abandoned DESC
LIMIT 1;
```

### Result

| Product | Abandoned |
|---|---:|
| Russian Caviar | 249 |

### Insight

Russian Caviar received strong interest but also had the highest number of abandoned cart additions.

Price information would be needed to determine whether cost contributed to this behaviour.

---

## 7. Which product had the highest view-to-purchase rate?

```sql
SELECT
    product_name,

    ROUND(
        100.0
        * purchased
        / NULLIF(views, 0),
        2
    ) AS view_to_purchase_percentage

FROM product_funnel
ORDER BY view_to_purchase_percentage DESC
LIMIT 1;
```

### Result

| Product | View-to-purchase rate |
|---|---:|
| Lobster | 48.74% |

---

## 8. What was the average view-to-cart conversion rate?

```sql
SELECT
    ROUND(
        AVG(
            100.0
            * cart_adds
            / NULLIF(views, 0)
        ),
        2
    ) AS average_view_to_cart_rate
FROM product_funnel;
```

### Result

| Conversion rate |
|---:|
| 60.95% |

---

## 9. What was the average cart-to-purchase conversion rate?

```sql
SELECT
    ROUND(
        AVG(
            100.0
            * purchased
            / NULLIF(cart_adds, 0)
        ),
        2
    ) AS average_cart_to_purchase_rate
FROM product_funnel;
```

### Result

| Conversion rate |
|---:|
| 75.93% |

---

## Product Funnel Summary

- Oyster was the most viewed product.
- Lobster generated the most cart additions.
- Lobster also generated the most purchases.
- Russian Caviar had the highest abandoned-cart count.
- Lobster had the strongest view-to-purchase conversion.
- Shellfish was the highest-performing product category overall.
