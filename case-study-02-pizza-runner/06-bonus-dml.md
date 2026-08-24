# Bonus DML Challenge

[← Back to Pizza Runner overview](./README.md)

## Business question

How would the existing design support a new Supreme pizza containing every available topping?

The current model can support the new pizza by adding:

1. A record to `pizza_names`
2. A corresponding recipe to `pizza_recipes`

---

## 1. Add the Supreme pizza

```sql
INSERT INTO pizza_runner.pizza_names (
    pizza_id,
    pizza_name
)
SELECT
    3,
    'Supreme'
WHERE NOT EXISTS (
    SELECT 1
    FROM pizza_runner.pizza_names
    WHERE pizza_id = 3
);
```

---

## 2. Add every topping to its recipe

```sql
INSERT INTO pizza_runner.pizza_recipes (
    pizza_id,
    toppings
)
SELECT
    3,
    STRING_AGG(
        topping_id::TEXT,
        ', '
        ORDER BY topping_id
    )
FROM pizza_runner.pizza_toppings
WHERE NOT EXISTS (
    SELECT 1
    FROM pizza_runner.pizza_recipes
    WHERE pizza_id = 3
);
```

---

## 3. Validate the new pizza

```sql
SELECT
    pn.pizza_id,
    pn.pizza_name,
    pr.toppings
FROM pizza_runner.pizza_names AS pn
INNER JOIN pizza_runner.pizza_recipes AS pr
    ON pn.pizza_id = pr.pizza_id
WHERE pn.pizza_id = 3;
```

### Expected result

| Pizza ID | Pizza | Toppings |
|---:|---|---|
| 3 | Supreme | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12 |

---

## Design Observation

The new pizza can be inserted without changing the existing tables.

However, storing topping identifiers as comma-separated text makes ingredient analysis more complicated. A more scalable design would use a bridge table:

```sql
CREATE TABLE pizza_runner.pizza_recipe_toppings (
    pizza_id INTEGER,
    topping_id INTEGER,
    PRIMARY KEY (
        pizza_id,
        topping_id
    )
);
```

In a normalised design, every pizza and topping combination would be stored as a separate row. This would simplify joins, filtering and recipe maintenance.
