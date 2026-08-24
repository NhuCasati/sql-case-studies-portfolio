# Danny's Diner: Customer and Loyalty Analysis

## Project Overview

This case study analyses restaurant transaction data using PostgreSQL.

The objective is to understand customer spending, visit frequency, menu
preferences and membership behaviour. The analysis also evaluates the
restaurant's loyalty programme.

This project is based on Case Study 1 of the
[8 Week SQL Challenge](https://8weeksqlchallenge.com/case-study-1/)
created by Data With Danny.

## Business Questions

1. What is the total amount each customer spent?
2. How many days did each customer visit the restaurant?
3. What was the first menu item purchased by each customer?
4. Which menu item was purchased most frequently?
5. What was the most popular item for each customer?
6. Which item was purchased first after becoming a member?
7. Which item was purchased immediately before becoming a member?
8. How many items did members purchase before joining, and how much did they spend?
9. How many loyalty points did each customer earn?
10. How did the first-week membership bonus affect customer points?

## Dataset

The database contains three tables:

| Table | Description |
|---|---|
| `sales` | Customer purchase transactions |
| `menu` | Menu items and prices |
| `members` | Customer membership dates |

## Tools and Techniques

- PostgreSQL
- DB Fiddle
- Table joins
- Aggregate functions
- Common Table Expressions
- Window functions
- Conditional logic
- Date analysis

## Project Files

- [`schema.sql`](./schema.sql) — database creation and sample data
- [`solutions.sql`](./solutions.sql) — SQL queries for all questions

## Analysis Status

The database schema and detailed SQL analysis are currently in progress.
