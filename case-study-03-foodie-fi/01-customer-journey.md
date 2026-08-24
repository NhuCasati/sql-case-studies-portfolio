# Customer Journey

[← Back to Foodie-Fi overview](./README.md)

## Business Question

Based on the eight sample customers, describe each customer's onboarding and subscription journey.

The customers included in this analysis are:

```text
1, 2, 11, 13, 15, 16, 18 and 19
```

---

## Query

```sql
SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    s.start_date
FROM foodie_fi.subscriptions AS s
INNER JOIN foodie_fi.plans AS p
    ON s.plan_id = p.plan_id
WHERE s.customer_id IN (
    1, 2, 11, 13, 15, 16, 18, 19
)
ORDER BY
    s.customer_id,
    s.start_date;
```

---

## Result

| Customer | Plan ID | Plan | Start date |
|---:|---:|---|---|
| 1 | 0 | Trial | 2020-08-01 |
| 1 | 1 | Basic monthly | 2020-08-08 |
| 2 | 0 | Trial | 2020-09-20 |
| 2 | 3 | Pro annual | 2020-09-27 |
| 11 | 0 | Trial | 2020-11-19 |
| 11 | 4 | Churn | 2020-11-26 |
| 13 | 0 | Trial | 2020-12-15 |
| 13 | 1 | Basic monthly | 2020-12-22 |
| 13 | 2 | Pro monthly | 2021-03-29 |
| 15 | 0 | Trial | 2020-03-17 |
| 15 | 2 | Pro monthly | 2020-03-24 |
| 15 | 4 | Churn | 2020-04-29 |
| 16 | 0 | Trial | 2020-05-31 |
| 16 | 1 | Basic monthly | 2020-06-07 |
| 16 | 3 | Pro annual | 2020-10-21 |
| 18 | 0 | Trial | 2020-07-06 |
| 18 | 2 | Pro monthly | 2020-07-13 |
| 19 | 0 | Trial | 2020-06-22 |
| 19 | 2 | Pro monthly | 2020-06-29 |
| 19 | 3 | Pro annual | 2020-08-29 |

---

## Customer Summaries

### Customer 1

Customer 1 started a free trial on 1 August 2020 and moved to the basic monthly plan when the trial ended seven days later.

### Customer 2

Customer 2 started a free trial on 20 September 2020 and upgraded directly to the pro annual plan after seven days.

### Customer 11

Customer 11 started a free trial on 19 November 2020 and churned immediately when the trial ended.

### Customer 13

Customer 13 started with a free trial, moved to the basic monthly plan after seven days and later upgraded to the pro monthly plan in March 2021.

### Customer 15

Customer 15 moved from the trial to the pro monthly plan but churned approximately one month later.

### Customer 16

Customer 16 moved from the trial to the basic monthly plan and later upgraded to the pro annual plan.

### Customer 18

Customer 18 completed the free trial and then subscribed to the pro monthly plan.

### Customer 19

Customer 19 moved from the trial to the pro monthly plan and upgraded to the pro annual plan two months later.

---

## Insight

The sample illustrates several important customer journeys:

- Immediate trial-to-paid conversions
- Direct annual-plan adoption
- Monthly-to-annual upgrades
- Immediate post-trial churn
- Churn after using a paid plan

These journeys can be analysed further to identify the behaviours associated with retention and churn.
