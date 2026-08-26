# Segment Analysis

[← Back to Fresh Segments overview](./README.md)

> Create `interest_metrics_filtered` in the Interest Analysis section before running these queries.

## Objective

Analyse the strongest, weakest and most volatile customer interests based on:

- Composition
- Average ranking
- Percentile-ranking variability

---

## 1. Which interests had the highest and lowest maximum composition?

### SQL approach

1. Rank each interest's monthly records by composition.
2. Retain the highest composition for every interest.
3. Rank the retained records from highest to lowest.
4. Return the top and bottom ten interests.

### Query

```sql
WITH interest_maximums AS (
    SELECT
        m.month_year,
        m.interest_id,
        im.interest_name,
        m.composition,

        ROW_NUMBER() OVER (
            PARTITION BY m.interest_id
            ORDER BY
                m.composition DESC,
                m.month_year
        ) AS interest_record_rank

    FROM fresh_segments.interest_metrics_filtered AS m

    INNER JOIN fresh_segments.interest_map AS im
        ON m.interest_id = im.id
),

ranked_maximums AS (
    SELECT
        month_year,
        interest_id,
        interest_name,
        composition,

        ROW_NUMBER() OVER (
            ORDER BY
                composition DESC,
                interest_id
        ) AS highest_rank,

        ROW_NUMBER() OVER (
            ORDER BY
                composition,
                interest_id
        ) AS lowest_rank

    FROM interest_maximums
    WHERE interest_record_rank = 1
)

SELECT
    'Top 10' AS result_group,
    month_year,
    interest_id,
    interest_name,
    composition

FROM ranked_maximums
WHERE highest_rank <= 10

UNION ALL

SELECT
    'Bottom 10' AS result_group,
    month_year,
    interest_id,
    interest_name,
    composition

FROM ranked_maximums
WHERE lowest_rank <= 10

ORDER BY
    result_group DESC,
    composition DESC;
```

### Top interests

| Rank | Interest |
|---:|---|
| 1 | Work Comes First Travelers |
| 2 | Gym Equipment Owners |
| 3 | Furniture Shoppers |
| 4 | Luxury Retail Shoppers |
| 5 | Luxury Boutique Hotel Researchers |
| 6 | Luxury Bedding Shoppers |
| 7 | Shoe Shoppers |
| 8 | Cosmetics and Beauty Shoppers |
| 9 | Luxury Hotel Guests |
| 10 | Luxury Retail Researchers |

The largest value was:

| Month | Interest | Composition |
|---|---|---:|
| 2018-12-01 | Work Comes First Travelers | 21.20 |

### Insight

The strongest interests are concentrated around:

- Travel
- Luxury accommodation
- Retail shopping
- Beauty products
- Furniture
- Fitness equipment

---

## 2. Which interests had the lowest average ranking?

A lower ranking represents stronger relative performance.

### Query

```sql
SELECT
    m.interest_id,
    im.interest_name,

    ROUND(
        AVG(m.ranking),
        2
    ) AS average_ranking,

    COUNT(*) AS months_present

FROM fresh_segments.interest_metrics_filtered AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

GROUP BY
    m.interest_id,
    im.interest_name

ORDER BY average_ranking
LIMIT 5;
```

### Result

| Interest | Average ranking |
|---|---:|
| Winter Apparel Shoppers | 1.00 |
| Fitness Activity Tracker Users | 4.11 |
| Mens Shoe Shoppers | 5.93 |
| Shoe Shoppers | 9.36 |
| Luxury Retail Researchers | 11.86 |

### Insight

The strongest consistently ranked interests are dominated by apparel, footwear, fitness technology and luxury retail.

---

## 3. Which interests had the greatest percentile-ranking variation?

### Query

```sql
SELECT
    m.interest_id,
    im.interest_name,

    ROUND(
        STDDEV_SAMP(
            m.percentile_ranking
        ),
        2
    ) AS percentile_standard_deviation,

    COUNT(*) AS months_present

FROM fresh_segments.interest_metrics_filtered AS m

INNER JOIN fresh_segments.interest_map AS im
    ON m.interest_id = im.id

GROUP BY
    m.interest_id,
    im.interest_name

ORDER BY
    percentile_standard_deviation DESC
    NULLS LAST

LIMIT 5;
```

### Result

| Interest | Standard deviation |
|---|---:|
| Techies | 30.18 |
| Entertainment Industry Decision Makers | 28.97 |
| Oregon Trip Planners | 28.32 |
| Personalized Gift Shoppers | 26.24 |
| Tampa and St Petersburg Trip Planners | 25.61 |

### Insight

These interests experienced the largest changes in relative popularity across the reporting period.

---

## 4. Find the minimum and maximum percentile rankings

### Query

```sql
WITH interest_variation AS (
    SELECT
        m.interest_id,
        im.interest_name,
        im.interest_summary,

        STDDEV_SAMP(
            m.percentile_ranking
        ) AS percentile_standard_deviation

    FROM fresh_segments.interest_metrics_filtered AS m

    INNER JOIN fresh_segments.interest_map AS im
        ON m.interest_id = im.id

    GROUP BY
        m.interest_id,
        im.interest_name,
        im.interest_summary
),

top_variable_interests AS (
    SELECT
        interest_id,
        interest_name,
        interest_summary,
        percentile_standard_deviation

    FROM interest_variation

    ORDER BY percentile_standard_deviation DESC
    LIMIT 5
),

interest_extremes AS (
    SELECT
        m.interest_id,
        t.interest_name,
        t.interest_summary,
        m.month_year,
        m.percentile_ranking,

        MIN(m.percentile_ranking) OVER (
            PARTITION BY m.interest_id
        ) AS minimum_percentile,

        MAX(m.percentile_ranking) OVER (
            PARTITION BY m.interest_id
        ) AS maximum_percentile

    FROM fresh_segments.interest_metrics_filtered AS m

    INNER JOIN top_variable_interests AS t
        ON m.interest_id = t.interest_id
)

SELECT
    interest_id,
    interest_name,

    MAX(month_year) FILTER (
        WHERE percentile_ranking = maximum_percentile
    ) AS maximum_month,

    MAX(maximum_percentile) AS maximum_percentile,

    MIN(month_year) FILTER (
        WHERE percentile_ranking = minimum_percentile
    ) AS minimum_month,

    MIN(minimum_percentile) AS minimum_percentile

FROM interest_extremes

GROUP BY
    interest_id,
    interest_name

ORDER BY interest_name;
```

### Result

| Interest | Maximum month | Maximum | Minimum month | Minimum |
|---|---|---:|---|---:|
| Tampa and St Petersburg Trip Planners | 2018-07-01 | 75.03 | 2019-03-01 | 4.84 |
| Entertainment Industry Decision Makers | 2018-07-01 | 86.15 | 2019-08-01 | 11.23 |
| Techies | 2018-07-01 | 86.69 | 2019-08-01 | 7.92 |
| Oregon Trip Planners | 2018-11-01 | 82.44 | 2019-07-01 | 2.20 |
| Personalized Gift Shoppers | 2019-03-01 | 73.15 | 2019-06-01 | 5.70 |

### Insight

The large differences between minimum and maximum percentile rankings indicate that these interests are not consistently popular.

Possible explanations include:

- Seasonal travel demand
- Holiday gift purchasing
- One-time events
- Marketing campaigns
- Changing technology trends
- Changes to the comparison population

For example, travel interest may be strong during holiday-planning periods but weak during other months.

---

## 5. Customer-segment description

The strongest interests indicate that this customer group is particularly interested in:

- Travel and accommodation
- Clothing and footwear
- Luxury retail
- Fitness products
- Beauty products
- Technology and entertainment trends
- Personalised gifts

### Recommended products and services

Fresh Segments could show these customers:

- Travel packages
- Hotels and flights
- Seasonal clothing
- Footwear
- Fitness trackers
- Luxury products
- Beauty and cosmetics
- Personalised gifts
- Technology news and products
- Entertainment-industry content

### Content to avoid

The client should limit:

- Repeated promotions after seasonal demand has ended
- Generic campaigns unrelated to the observed interests
- Long-term assumptions based on one unusually strong month
- Expensive campaigns targeting highly volatile interests without testing

### Marketing recommendation

Separate customer interests into:

1. **Consistent interests** for ongoing campaigns
2. **Seasonal interests** for time-limited campaigns
3. **Emerging interests** for experimental campaigns
4. **Declining interests** for reduced advertising investment
