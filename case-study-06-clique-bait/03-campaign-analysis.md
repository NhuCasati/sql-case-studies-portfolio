# Campaign Analysis

[← Back to Clique Bait overview](./README.md)

## Objective

Create one record for each website visit containing:

- User ID
- Visit ID
- Visit start time
- Page views
- Cart additions
- Purchase indicator
- Campaign name
- Ad impressions
- Ad clicks
- Products added to the cart

---

## 1. Create the visit-summary table

```sql
DROP TABLE IF EXISTS clique_bait.visit_summary;

CREATE TABLE clique_bait.visit_summary AS

WITH visit_metrics AS (
    SELECT
        u.user_id,
        e.visit_id,
        MIN(e.event_time) AS visit_start_time,

        COUNT(*) FILTER (
            WHERE e.event_type = 1
        ) AS page_views,

        COUNT(*) FILTER (
            WHERE e.event_type = 2
              AND ph.product_id IS NOT NULL
        ) AS cart_adds,

        MAX(
            CASE
                WHEN e.event_type = 3 THEN 1
                ELSE 0
            END
        ) AS purchase,

        COUNT(*) FILTER (
            WHERE e.event_type = 4
        ) AS impression,

        COUNT(*) FILTER (
            WHERE e.event_type = 5
        ) AS click,

        STRING_AGG(
            ph.page_name,
            ', '
            ORDER BY e.sequence_number
        ) FILTER (
            WHERE e.event_type = 2
              AND ph.product_id IS NOT NULL
        ) AS cart_products

    FROM clique_bait.events AS e

    INNER JOIN clique_bait.users AS u
        ON e.cookie_id = u.cookie_id

    LEFT JOIN clique_bait.page_hierarchy AS ph
        ON e.page_id = ph.page_id

    GROUP BY
        u.user_id,
        e.visit_id
)

SELECT
    vm.user_id,
    vm.visit_id,
    vm.visit_start_time,
    vm.page_views,
    vm.cart_adds,
    vm.purchase,
    ci.campaign_name,
    vm.impression,
    vm.click,
    vm.cart_products

FROM visit_metrics AS vm

LEFT JOIN clique_bait.campaign_identifier AS ci
    ON vm.visit_start_time
       BETWEEN ci.start_date AND ci.end_date;
```

### Validate the table

```sql
SELECT *
FROM clique_bait.visit_summary
ORDER BY
    user_id,
    visit_start_time
LIMIT 20;
```

### Example output

| User | Visit | Page views | Cart adds | Purchase | Campaign | Impression | Click |
|---:|---|---:|---:|---:|---|---:|---:|
| 1 | 0fc437 | 10 | 6 | 1 | Half Off - Treat Your Shellf(ish) | 1 | 1 |
| 1 | 0826dc | 1 | 0 | 0 | Half Off - Treat Your Shellf(ish) | 0 | 0 |
| 1 | 30b94d | 9 | 7 | 1 | Half Off - Treat Your Shellf(ish) | 1 | 1 |

---

# Campaign Insights

## 1. How did each campaign perform?

```sql
SELECT
    COALESCE(
        campaign_name,
        'No Campaign'
    ) AS campaign,

    COUNT(*) AS visits,

    SUM(purchase) AS purchases,

    ROUND(
        AVG(page_views),
        2
    ) AS average_page_views,

    ROUND(
        AVG(cart_adds),
        2
    ) AS average_cart_adds,

    ROUND(
        100.0
        * SUM(purchase)
        / COUNT(*),
        2
    ) AS purchase_rate

FROM clique_bait.visit_summary

GROUP BY COALESCE(
    campaign_name,
    'No Campaign'
)

ORDER BY visits DESC;
```

### Insight to document

Compare campaign reach, visitor engagement and purchase conversion rather than judging campaign success from visit volume alone.

---

## 2. Did visitors who received an impression perform differently?

```sql
SELECT
    CASE
        WHEN impression > 0
            THEN 'Received impression'
        ELSE 'No impression'
    END AS impression_group,

    COUNT(*) AS visits,

    ROUND(
        AVG(page_views),
        2
    ) AS average_page_views,

    ROUND(
        AVG(cart_adds),
        2
    ) AS average_cart_adds,

    ROUND(
        100.0
        * SUM(purchase)
        / COUNT(*),
        2
    ) AS purchase_rate

FROM clique_bait.visit_summary

GROUP BY
    CASE
        WHEN impression > 0
            THEN 'Received impression'
        ELSE 'No impression'
    END

ORDER BY purchase_rate DESC;
```

### Insight to document

Determine whether visitors exposed to an advertisement displayed stronger engagement and conversion.

This represents association rather than proof that the impression caused the purchase.

---

## 3. Did clicking an advertisement improve conversion?

```sql
SELECT
    CASE
        WHEN click > 0
            THEN 'Clicked advertisement'

        WHEN impression > 0
            THEN 'Impression without click'

        ELSE 'No impression'
    END AS advertising_group,

    COUNT(*) AS visits,

    SUM(purchase) AS purchases,

    ROUND(
        100.0
        * SUM(purchase)
        / COUNT(*),
        2
    ) AS purchase_rate

FROM clique_bait.visit_summary

GROUP BY
    CASE
        WHEN click > 0
            THEN 'Clicked advertisement'

        WHEN impression > 0
            THEN 'Impression without click'

        ELSE 'No impression'
    END

ORDER BY purchase_rate DESC;
```

### Insight to document

Compare customers who:

- Clicked an advertisement
- Viewed an advertisement but did not click
- Received no advertisement

The result shows whether stronger advertising engagement is associated with higher purchase conversion.

---

## 4. What was the advertising click-through rate?

```sql
SELECT
    COUNT(*) FILTER (
        WHERE impression > 0
    ) AS impression_visits,

    COUNT(*) FILTER (
        WHERE click > 0
    ) AS click_visits,

    ROUND(
        100.0
        * COUNT(*) FILTER (
            WHERE click > 0
        )
        / NULLIF(
            COUNT(*) FILTER (
                WHERE impression > 0
            ),
            0
        ),
        2
    ) AS click_through_rate

FROM clique_bait.visit_summary;
```

### Insight to document

The click-through rate measures how effectively ad impressions generated active engagement.

---

## 5. Which campaign generated the most advertisement clicks?

```sql
SELECT
    campaign_name,
    COUNT(*) FILTER (
        WHERE impression > 0
    ) AS impression_visits,

    COUNT(*) FILTER (
        WHERE click > 0
    ) AS click_visits,

    ROUND(
        100.0
        * COUNT(*) FILTER (
            WHERE click > 0
        )
        / NULLIF(
            COUNT(*) FILTER (
                WHERE impression > 0
            ),
            0
        ),
        2
    ) AS click_through_rate

FROM clique_bait.visit_summary

WHERE campaign_name IS NOT NULL

GROUP BY campaign_name
ORDER BY click_visits DESC;
```

### Insight to document

Separate total clicks from click-through rate:

- Total clicks measure campaign scale.
- Click-through rate measures campaign efficiency.

---

## 6. Which products were added most often during each campaign?

```sql
WITH campaign_product_adds AS (
    SELECT
        COALESCE(
            vs.campaign_name,
            'No Campaign'
        ) AS campaign,

        ph.page_name AS product_name,

        COUNT(*) AS cart_adds

    FROM clique_bait.events AS e

    INNER JOIN clique_bait.visit_summary AS vs
        ON e.visit_id = vs.visit_id

    INNER JOIN clique_bait.page_hierarchy AS ph
        ON e.page_id = ph.page_id

    WHERE e.event_type = 2
      AND ph.product_id IS NOT NULL

    GROUP BY
        COALESCE(
            vs.campaign_name,
            'No Campaign'
        ),
        ph.page_name
),

ranked_products AS (
    SELECT
        campaign,
        product_name,
        cart_adds,

        DENSE_RANK() OVER (
            PARTITION BY campaign
            ORDER BY cart_adds DESC
        ) AS product_rank

    FROM campaign_product_adds
)

SELECT
    campaign,
    product_name,
    cart_adds
FROM ranked_products
WHERE product_rank = 1
ORDER BY campaign;
```

### Insight to document

This identifies the product most strongly associated with cart activity during each campaign.

Compare the result with the products specifically promoted by each campaign.

---

## 7. How much purchase-rate uplift was associated with ad clicks?

```sql
WITH group_rates AS (
    SELECT
        CASE
            WHEN click > 0
                THEN 'Clicked advertisement'
            ELSE 'Did not click'
        END AS click_group,

        100.0
        * SUM(purchase)
        / COUNT(*) AS purchase_rate

    FROM clique_bait.visit_summary
    GROUP BY
        CASE
            WHEN click > 0
                THEN 'Clicked advertisement'
            ELSE 'Did not click'
        END
)

SELECT
    MAX(purchase_rate) FILTER (
        WHERE click_group = 'Clicked advertisement'
    ) AS clicked_purchase_rate,

    MAX(purchase_rate) FILTER (
        WHERE click_group = 'Did not click'
    ) AS non_clicked_purchase_rate,

    ROUND(
        (
            MAX(purchase_rate) FILTER (
                WHERE click_group = 'Clicked advertisement'
            )
            -
            MAX(purchase_rate) FILTER (
                WHERE click_group = 'Did not click'
            )
        )::NUMERIC,
        2
    ) AS percentage_point_uplift

FROM group_rates;
```

### Insight to document

The result measures the difference in purchase rates between visitors who clicked an advertisement and those who did not.

It should not be interpreted as causal uplift without randomised campaign assignment.

---

## Recommended Management Metrics

Clique Bait should monitor:

- Visits by campaign
- Advertisement impressions
- Advertisement clicks
- Click-through rate
- Purchase conversion rate
- Cart additions per visit
- Checkout abandonment
- Product-level conversion
- Campaign-attributed purchases
- Incremental revenue, when price data becomes available

---

## Final Recommendations

### Improve checkout conversion

Investigate the visitors who reached checkout but did not purchase.

Potential issues include:

- Unexpected delivery costs
- Payment problems
- Checkout complexity
- Product availability
- Lack of trust information

### Retarget abandoned carts

Russian Caviar and other frequently abandoned products could be included in personalised reminder campaigns.

### Promote high-converting products

Lobster performed strongly across cart additions, purchases and view-to-purchase conversion.

### Test campaigns experimentally

Future campaigns should use control and treatment groups so that campaign effects can be separated from existing customer behaviour.

### Add revenue data

Campaign success should ultimately be measured using revenue and profit rather than event volume alone.
