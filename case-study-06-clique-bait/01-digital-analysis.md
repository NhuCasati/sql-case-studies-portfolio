# Digital Analysis

[← Back to Clique Bait overview](./README.md)

## Overview

This section examines user activity, website visits, event volume, checkout behaviour and product engagement.

---

## 1. How many users are there?

### Query

```sql
SELECT
    COUNT(DISTINCT user_id) AS total_users
FROM clique_bait.users;
```

### Result

| Total users |
|---:|
| 500 |

### Insight

Clique Bait has 500 identifiable users in the dataset.

---

## 2. How many cookies does each user have on average?

### SQL approach

First count the number of cookies assigned to each user, then calculate the average of those counts.

### Query

```sql
WITH user_cookies AS (
    SELECT
        user_id,
        COUNT(DISTINCT cookie_id) AS cookie_count
    FROM clique_bait.users
    GROUP BY user_id
)

SELECT
    ROUND(
        AVG(cookie_count),
        2
    ) AS average_cookies
FROM user_cookies;
```

### Result

| Average cookies |
|---:|
| 3.56 |

### Insight

Each user has approximately 3.56 cookies, indicating that customers may access Clique Bait from several devices or browser sessions.

---

## 3. How many unique visits occurred each month?

### Query

```sql
SELECT
    DATE_TRUNC(
        'month',
        event_time
    )::DATE AS visit_month,

    COUNT(
        DISTINCT visit_id
    ) AS unique_visits

FROM clique_bait.events
GROUP BY DATE_TRUNC(
    'month',
    event_time
)
ORDER BY visit_month;
```

### Result

| Month | Unique visits |
|---|---:|
| 2020-01-01 | 876 |
| 2020-02-01 | 1,488 |
| 2020-03-01 | 916 |
| 2020-04-01 | 248 |
| 2020-05-01 | 36 |

### Insight

February recorded the highest number of unique visits.

The lower figures for April and May may reflect incomplete coverage rather than a sustained traffic decline.

---

## 4. How many events occurred for each event type?

### Query

```sql
SELECT
    ei.event_name,
    COUNT(*) AS event_count
FROM clique_bait.events AS e
INNER JOIN clique_bait.event_identifier AS ei
    ON e.event_type = ei.event_type
GROUP BY
    ei.event_type,
    ei.event_name
ORDER BY event_count DESC;
```

### Result

| Event | Count |
|---|---:|
| Page View | 20,928 |
| Add to Cart | 8,451 |
| Purchase | 1,777 |
| Ad Impression | 876 |
| Ad Click | 702 |

### Insight

Page views were the most frequent event, followed by shopping-cart additions.

---

## 5. What percentage of visits contained a purchase event?

### Query

```sql
WITH visit_flags AS (
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
)

SELECT
    ROUND(
        100.0
        * SUM(purchased)
        / COUNT(*),
        2
    ) AS purchase_visit_percentage
FROM visit_flags;
```

### Result

| Purchase visits |
|---:|
| 49.86% |

### Insight

Approximately half of all recorded visits ended with a purchase.

---

## 6. What percentage of checkout visits did not result in a purchase?

### SQL approach

Create checkout and purchase flags for every visit, then calculate the percentage of checkout visits without a purchase.

### Query

```sql
WITH visit_flags AS (
    SELECT
        visit_id,

        MAX(
            CASE
                WHEN event_type = 1
                 AND page_id = 12
                    THEN 1
                ELSE 0
            END
        ) AS reached_checkout,

        MAX(
            CASE
                WHEN event_type = 3
                    THEN 1
                ELSE 0
            END
        ) AS purchased

    FROM clique_bait.events
    GROUP BY visit_id
)

SELECT
    ROUND(
        100.0
        * COUNT(*) FILTER (
            WHERE reached_checkout = 1
              AND purchased = 0
        )
        / NULLIF(
            COUNT(*) FILTER (
                WHERE reached_checkout = 1
            ),
            0
        ),
        2
    ) AS checkout_abandonment_percentage
FROM visit_flags;
```

### Result

| Checkout abandonment |
|---:|
| 15.50% |

### Insight

Approximately 15.5% of visits that reached the checkout page did not complete a purchase.

---

## 7. What were the top three pages by number of views?

### Query

```sql
SELECT
    ph.page_name,
    COUNT(*) AS page_views
FROM clique_bait.events AS e
INNER JOIN clique_bait.page_hierarchy AS ph
    ON e.page_id = ph.page_id
WHERE e.event_type = 1
GROUP BY ph.page_name
ORDER BY page_views DESC
LIMIT 3;
```

### Result

| Page | Views |
|---|---:|
| All Products | 3,174 |
| Checkout | 2,103 |
| Home Page | 1,782 |

### Insight

The All Products page was the most viewed page, suggesting that visitors frequently browse across the complete product range.

---

## 8. How many views and cart additions occurred for each product category?

### Query

```sql
SELECT
    ph.product_category,

    COUNT(*) FILTER (
        WHERE e.event_type = 1
    ) AS product_views,

    COUNT(*) FILTER (
        WHERE e.event_type = 2
    ) AS cart_adds

FROM clique_bait.events AS e
INNER JOIN clique_bait.page_hierarchy AS ph
    ON e.page_id = ph.page_id
WHERE ph.product_category IS NOT NULL
GROUP BY ph.product_category
ORDER BY product_views DESC;
```

### Result

| Category | Views | Cart additions |
|---|---:|---:|
| Shellfish | 6,204 | 3,792 |
| Fish | 4,633 | 2,789 |
| Luxury | 3,032 | 1,870 |

### Insight

Shellfish generated the highest number of both product views and cart additions.

---

## 9. What were the top three products by purchases?

### SQL approach

A product is treated as purchased when it was added to a cart during a visit that contained a purchase event.

### Query

```sql
WITH purchased_visits AS (
    SELECT DISTINCT
        visit_id
    FROM clique_bait.events
    WHERE event_type = 3
)

SELECT
    ph.page_name AS product_name,
    COUNT(*) AS purchases
FROM clique_bait.events AS e
INNER JOIN purchased_visits AS pv
    ON e.visit_id = pv.visit_id
INNER JOIN clique_bait.page_hierarchy AS ph
    ON e.page_id = ph.page_id
WHERE e.event_type = 2
  AND ph.product_id IS NOT NULL
GROUP BY ph.page_name
ORDER BY purchases DESC
LIMIT 3;
```

### Result

| Product | Purchases |
|---|---:|
| Lobster | 754 |
| Oyster | 726 |
| Crab | 719 |

### Insight

The three most purchased products were all from the Shellfish category.
