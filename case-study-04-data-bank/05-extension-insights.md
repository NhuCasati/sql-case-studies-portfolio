# Extension Insights

[← Back to Data Bank overview](./README.md)

## Investor and Customer Headlines

### Global node coverage

Data Bank operates five secure nodes across every region represented in the dataset.

### Frequent customer reassignment

Customers are moved between nodes approximately every two weeks, reducing the amount of time customer funds and data remain in one location.

### Distributed infrastructure

The node network combines regional availability with frequent reassignment, supporting Data Bank's security-focused operating model.

### Active customer base

The dataset includes 500 customers conducting thousands of deposits, purchases and withdrawals.

### Strong deposit activity

Deposits were the most frequent transaction type and represented more than $1.35 million in transaction value.

---

## Data-Provisioning Options

| Model | Responsiveness | Stability | Complexity | Recommended use |
|---|---|---|---|---|
| Previous month-end | Low | High | Low | Simple initial implementation |
| Previous 30-day average | Medium | High | Medium | Recommended operating model |
| Real-time | High | Low | High | Premium or future implementation |
| Interest-adjusted | High | Medium | High | Customer-reward programme |

---

## Recommended Model

The previous 30-day average model is the strongest initial choice.

### Reasons

- Reduces the effect of unusually large individual transactions
- Produces more stable storage requirements
- Responds more quickly than a monthly allocation model
- Requires less infrastructure than real-time allocation
- Supports more reliable capacity forecasting

---

## Suggested Presentation Content

### Slide 1: Secure Global Network

Include:

- Number of customers
- Number of nodes
- Number of regions
- Average reassignment duration
- Regional customer distribution

### Slide 2: Transaction Activity

Include:

- Transaction counts by type
- Total value by type
- Average deposits per customer
- Average deposit amount
- Percentage of customers with meaningful balance growth

### Slide 3: Storage Allocation Decision

Compare:

- Previous month-end allocation
- Previous 30-day average
- Real-time allocation
- Interest-adjusted allocation

Use a simple table showing:

- Required storage
- Volatility
- Implementation complexity
- Operational recommendation

---

## Final Recommendation

Data Bank should begin with the trailing 30-day average allocation model and use the real-time model as a longer-term development goal.

The company should also monitor:

- Customer growth
- Average customer balances
- Regional customer distribution
- Storage utilisation
- Monthly peak storage demand
- Frequency of negative balances
- Infrastructure cost per customer
