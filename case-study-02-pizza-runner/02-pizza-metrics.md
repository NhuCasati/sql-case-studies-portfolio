# Pizza Metrics

[← Back to Pizza Runner overview](./README.md)

## 1. How many pizzas were ordered?

### SQL approach

Each row in `customer_orders_clean` represents one pizza.

### Query

```sql
SELECT
    COUNT(*) AS total_pizzas_ordered
FROM customer_orders_clean;
```

### Result

| Total pizzas ordered |
|---:|
| 14 |

### Insight

A total of 14 pizzas were ordered during the period covered by the dataset.

---

## 2. How many unique customer orders were made?

### SQL approach

Count the distinct `order_id` values.

### Query

```sql
SELECT
    COUNT(DISTINCT order_id) AS unique_orders
FROM customer_orders_clean;
```

### Result

Add the result after running the query.

### Insight

Add your interpretation here.
