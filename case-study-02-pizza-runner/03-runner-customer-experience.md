# Runner and Customer Experience

[← Back to Pizza Runner overview](./README.md)


---

## 1. How many runners joined during each weekly period?

### Query

```sql
SELECT
    DATE '2021-01-01'
    + (
        (
            registration_date - DATE '2021-01-01'
        ) / 7
    ) * 7 AS week_start,
    COUNT(*) AS runner_signups
FROM pizza_runner.runners
GROUP BY week_start
ORDER BY week_start;
```

### Result

| Week starting | Signups |
|---|---:|
| 2021-01-01 | 2 |
| 2021-01-08 | 1 |
| 2021-01-15 | 1 |

### Insight

Half of the runners joined during the first week.

---

## 2. How long did each runner take to collect an order?

### Query

```sql
WITH order_times AS (
    SELECT
        order_id,
        MIN(order_time) AS order_time
    FROM customer_orders_clean
    GROUP BY order_id
)

SELECT
    ro.runner_id,
    ROUND(
        AVG(
            EXTRACT(
                EPOCH FROM (
                    ro.pickup_time - ot.order_time
                )
            ) / 60
        )::NUMERIC,
        2
    ) AS average_pickup_minutes
FROM runner_orders_clean AS ro
INNER JOIN order_times AS ot
    ON ro.order_id = ot.order_id
WHERE ro.cancellation IS NULL
GROUP BY ro.runner_id
ORDER BY ro.runner_id;
```

### Result

| Runner | Average pickup minutes |
|---:|---:|
| 1 | 14.33 |
| 2 | 20.01 |
| 3 | 10.47 |

### Insight

Runner 3 had the shortest average collection time, although the runner completed only one successful delivery.

---

## 3. Does pizza quantity affect preparation time?

### Query

```sql
WITH order_summary AS (
    SELECT
        order_id,
        MIN(order_time) AS order_time,
        COUNT(*) AS pizza_count
    FROM customer_orders_clean
    GROUP BY order_id
),

preparation_times AS (
    SELECT
        os.order_id,
        os.pizza_count,
        EXTRACT(
            EPOCH FROM (
                ro.pickup_time - os.order_time
            )
        ) / 60 AS preparation_minutes
    FROM order_summary AS os
    INNER JOIN runner_orders_clean AS ro
        ON os.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
)

SELECT
    pizza_count,
    ROUND(
        AVG(preparation_minutes)::NUMERIC,
        2
    ) AS average_preparation_minutes
FROM preparation_times
GROUP BY pizza_count
ORDER BY pizza_count;
```

### Result

| Pizzas | Average preparation minutes |
|---:|---:|
| 1 | 12.36 |
| 2 | 18.38 |
| 3 | 29.28 |

### Insight

Preparation time increased as the number of pizzas in an order increased.

---

## 4. What was the average delivery distance for each customer?

### Query

```sql
WITH customer_orders AS (
    SELECT DISTINCT
        order_id,
        customer_id
    FROM customer_orders_clean
)

SELECT
    co.customer_id,
    ROUND(
        AVG(ro.distance_km),
        2
    ) AS average_distance_km
FROM customer_orders AS co
INNER JOIN runner_orders_clean AS ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
GROUP BY co.customer_id
ORDER BY co.customer_id;
```

### Result

| Customer | Average distance |
|---:|---:|
| 101 | 20.00 |
| 102 | 18.40 |
| 103 | 23.40 |
| 104 | 10.00 |
| 105 | 25.00 |

### Insight

Customer 105 lived furthest from Pizza Runner, while Customer 104 had the shortest average delivery distance.

---

## 5. What was the difference between the longest and shortest delivery times?

### Query

```sql
SELECT
    MAX(duration_minutes)
    - MIN(duration_minutes) AS duration_difference_minutes
FROM runner_orders_clean
WHERE cancellation IS NULL;
```

### Result

| Difference |
|---:|
| 30 minutes |

### Insight

The delivery durations varied by 30 minutes between the shortest and longest deliveries.

---

## 6. What was the speed of each delivery?

### Query

```sql
SELECT
    runner_id,
    order_id,
    ROUND(
        (
            distance_km
            / (duration_minutes / 60.0)
        )::NUMERIC,
        2
    ) AS average_speed_kmh
FROM runner_orders_clean
WHERE cancellation IS NULL
ORDER BY
    runner_id,
    order_id;
```

### Result

| Runner | Order | Speed (km/h) |
|---:|---:|---:|
| 1 | 1 | 37.50 |
| 1 | 2 | 44.44 |
| 1 | 3 | 40.20 |
| 1 | 10 | 60.00 |
| 2 | 4 | 35.10 |
| 2 | 7 | 60.00 |
| 2 | 8 | 93.60 |
| 3 | 5 | 40.00 |

### Insight

The calculated speeds vary considerably. Order 8 produced an unusually high average speed of 93.60 km/h, which may indicate inaccurate distance or duration data.

---

## 7. What was each runner's successful-delivery percentage?

### Query

```sql
SELECT
    runner_id,
    ROUND(
        (
            100.0
            * COUNT(*) FILTER (
                WHERE cancellation IS NULL
            )
            / COUNT(*)
        )::NUMERIC,
        2
    ) AS successful_delivery_percentage
FROM runner_orders_clean
GROUP BY runner_id
ORDER BY runner_id;
```

### Result

| Runner | Successful deliveries |
|---:|---:|
| 1 | 100.00% |
| 2 | 75.00% |
| 3 | 50.00% |

### Insight

Runner 1 successfully completed every assigned delivery. Runner 4 had not yet received an order.
