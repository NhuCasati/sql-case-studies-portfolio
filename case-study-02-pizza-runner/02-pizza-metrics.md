# Pizza Metrics

[← Back to Pizza Runner overview](./README.md)

---

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

Customers ordered 14 pizzas across the period covered by the dataset.

---

## 2. How many unique customer orders were made?

### SQL approach

Count the distinct order identifiers.

### Query

```sql
SELECT
    COUNT(DISTINCT order_id) AS unique_orders
FROM customer_orders_clean;
```

### Result

| Unique orders |
|---:|
| 10 |

### Insight

The 14 pizzas belonged to 10 separate customer orders.

---

## 3. How many successful orders were delivered by each runner?

### SQL approach

A delivery is considered successful when `cancellation` is `NULL`.

### Query

```sql
SELECT
    runner_id,
    COUNT(*) AS successful_orders
FROM runner_orders_clean
WHERE cancellation IS NULL
GROUP BY runner_id
ORDER BY runner_id;
```

### Result

| Runner | Successful orders |
|---:|---:|
| 1 | 4 |
| 2 | 3 |
| 3 | 1 |

### Insight

Runner 1 completed the largest number of successful deliveries.

---

## 4. How many of each pizza type were delivered?

### SQL approach

Join successfully delivered orders to the pizza items and pizza names.

### Query

```sql
SELECT
    pn.pizza_name,
    COUNT(*) AS delivered_pizzas
FROM customer_orders_clean AS co
INNER JOIN runner_orders_clean AS ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names AS pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL
GROUP BY pn.pizza_name
ORDER BY delivered_pizzas DESC;
```

### Result

| Pizza | Delivered |
|---|---:|
| Meatlovers | 9 |
| Vegetarian | 3 |

### Insight

Meat Lovers represented 75% of all delivered pizzas.

---

## 5. How many Vegetarian and Meat Lovers pizzas did each customer order?

### Query

```sql
SELECT
    co.customer_id,
    pn.pizza_name,
    COUNT(*) AS pizzas_ordered
FROM customer_orders_clean AS co
INNER JOIN pizza_runner.pizza_names AS pn
    ON co.pizza_id = pn.pizza_id
GROUP BY
    co.customer_id,
    pn.pizza_name
ORDER BY
    co.customer_id,
    pn.pizza_name;
```

### Result

| Customer | Pizza | Ordered |
|---:|---|---:|
| 101 | Meatlovers | 2 |
| 101 | Vegetarian | 1 |
| 102 | Meatlovers | 2 |
| 102 | Vegetarian | 1 |
| 103 | Meatlovers | 3 |
| 103 | Vegetarian | 1 |
| 104 | Meatlovers | 3 |
| 105 | Vegetarian | 1 |

### Insight

Meat Lovers was the dominant choice for most customers.

---

## 6. What was the maximum number of pizzas delivered in one order?

### Query

```sql
WITH delivered_order_sizes AS (
    SELECT
        co.order_id,
        COUNT(*) AS pizza_count
    FROM customer_orders_clean AS co
    INNER JOIN runner_orders_clean AS ro
        ON co.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
    GROUP BY co.order_id
)

SELECT
    MAX(pizza_count) AS maximum_pizzas
FROM delivered_order_sizes;
```

### Result

| Maximum pizzas |
|---:|
| 3 |

### Insight

The largest successful order contained three pizzas.

---

## 7. How many delivered pizzas had changes?

### SQL approach

A pizza is considered changed when it contains at least one exclusion or extra.

### Query

```sql
SELECT
    co.customer_id,
    COUNT(*) FILTER (
        WHERE co.exclusions IS NOT NULL
           OR co.extras IS NOT NULL
    ) AS changed_pizzas,
    COUNT(*) FILTER (
        WHERE co.exclusions IS NULL
          AND co.extras IS NULL
    ) AS unchanged_pizzas
FROM customer_orders_clean AS co
INNER JOIN runner_orders_clean AS ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
GROUP BY co.customer_id
ORDER BY co.customer_id;
```

### Result

| Customer | Changed | Unchanged |
|---:|---:|---:|
| 101 | 0 | 2 |
| 102 | 0 | 3 |
| 103 | 3 | 0 |
| 104 | 2 | 1 |
| 105 | 1 | 0 |

### Insight

Customers 103 and 105 customised every delivered pizza they ordered.

---

## 8. How many delivered pizzas had both exclusions and extras?

### Query

```sql
SELECT
    COUNT(*) AS pizzas_with_exclusions_and_extras
FROM customer_orders_clean AS co
INNER JOIN runner_orders_clean AS ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
  AND co.exclusions IS NOT NULL
  AND co.extras IS NOT NULL;
```

### Result

| Pizzas |
|---:|
| 1 |

### Insight

Only one successfully delivered pizza contained both exclusions and extras.

---

## 9. What was the pizza-order volume by hour?

### Query

```sql
SELECT
    EXTRACT(HOUR FROM order_time)::INTEGER AS order_hour,
    COUNT(*) AS pizzas_ordered
FROM customer_orders_clean
GROUP BY order_hour
ORDER BY order_hour;
```

### Result

| Hour | Pizzas |
|---:|---:|
| 11 | 1 |
| 13 | 3 |
| 18 | 3 |
| 19 | 1 |
| 21 | 3 |
| 23 | 3 |

### Insight

The busiest recorded hours were 13:00, 18:00, 21:00 and 23:00.

---

## 10. What was the order volume by weekday?

### Query

```sql
SELECT
    TO_CHAR(order_time, 'FMDay') AS weekday,
    COUNT(*) AS pizza_order_rows
FROM customer_orders_clean
GROUP BY
    EXTRACT(ISODOW FROM order_time),
    TO_CHAR(order_time, 'FMDay')
ORDER BY EXTRACT(ISODOW FROM order_time);
```

### Result

| Weekday | Pizza-order rows |
|---|---:|
| Wednesday | 5 |
| Thursday | 3 |
| Friday | 1 |
| Saturday | 5 |

### Insight

Wednesday and Saturday recorded the highest pizza volume.
