# High-Level Sales Analysis

[← Back to Balanced Tree overview](./README.md)

## Overview

This section provides an overall view of sales quantity, gross revenue and discount activity.

---

## 1. What was the total quantity sold?

### Query

```sql
SELECT
    SUM(qty) AS total_quantity_sold
FROM balanced_tree.sales;
```

### Result

| Total quantity |
|---:|
| 45,216 |

### Insight

Balanced Tree sold 45,216 individual units during the period covered by the dataset.

---

## 2. What was the total generated revenue before discounts?

### SQL approach

Gross revenue is calculated by multiplying the quantity by the listed unit price for every sales row.

### Query

```sql
SELECT
    SUM(qty * price) AS gross_revenue
FROM balanced_tree.sales;
```

### Result

| Gross revenue |
|---:|
| $1,289,453 |

### Insight

Balanced Tree generated approximately $1.29 million in gross revenue before applying discounts.

---

## 3. What was the total discount amount?

### Query

```sql
SELECT
    ROUND(
        SUM(
            qty
            * price
            * discount::NUMERIC
            / 100
        ),
        2
    ) AS total_discount
FROM balanced_tree.sales;
```

### Result

| Total discount |
|---:|
| $156,229.14 |

### Insight

Discounts reduced revenue by approximately $156,229.

---

## Additional Metric: Net Revenue

```sql
SELECT
    ROUND(
        SUM(
            qty
            * price
            * (
                1 - discount::NUMERIC / 100
            )
        ),
        2
    ) AS net_revenue
FROM balanced_tree.sales;
```

### Result

| Net revenue |
|---:|
| $1,133,223.86 |

### Insight

After discounts, Balanced Tree retained approximately $1.13 million in sales revenue.
