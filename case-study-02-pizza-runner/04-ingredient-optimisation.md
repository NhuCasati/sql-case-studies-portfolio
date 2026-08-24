# Ingredient Optimisation

[← Back to Pizza Runner overview](./README.md)


---

## 1. What are the standard ingredients for each pizza?

### Query

```sql
SELECT
    pn.pizza_name,
    STRING_AGG(
        pt.topping_name,
        ', '
        ORDER BY pt.topping_name
    ) AS standard_ingredients
FROM pizza_runner.pizza_recipes AS pr
INNER JOIN pizza_runner.pizza_names AS pn
    ON pr.pizza_id = pn.pizza_id
CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
    pr.toppings,
    '\s*,\s*'
) AS x(topping_id)
INNER JOIN pizza_runner.pizza_toppings AS pt
    ON pt.topping_id = x.topping_id::INTEGER
GROUP BY pn.pizza_name
ORDER BY pn.pizza_name;
```

### Result

| Pizza | Standard ingredients |
|---|---|
| Meatlovers | BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| Vegetarian | Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes |

---

## 2. What was the most common extra?

### Query

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_added
FROM customer_orders_clean AS co
CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
    co.extras,
    '\s*,\s*'
) AS x(topping_id)
INNER JOIN pizza_runner.pizza_toppings AS pt
    ON pt.topping_id = x.topping_id::INTEGER
WHERE co.extras IS NOT NULL
GROUP BY pt.topping_name
ORDER BY
    times_added DESC,
    pt.topping_name
LIMIT 1;
```

### Result

| Extra | Times added |
|---|---:|
| Bacon | 4 |

### Insight

Bacon was the most frequently requested extra topping.

---

## 3. What was the most common exclusion?

### Query

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_excluded
FROM customer_orders_clean AS co
CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
    co.exclusions,
    '\s*,\s*'
) AS x(topping_id)
INNER JOIN pizza_runner.pizza_toppings AS pt
    ON pt.topping_id = x.topping_id::INTEGER
WHERE co.exclusions IS NOT NULL
GROUP BY pt.topping_name
ORDER BY
    times_excluded DESC,
    pt.topping_name
LIMIT 1;
```

### Result

| Exclusion | Times excluded |
|---|---:|
| Cheese | 4 |

### Insight

Cheese was the ingredient customers removed most frequently.

---

## 4. Generate a readable description for every pizza item

### Query

```sql
SELECT
    co.order_item_id,
    co.order_id,
    co.customer_id,
    CONCAT(
        pn.pizza_name,
        CASE
            WHEN co.exclusions IS NOT NULL
                THEN ' - Exclude ' || (
                    SELECT STRING_AGG(
                        pt.topping_name,
                        ', '
                        ORDER BY x.position
                    )
                    FROM REGEXP_SPLIT_TO_TABLE(
                        co.exclusions,
                        '\s*,\s*'
                    ) WITH ORDINALITY
                        AS x(topping_id, position)
                    INNER JOIN pizza_runner.pizza_toppings AS pt
                        ON pt.topping_id =
                           x.topping_id::INTEGER
                )
            ELSE ''
        END,
        CASE
            WHEN co.extras IS NOT NULL
                THEN ' - Extra ' || (
                    SELECT STRING_AGG(
                        pt.topping_name,
                        ', '
                        ORDER BY x.position
                    )
                    FROM REGEXP_SPLIT_TO_TABLE(
                        co.extras,
                        '\s*,\s*'
                    ) WITH ORDINALITY
                        AS x(topping_id, position)
                    INNER JOIN pizza_runner.pizza_toppings AS pt
                        ON pt.topping_id =
                           x.topping_id::INTEGER
                )
            ELSE ''
        END
    ) AS order_description
FROM customer_orders_clean AS co
INNER JOIN pizza_runner.pizza_names AS pn
    ON co.pizza_id = pn.pizza_id
ORDER BY co.order_item_id;
```

### Result

| Order | Description |
|---:|---|
| 1 | Meatlovers |
| 2 | Meatlovers |
| 3 | Meatlovers |
| 3 | Vegetarian |
| 4 | Meatlovers - Exclude Cheese |
| 4 | Meatlovers - Exclude Cheese |
| 4 | Vegetarian - Exclude Cheese |
| 5 | Meatlovers - Extra Bacon |
| 6 | Vegetarian |
| 7 | Vegetarian - Extra Bacon |
| 8 | Meatlovers |
| 9 | Meatlovers - Exclude Cheese - Extra Bacon, Chicken |
| 10 | Meatlovers |
| 10 | Meatlovers - Exclude BBQ Sauce, Mushrooms - Extra Bacon, Cheese |

---

## 5. Generate the complete ingredient list for each pizza

### Query

```sql
WITH base_ingredients AS (
    SELECT
        co.order_item_id,
        co.order_id,
        co.customer_id,
        pn.pizza_name,
        pt.topping_id,
        pt.topping_name
    FROM customer_orders_clean AS co
    INNER JOIN pizza_runner.pizza_names AS pn
        ON co.pizza_id = pn.pizza_id
    INNER JOIN pizza_runner.pizza_recipes AS pr
        ON co.pizza_id = pr.pizza_id
    CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
        pr.toppings,
        '\s*,\s*'
    ) AS recipe(topping_id)
    INNER JOIN pizza_runner.pizza_toppings AS pt
        ON pt.topping_id =
           recipe.topping_id::INTEGER
    WHERE NOT EXISTS (
        SELECT 1
        FROM REGEXP_SPLIT_TO_TABLE(
            COALESCE(co.exclusions, ''),
            '\s*,\s*'
        ) AS excluded(topping_id)
        WHERE NULLIF(
            excluded.topping_id,
            ''
        )::INTEGER = pt.topping_id
    )
),

extra_ingredients AS (
    SELECT
        co.order_item_id,
        co.order_id,
        co.customer_id,
        pn.pizza_name,
        pt.topping_id,
        pt.topping_name
    FROM customer_orders_clean AS co
    INNER JOIN pizza_runner.pizza_names AS pn
        ON co.pizza_id = pn.pizza_id
    CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
        co.extras,
        '\s*,\s*'
    ) AS extra(topping_id)
    INNER JOIN pizza_runner.pizza_toppings AS pt
        ON pt.topping_id =
           extra.topping_id::INTEGER
    WHERE co.extras IS NOT NULL
),

all_ingredients AS (
    SELECT * FROM base_ingredients

    UNION ALL

    SELECT * FROM extra_ingredients
),

ingredient_counts AS (
    SELECT
        order_item_id,
        order_id,
        customer_id,
        pizza_name,
        topping_name,
        COUNT(*) AS quantity
    FROM all_ingredients
    GROUP BY
        order_item_id,
        order_id,
        customer_id,
        pizza_name,
        topping_name
)

SELECT
    order_item_id,
    order_id,
    customer_id,
    pizza_name || ': ' ||
    STRING_AGG(
        CASE
            WHEN quantity > 1
                THEN quantity || 'x' || topping_name
            ELSE topping_name
        END,
        ', '
        ORDER BY topping_name
    ) AS ingredient_list
FROM ingredient_counts
GROUP BY
    order_item_id,
    order_id,
    customer_id,
    pizza_name
ORDER BY order_item_id;
```

### Example results

| Order | Ingredient list |
|---:|---|
| 1 | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| 4 | Meatlovers: BBQ Sauce, Bacon, Beef, Chicken, Mushrooms, Pepperoni, Salami |
| 5 | Meatlovers: BBQ Sauce, 2xBacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| 9 | Meatlovers: BBQ Sauce, 2xBacon, Beef, 2xChicken, Mushrooms, Pepperoni, Salami |
| 10 | Meatlovers: 2xBacon, Beef, 2xCheese, Chicken, Pepperoni, Salami |

---

## 6. What was the total ingredient usage for delivered pizzas?

### Query

```sql
WITH delivered_items AS (
    SELECT co.*
    FROM customer_orders_clean AS co
    INNER JOIN runner_orders_clean AS ro
        ON co.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
),

base_ingredients AS (
    SELECT
        di.order_item_id,
        pt.topping_id,
        pt.topping_name
    FROM delivered_items AS di
    INNER JOIN pizza_runner.pizza_recipes AS pr
        ON di.pizza_id = pr.pizza_id
    CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
        pr.toppings,
        '\s*,\s*'
    ) AS recipe(topping_id)
    INNER JOIN pizza_runner.pizza_toppings AS pt
        ON pt.topping_id =
           recipe.topping_id::INTEGER
    WHERE NOT EXISTS (
        SELECT 1
        FROM REGEXP_SPLIT_TO_TABLE(
            COALESCE(di.exclusions, ''),
            '\s*,\s*'
        ) AS excluded(topping_id)
        WHERE NULLIF(
            excluded.topping_id,
            ''
        )::INTEGER = pt.topping_id
    )
),

extra_ingredients AS (
    SELECT
        di.order_item_id,
        pt.topping_id,
        pt.topping_name
    FROM delivered_items AS di
    CROSS JOIN LATERAL REGEXP_SPLIT_TO_TABLE(
        di.extras,
        '\s*,\s*'
    ) AS extra(topping_id)
    INNER JOIN pizza_runner.pizza_toppings AS pt
        ON pt.topping_id =
           extra.topping_id::INTEGER
    WHERE di.extras IS NOT NULL
),

all_ingredients AS (
    SELECT * FROM base_ingredients

    UNION ALL

    SELECT * FROM extra_ingredients
)

SELECT
    topping_name,
    COUNT(*) AS total_quantity
FROM all_ingredients
GROUP BY topping_name
ORDER BY
    total_quantity DESC,
    topping_name;
```

### Result

| Ingredient | Quantity |
|---|---:|
| Bacon | 12 |
| Mushrooms | 11 |
| Cheese | 10 |
| Beef | 9 |
| Chicken | 9 |
| Pepperoni | 9 |
| Salami | 9 |
| BBQ Sauce | 8 |
| Onions | 3 |
| Peppers | 3 |
| Tomato Sauce | 3 |
| Tomatoes | 3 |

### Insight

Bacon was the most heavily used ingredient across successful deliveries.
