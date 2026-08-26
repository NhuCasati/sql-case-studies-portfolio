# Index Analysis

[← Back to Fresh Segments overview](./README.md)

## Objective

Use the `index_value` metric to estimate the average composition for all Fresh Segments clients.

The calculation is:

```text
Average composition =
client composition ÷ index value
```

---

## 1. What were the top ten interests by average composition each month?

### Query

```sql
WITH ranked_interests AS (
    SELECT
        m.month_year,
        m.interest_id,
        im.interest_name,

        ROUND(
            (
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                )
            )::NUMERIC,
            2
        ) AS average_composition,

        ROW_NUMBER() OVER (
            PARTITION BY m.month_year
            ORDER BY
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                ) DESC,
                m.interest_id
        ) AS monthly_rank

    FROM fresh_segments.interest_metrics_clean AS m

    INNER JOIN fresh_segments.interest_map AS im
        ON m.interest_id = im.id
)

SELECT
    month_year,
    interest_id,
    interest_name,
    average_composition,
    monthly_rank

FROM ranked_interests

WHERE monthly_rank <= 10

ORDER BY
    month_year,
    monthly_rank;
```

The query returns exactly ten interests for each of the 14 months.

### July 2018 example

| Rank | Interest | Average composition |
|---:|---|---:|
| 1 | Las Vegas Trip Planners | 7.36 |
| 2 | Gym Equipment Owners | 6.94 |
| 3 | Cosmetics and Beauty Shoppers | 6.78 |
| 4 | Luxury Retail Shoppers | 6.61 |
| 5 | Furniture Shoppers | 6.51 |
| 6 | Asian Food Enthusiasts | 6.10 |
| 7 | Recently Retired Individuals | 5.72 |
| 8 | Family Adventures Travelers | 4.85 |
| 9 | Work Comes First Travelers | 4.80 |
| 10 | HDTV Researchers | 4.71 |

---

## 2. Which interests appeared most frequently in the monthly top ten?

### Query

```sql
WITH ranked_interests AS (
    SELECT
        m.month_year,
        m.interest_id,
        im.interest_name,

        ROW_NUMBER() OVER (
            PARTITION BY m.month_year
            ORDER BY
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                ) DESC,
                m.interest_id
        ) AS monthly_rank

    FROM fresh_segments.interest_metrics_clean AS m

    INNER JOIN fresh_segments.interest_map AS im
        ON m.interest_id = im.id
),

interest_frequency AS (
    SELECT
        interest_id,
        interest_name,
        COUNT(*) AS top_ten_appearances

    FROM ranked_interests

    WHERE monthly_rank <= 10

    GROUP BY
        interest_id,
        interest_name
)

SELECT
    interest_id,
    interest_name,
    top_ten_appearances

FROM interest_frequency

WHERE top_ten_appearances = (
    SELECT MAX(top_ten_appearances)
    FROM interest_frequency
)

ORDER BY interest_name;
```

### Result

| Interest | Top-ten appearances |
|---|---:|
| Alabama Trip Planners | 10 |
| Luxury Bedding Shoppers | 10 |
| Solar Energy Researchers | 10 |

### Insight

These three interests displayed the most consistent above-average client composition.

---

## 3. What was the average top-ten composition each month?

### Query

```sql
WITH ranked_interests AS (
    SELECT
        m.month_year,

        ROUND(
            (
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                )
            )::NUMERIC,
            2
        ) AS average_composition,

        ROW_NUMBER() OVER (
            PARTITION BY m.month_year
            ORDER BY
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                ) DESC,
                m.interest_id
        ) AS monthly_rank

    FROM fresh_segments.interest_metrics_clean AS m
)

SELECT
    month_year,

    ROUND(
        AVG(average_composition),
        2
    ) AS top_ten_average_composition

FROM ranked_interests

WHERE monthly_rank <= 10

GROUP BY month_year
ORDER BY month_year;
```

### Result

| Month | Top-ten average |
|---|---:|
| 2018-07-01 | 6.04 |
| 2018-08-01 | 5.95 |
| 2018-09-01 | 6.90 |
| 2018-10-01 | 7.07 |
| 2018-11-01 | 6.62 |
| 2018-12-01 | 6.65 |
| 2019-01-01 | 6.40 |
| 2019-02-01 | 6.58 |
| 2019-03-01 | 6.17 |
| 2019-04-01 | 5.75 |
| 2019-05-01 | 3.54 |
| 2019-06-01 | 2.43 |
| 2019-07-01 | 2.77 |
| 2019-08-01 | 2.63 |

### Insight

Top-interest average composition remained relatively strong through April 2019 and then declined sharply from May onward.

---

## 4. Calculate the three-month rolling average

### Query

```sql
WITH monthly_interest_values AS (
    SELECT
        m.month_year,
        m.interest_id,
        im.interest_name,

        ROUND(
            (
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                )
            )::NUMERIC,
            2
        ) AS average_composition,

        ROW_NUMBER() OVER (
            PARTITION BY m.month_year
            ORDER BY
                m.composition
                / NULLIF(
                    m.index_value,
                    0
                ) DESC,
                m.interest_id
        ) AS monthly_rank

    FROM fresh_segments.interest_metrics_clean AS m

    INNER JOIN fresh_segments.interest_map AS im
        ON m.interest_id = im.id
),

monthly_maximums AS (
    SELECT
        month_year,
        interest_name,
        average_composition AS max_index_composition

    FROM monthly_interest_values
    WHERE monthly_rank = 1
),

moving_values AS (
    SELECT
        month_year,
        interest_name,
        max_index_composition,

        ROUND(
            AVG(max_index_composition) OVER (
                ORDER BY month_year
                ROWS BETWEEN 2 PRECEDING
                         AND CURRENT ROW
            ),
            2
        ) AS three_month_moving_average,

        LAG(interest_name) OVER (
            ORDER BY month_year
        )
        || ': '
        || LAG(max_index_composition) OVER (
            ORDER BY month_year
        ) AS one_month_ago,

        LAG(interest_name, 2) OVER (
            ORDER BY month_year
        )
        || ': '
        || LAG(max_index_composition, 2) OVER (
            ORDER BY month_year
        ) AS two_months_ago

    FROM monthly_maximums
)

SELECT
    month_year,
    interest_name,
    max_index_composition,
    three_month_moving_average,
    one_month_ago,
    two_months_ago

FROM moving_values

WHERE month_year BETWEEN
      DATE '2018-09-01'
      AND DATE '2019-08-01'

ORDER BY month_year;
```

### Result

| Month | Top interest | Maximum | 3-month average | 1 month ago | 2 months ago |
|---|---|---:|---:|---|---|
| 2018-09-01 | Work Comes First Travelers | 8.26 | 7.61 | Las Vegas Trip Planners: 7.21 | Las Vegas Trip Planners: 7.36 |
| 2018-10-01 | Work Comes First Travelers | 9.14 | 8.20 | Work Comes First Travelers: 8.26 | Las Vegas Trip Planners: 7.21 |
| 2018-11-01 | Work Comes First Travelers | 8.28 | 8.56 | Work Comes First Travelers: 9.14 | Work Comes First Travelers: 8.26 |
| 2018-12-01 | Work Comes First Travelers | 8.31 | 8.58 | Work Comes First Travelers: 8.28 | Work Comes First Travelers: 9.14 |
| 2019-01-01 | Work Comes First Travelers | 7.66 | 8.08 | Work Comes First Travelers: 8.31 | Work Comes First Travelers: 8.28 |
| 2019-02-01 | Work Comes First Travelers | 7.66 | 7.88 | Work Comes First Travelers: 7.66 | Work Comes First Travelers: 8.31 |
| 2019-03-01 | Alabama Trip Planners | 6.54 | 7.29 | Work Comes First Travelers: 7.66 | Work Comes First Travelers: 7.66 |
| 2019-04-01 | Solar Energy Researchers | 6.28 | 6.83 | Alabama Trip Planners: 6.54 | Work Comes First Travelers: 7.66 |
| 2019-05-01 | Readers of Honduran Content | 4.41 | 5.74 | Solar Energy Researchers: 6.28 | Alabama Trip Planners: 6.54 |
| 2019-06-01 | Las Vegas Trip Planners | 2.77 | 4.49 | Readers of Honduran Content: 4.41 | Solar Energy Researchers: 6.28 |
| 2019-07-01 | Las Vegas Trip Planners | 2.82 | 3.33 | Las Vegas Trip Planners: 2.77 | Readers of Honduran Content: 4.41 |
| 2019-08-01 | Cosmetics and Beauty Shoppers | 2.73 | 2.77 | Las Vegas Trip Planners: 2.82 | Las Vegas Trip Planners: 2.77 |

### Insight

The three-month moving average declined from `8.58` in December 2018 to `2.77` in August 2019.

The leading interests also changed from work-related travel to regional travel, energy, international content and beauty products.

---

## 5. Why might maximum average composition change?

Possible explanations include:

### Seasonal behaviour

Travel, gifts, clothing and holiday-related interests naturally change across the year.

### Changes to the customer list

The client may add or remove customers, changing the audience composition.

### Changes to the comparison group

The index compares the client with other Fresh Segments clients. A changing comparison population can affect the index even when the client's behaviour remains stable.

### Marketing activity

Advertising campaigns can temporarily increase interactions with particular interests.

### Interest-definition changes

Changes to interest models or data-collection methods may alter monthly values.

### Data-quality problems

The sharp decline after April 2019 could indicate:

- Reduced customer matching
- Missing interaction data
- Changes to tracking systems
- Changes to the client list
- New interest-classification logic

---

## Business Model Observation

Fresh Segments relies on relative index values rather than only the client's direct composition.

This creates a potential weakness: an interest's index can change because of changes among other Fresh Segments clients, not because the analysed client's customers changed.

Fresh Segments should therefore report both:

- Direct client composition
- Relative index value

The agency should also monitor:

- Customer-list size
- Match rate
- Number of active interests
- Changes in comparison populations
- Missing records
- Tracking-system changes

---

## Final Recommendations

1. Use stable interests for ongoing campaigns.
2. Use seasonal interests for temporary promotions.
3. Investigate the sharp decline beginning in May 2019.
4. Track composition and index values separately.
5. Document changes to customer lists and interest models.
6. Avoid interpreting index changes as direct evidence of customer-behaviour changes.
