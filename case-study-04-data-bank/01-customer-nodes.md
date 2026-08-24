# Customer Nodes Exploration

[← Back to Data Bank overview](./README.md)

## Overview

This section analyses the distribution of customers across Data Bank's global node network and measures how frequently customers are reassigned.

The active node-allocation date `9999-12-31` is excluded from duration calculations.

---

## 1. How many unique nodes are in the Data Bank system?

### Query

```sql
SELECT
    COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes;
```

### Result

| Unique nodes |
|---:|
| 5 |

### Insight

Data Bank operates five unique nodes across its global network.

---

## 2. How many nodes are available in each region?

### Query

```sql
SELECT
    r.region_id,
    r.region_name,
    COUNT(DISTINCT cn.node_id) AS node_count
FROM data_bank.regions AS r
INNER JOIN data_bank.customer_nodes AS cn
    ON r.region_id = cn.region_id
GROUP BY
    r.region_id,
    r.region_name
ORDER BY r.region_id;
```

### Result

| Region | Nodes |
|---|---:|
| Africa | 5 |
| America | 5 |
| Asia | 5 |
| Region 4 | 5 |
| Region 5 | 5 |

Replace `Region 4` and `Region 5` with the names returned by your DB Fiddle dataset.

### Insight

All regions have access to the complete five-node network.

---

## 3. How many customers are allocated to each region?

### SQL approach

Count distinct customers rather than allocation records because each customer can appear several times in `customer_nodes`.

### Query

```sql
SELECT
    r.region_id,
    r.region_name,
    COUNT(DISTINCT cn.customer_id) AS customer_count
FROM data_bank.regions AS r
INNER JOIN data_bank.customer_nodes AS cn
    ON r.region_id = cn.region_id
GROUP BY
    r.region_id,
    r.region_name
ORDER BY
    customer_count DESC,
    r.region_name;
```

### Result

Add the result produced by DB Fiddle.

### Insight

This result identifies the regions with the largest customer populations and can support regional capacity planning.

---

## 4. How many days on average are customers allocated to a node?

### Query

```sql
SELECT
    ROUND(
        AVG(end_date - start_date),
        0
    ) AS average_reallocation_days
FROM data_bank.customer_nodes
WHERE end_date <> DATE '9999-12-31';
```

### Result

| Average days |
|---:|
| 14 |

### Insight

Customers are reassigned to another node approximately every two weeks, supporting Data Bank's distributed-security model.

---

## 5. What are the median, 80th and 95th percentiles by region?

### SQL approach

Calculate the number of days in each completed allocation and use `PERCENTILE_CONT()` for each region.

### Query

```sql
WITH allocation_periods AS (
    SELECT
        region_id,
        end_date - start_date AS allocation_days
    FROM data_bank.customer_nodes
    WHERE end_date <> DATE '9999-12-31'
)

SELECT
    r.region_id,
    r.region_name,
    ROUND(
        PERCENTILE_CONT(0.50)
        WITHIN GROUP (
            ORDER BY ap.allocation_days
        )::NUMERIC,
        2
    ) AS median_days,
    ROUND(
        PERCENTILE_CONT(0.80)
        WITHIN GROUP (
            ORDER BY ap.allocation_days
        )::NUMERIC,
        2
    ) AS percentile_80_days,
    ROUND(
        PERCENTILE_CONT(0.95)
        WITHIN GROUP (
            ORDER BY ap.allocation_days
        )::NUMERIC,
        2
    ) AS percentile_95_days
FROM allocation_periods AS ap
INNER JOIN data_bank.regions AS r
    ON ap.region_id = r.region_id
GROUP BY
    r.region_id,
    r.region_name
ORDER BY r.region_id;
```

### Result

Add the values produced by DB Fiddle.

### Interpretation

- The median represents a typical node-allocation duration.
- The 80th percentile shows the period within which 80% of completed allocations ended.
- The 95th percentile highlights longer allocation periods.
- Comparing the percentiles reveals whether reassignment timing is consistent across regions.
