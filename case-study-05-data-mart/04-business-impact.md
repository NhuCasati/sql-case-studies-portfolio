# Business Impact and Recommendations

[← Back to Data Mart overview](./README.md)

## Objective

Identify which business groups experienced the greatest sales decline across the twelve-week before-and-after period in 2020.

The dimensions analysed are:

- Region
- Platform
- Age band
- Demographic
- Customer type

---

## Reusable Dimensional Query

```sql
WITH event_week AS (
    SELECT
        week_number AS baseline_week
    FROM data_mart.clean_weekly_sales
    WHERE week_date = DATE '2020-06-15'
    LIMIT 1
),

scoped_sales AS (
    SELECT
        cws.*,

        CASE
            WHEN week_number < baseline_week
                THEN 'Before'
            ELSE 'After'
        END AS period

    FROM data_mart.clean_weekly_sales AS cws
    CROSS JOIN event_week

    WHERE calendar_year = 2020
      AND week_number BETWEEN
          baseline_week - 12
          AND baseline_week + 11
),

dimension_sales AS (
    SELECT
        d.dimension,
        d.dimension_value,
        period,
        SUM(sales) AS total_sales

    FROM scoped_sales

    CROSS JOIN LATERAL (
        VALUES
            ('region', region),
            ('platform', platform),
            ('age_band', age_band),
            ('demographic', demographic),
            ('customer_type', customer_type)
    ) AS d(
        dimension,
        dimension_value
    )

    GROUP BY
        d.dimension,
        d.dimension_value,
        period
),

impact AS (
    SELECT
        dimension,
        dimension_value,

        SUM(total_sales) FILTER (
            WHERE period = 'Before'
        ) AS before_sales,

        SUM(total_sales) FILTER (
            WHERE period = 'After'
        ) AS after_sales

    FROM dimension_sales
    GROUP BY
        dimension,
        dimension_value
)

SELECT
    dimension,
    dimension_value,
    before_sales,
    after_sales,

    after_sales
    - before_sales AS sales_change,

    ROUND(
        100.0
        * (
            after_sales
            - before_sales
        )
        / before_sales,
        2
    ) AS percentage_change

FROM impact
ORDER BY
    dimension,
    percentage_change;
```

---

## 1. Regional impact

| Region | Before | After | Change |
|---|---:|---:|---:|
| Asia | 1,637,244,466 | 1,583,807,621 | -3.26% |
| Oceania | 2,354,116,790 | 2,282,795,690 | -3.03% |
| South America | 213,036,207 | 208,452,033 | -2.15% |
| Canada | 426,438,454 | 418,264,441 | -1.92% |
| USA | 677,013,558 | 666,198,715 | -1.60% |
| Africa | 1,709,537,105 | 1,700,390,294 | -0.54% |
| Europe | 108,886,567 | 114,038,959 | +4.73% |

### Insight

Asia experienced the largest percentage decline, closely followed by Oceania.

Europe was the only region to record sales growth.

---

## 2. Platform impact

| Platform | Before | After | Change |
|---|---:|---:|---:|
| Retail | 6,906,861,113 | 6,738,777,279 | -2.43% |
| Shopify | 219,412,034 | 235,170,474 | +7.18% |

### Insight

The overall decline was driven by Retail.

Shopify sales increased after the packaging change, indicating that the negative effect was not consistent across channels.

---

## 3. Age-band impact

| Age band | Before | After | Change |
|---|---:|---:|---:|
| Unknown | 2,764,354,464 | 2,671,961,443 | -3.34% |
| Middle Aged | 1,164,847,640 | 1,141,853,348 | -1.97% |
| Retirees | 2,395,264,515 | 2,365,714,994 | -1.23% |
| Young Adults | 801,806,528 | 794,417,968 | -0.92% |

### Insight

The unknown group recorded the largest decline.

Among identified age bands, Middle Aged customers experienced the largest percentage reduction.

---

## 4. Demographic impact

| Demographic | Before | After | Change |
|---|---:|---:|---:|
| Unknown | 2,764,354,464 | 2,671,961,443 | -3.34% |
| Families | 2,328,329,040 | 2,286,009,025 | -1.82% |
| Couples | 2,033,589,643 | 2,015,977,285 | -0.87% |

### Insight

Families experienced a larger decline than Couples.

The large unknown category remains a major data-quality limitation.

---

## 5. Customer-type impact

| Customer type | Before | After | Change |
|---|---:|---:|---:|
| Guest | 2,573,436,301 | 2,496,233,635 | -3.00% |
| Existing | 3,690,116,427 | 3,606,243,454 | -2.27% |
| New | 862,720,419 | 871,470,664 | +1.01% |

### Insight

Guest customers experienced the largest decline.

New-customer sales increased, suggesting that sustainable packaging may have supported acquisition while negatively affecting some established shopping behaviour.

---

## Most Affected Groups

The strongest negative impacts were:

| Dimension | Most affected group | Change |
|---|---|---:|
| Region | Asia | -3.26% |
| Platform | Retail | -2.43% |
| Age band | Unknown | -3.34% |
| Known age band | Middle Aged | -1.97% |
| Demographic | Unknown | -3.34% |
| Known demographic | Families | -1.82% |
| Customer type | Guest | -3.00% |

---

## Business Recommendations

### 1. Investigate the Retail experience

The decline occurred primarily in Retail rather than Shopify.

Data Mart should investigate:

- Whether the new packaging was difficult to recognise
- Whether product placement changed
- Whether package sizes or prices changed
- Whether customers understood the sustainability message
- Whether checkout or bagging processes were affected

### 2. Focus on Asia and Oceania

These regions contributed large sales volumes and experienced some of the strongest declines.

Regional research should determine whether the packaging change affected:

- Product recognition
- Customer preferences
- Local pricing
- Packaging expectations
- Supply availability

### 3. Improve communication with Guest customers

Guest customers experienced the largest decline by customer type.

Data Mart could test:

- In-store sustainability messaging
- Clear package labelling
- Short-term promotions
- Educational campaigns
- Product comparison displays

### 4. Learn from Shopify growth

Shopify sales increased after the packaging change.

Data Mart should compare the online and Retail customer journeys to identify whether Shopify provided:

- Better product information
- Clearer sustainability messaging
- More effective product images
- Easier product discovery

### 5. Improve customer-segment data

Unknown customers represent a very large portion of sales.

Improving demographic-data collection would make future customer-impact analysis more reliable.

### 6. Use controlled rollouts

Future operational changes should be introduced through:

1. Pilot regions or stores
2. Control groups
3. Predefined success measures
4. Customer surveys
5. Weekly monitoring
6. Gradual expansion

---

## Final Conclusion

The packaging change was followed by a measurable sales decline in 2020.

However, the effect differed considerably across the business:

- Retail declined while Shopify grew.
- Most regions declined, but Europe grew.
- Guest and Existing customers declined, while New customers grew.
- Asia and Oceania were among the most affected regions.

Future sustainability initiatives should be introduced gradually and supported by clear customer communication, controlled testing and more detailed customer data.
