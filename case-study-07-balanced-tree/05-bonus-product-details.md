# Bonus: Rebuild the Product Details Table

[← Back to Balanced Tree overview](./README.md)

## Objective

Rebuild `product_details` using only:

- `product_hierarchy`
- `product_prices`

The hierarchy contains three levels:

```text
Category → Segment → Style
```

A recursive CTE is used to pass category and segment information down to each style.

---

## Query

```sql
WITH RECURSIVE hierarchy AS (
    SELECT
        ph.id,
        ph.parent_id,
        ph.level_text,
        ph.level_name,

        ph.id AS category_id,
        ph.level_text AS category_name,

        NULL::INTEGER AS segment_id,
        NULL::VARCHAR AS segment_name

    FROM balanced_tree.product_hierarchy AS ph

    WHERE ph.parent_id IS NULL

    UNION ALL

    SELECT
        child.id,
        child.parent_id,
        child.level_text,
        child.level_name,

        parent.category_id,
        parent.category_name,

        CASE
            WHEN child.level_name = 'Segment'
                THEN child.id
            ELSE parent.segment_id
        END AS segment_id,

        CASE
            WHEN child.level_name = 'Segment'
                THEN child.level_text
            ELSE parent.segment_name
        END AS segment_name

    FROM balanced_tree.product_hierarchy AS child

    INNER JOIN hierarchy AS parent
        ON child.parent_id = parent.id
)

SELECT
    pp.product_id,
    pp.price,

    h.level_text
        || ' '
        || h.segment_name
        || ' - '
        || h.category_name AS product_name,

    h.category_id,
    h.segment_id,
    h.id AS style_id,

    h.category_name,
    h.segment_name,
    h.level_text AS style_name

FROM hierarchy AS h

INNER JOIN balanced_tree.product_prices AS pp
    ON h.id = pp.id

WHERE h.level_name = 'Style'

ORDER BY h.id;
```

---

## Expected Structure

| Column |
|---|
| `product_id` |
| `price` |
| `product_name` |
| `category_id` |
| `segment_id` |
| `style_id` |
| `category_name` |
| `segment_name` |
| `style_name` |

### Example

| Product ID | Product name | Category | Segment | Style |
|---|---|---|---|---|
| `c4a632` | Navy Oversized Jeans - Womens | Womens | Jeans | Navy Oversized |
| `e83aa3` | Black Straight Jeans - Womens | Womens | Jeans | Black Straight |
| `5d267b` | White Tee Shirt - Mens | Mens | Shirt | White Tee |

---

## Validate Against the Existing Table

Create the rebuilt table:

```sql
DROP TABLE IF EXISTS product_details_rebuilt;

CREATE TEMP TABLE product_details_rebuilt AS

WITH RECURSIVE hierarchy AS (
    SELECT
        ph.id,
        ph.parent_id,
        ph.level_text,
        ph.level_name,
        ph.id AS category_id,
        ph.level_text AS category_name,
        NULL::INTEGER AS segment_id,
        NULL::VARCHAR AS segment_name

    FROM balanced_tree.product_hierarchy AS ph
    WHERE ph.parent_id IS NULL

    UNION ALL

    SELECT
        child.id,
        child.parent_id,
        child.level_text,
        child.level_name,
        parent.category_id,
        parent.category_name,

        CASE
            WHEN child.level_name = 'Segment'
                THEN child.id
            ELSE parent.segment_id
        END,

        CASE
            WHEN child.level_name = 'Segment'
                THEN child.level_text
            ELSE parent.segment_name
        END

    FROM balanced_tree.product_hierarchy AS child

    INNER JOIN hierarchy AS parent
        ON child.parent_id = parent.id
)

SELECT
    pp.product_id,
    pp.price,

    h.level_text
        || ' '
        || h.segment_name
        || ' - '
        || h.category_name AS product_name,

    h.category_id,
    h.segment_id,
    h.id AS style_id,
    h.category_name,
    h.segment_name,
    h.level_text AS style_name

FROM hierarchy AS h

INNER JOIN balanced_tree.product_prices AS pp
    ON h.id = pp.id

WHERE h.level_name = 'Style';
```

Check for differences:

```sql
SELECT *
FROM product_details_rebuilt

EXCEPT

SELECT *
FROM balanced_tree.product_details;
```

Then reverse the comparison:

```sql
SELECT *
FROM balanced_tree.product_details

EXCEPT

SELECT *
FROM product_details_rebuilt;
```

Both queries should return zero rows.

---

## Design Insight

The hierarchy-based structure is more scalable than storing repeated category and segment names in every product record.

However, the denormalised `product_details` table is more convenient for frequent sales reporting.

A production system could:

1. Maintain the hierarchy as the source of truth.
2. Generate `product_details` as a view or materialised view.
3. Use the denormalised output for analytics and dashboards.
