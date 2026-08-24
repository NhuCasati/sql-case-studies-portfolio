# Pricing and Ratings

[← Back to Pizza Runner overview](./README.md)

---

## 1. How much base revenue did successful deliveries generate?

### Pricing assumptions

- Meat Lovers: $12
- Vegetarian: $10
- No delivery fee
- No charges for changes
- Cancelled orders are excluded

### Query

```sql
SELECT
    SUM(
        CASE
            WHEN pn.pizza_name = 'Meatlovers'
                THEN 12
            WHEN pn.pizza_name = 'Vegetarian'
                THEN 10
            ELSE 0
        END
    ) AS total_revenue
FROM customer_orders_clean AS co
INNER JOIN runner_orders_clean AS ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names AS pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL;
```

### Result

| Revenue |
|---:|
| $138 |

---

## 2. What if every extra topping cost $1?

### Query

```sql
WITH delivered_pizzas AS (
    SELECT
        co.order_item_id,
        co.pizza_id,
        co.extras
    FROM customer_orders_clean AS co
    INNER JOIN runner_orders_clean AS ro
        ON co.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
),

pizza_charges AS (
    SELECT
        dp.order_item_id,
        CASE
            WHEN dp.pizza_id = 1 THEN 12
            WHEN dp.pizza_id = 2 THEN 10
        END AS base_price,
        COUNT(extra.extra_id) AS number_of_extras
    FROM delivered_pizzas AS dp
    LEFT JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
        dp.extras,
        '\s*,\s*'
    ) AS extra(extra_id)
        ON dp.extras IS NOT NULL
    GROUP BY
        dp.order_item_id,
        dp.pizza_id
)

SELECT
    SUM(base_price + number_of_extras) AS total_revenue
FROM pizza_charges;
```

### Result

| Revenue including extras |
|---:|
| $142 |

### Insight

Charging $1 for each extra topping would add $4 to total revenue.

---

## 3. Create a runner-ratings table

### Design

Each successful order receives one rating between 1 and 5.

### Query

```sql
DROP TABLE IF EXISTS pizza_runner.runner_ratings;

CREATE TABLE pizza_runner.runner_ratings (
    order_id INTEGER PRIMARY KEY,
    rating INTEGER NOT NULL
        CHECK (rating BETWEEN 1 AND 5)
);
```

### Sample ratings

```sql
INSERT INTO pizza_runner.runner_ratings (
    order_id,
    rating
)
VALUES
    (1, 5),
    (2, 4),
    (3, 5),
    (4, 4),
    (5, 5),
    (7, 3),
    (8, 4),
    (10, 5);
```

The ratings are sample values created for this analysis.

---

## 4. Create a combined delivery-performance table

### Query

```sql
WITH order_summary AS (
    SELECT
        order_id,
        MIN(customer_id) AS customer_id,
        MIN(order_time) AS order_time,
        COUNT(*) AS total_pizzas
    FROM customer_orders_clean
    GROUP BY order_id
)

SELECT
    os.customer_id,
    os.order_id,
    ro.runner_id,
    rr.rating,
    os.order_time,
    ro.pickup_time,
    ROUND(
        (
            EXTRACT(
                EPOCH FROM (
                    ro.pickup_time - os.order_time
                )
            ) / 60
        )::NUMERIC,
        2
    ) AS pickup_delay_minutes,
    ro.duration_minutes,
    ROUND(
        (
            ro.distance_km
            / (ro.duration_minutes / 60.0)
        )::NUMERIC,
        2
    ) AS average_speed_kmh,
    os.total_pizzas
FROM order_summary AS os
INNER JOIN runner_orders_clean AS ro
    ON os.order_id = ro.order_id
INNER JOIN pizza_runner.runner_ratings AS rr
    ON os.order_id = rr.order_id
WHERE ro.cancellation IS NULL
ORDER BY os.order_id;
```

### Result

| Customer | Order | Runner | Rating | Pickup delay | Duration | Speed | Pizzas |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 101 | 1 | 1 | 5 | 10.53 | 32 | 37.50 | 1 |
| 101 | 2 | 1 | 4 | 10.03 | 27 | 44.44 | 1 |
| 102 | 3 | 1 | 5 | 21.23 | 20 | 40.20 | 2 |
| 103 | 4 | 2 | 4 | 29.28 | 40 | 35.10 | 3 |
| 104 | 5 | 3 | 5 | 10.47 | 15 | 40.00 | 1 |
| 105 | 7 | 2 | 3 | 10.27 | 25 | 60.00 | 1 |
| 102 | 8 | 2 | 4 | 20.48 | 15 | 93.60 | 1 |
| 104 | 10 | 1 | 5 | 15.52 | 10 | 60.00 | 2 |

---

## 5. How much remained after paying runners?

### Assumptions

- Meat Lovers: $12
- Vegetarian: $10
- Extras are free
- Runners receive $0.30 per kilometre
- Cancelled orders produce no revenue or runner cost

### Query

```sql
WITH revenue AS (
    SELECT
        SUM(
            CASE
                WHEN co.pizza_id = 1 THEN 12
                WHEN co.pizza_id = 2 THEN 10
                ELSE 0
            END
        ) AS total_revenue
    FROM customer_orders_clean AS co
    INNER JOIN runner_orders_clean AS ro
        ON co.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
),

runner_costs AS (
    SELECT
        SUM(distance_km * 0.30) AS total_runner_cost
    FROM runner_orders_clean
    WHERE cancellation IS NULL
)

SELECT
    total_revenue,
    ROUND(
        total_runner_cost,
        2
    ) AS total_runner_cost,
    ROUND(
        total_revenue - total_runner_cost,
        2
    ) AS amount_remaining
FROM revenue
CROSS JOIN runner_costs;
```

### Result

| Revenue | Runner cost | Amount remaining |
|---:|---:|---:|
| $138.00 | $43.56 | $94.44 |

### Insight

After paying runners based on delivery distance, Pizza Runner retained $94.44 before considering ingredient and other operating costs.
