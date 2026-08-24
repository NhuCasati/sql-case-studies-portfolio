# Pizza Runner: Operations and Delivery Analysis

## Project Overview

Pizza Runner is a pizza delivery business that uses freelance runners to deliver customer orders.

This case study uses PostgreSQL to clean and analyse order, delivery, runner and ingredient data. The objective is to understand ordering behaviour, delivery performance, customer preferences and business profitability.

This project is based on Case Study 2 of the [8 Week SQL Challenge](https://8weeksqlchallenge.com/case-study-2/) created by Data With Danny.

---

## Table of Contents

- [Business Task](#business-task)
- [Dataset](#dataset)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Data Cleaning and Transformation](#data-cleaning-and-transformation)
- [A. Pizza Metrics](#a-pizza-metrics)
- [B. Runner and Customer Experience](#b-runner-and-customer-experience)
- [C. Ingredient Optimisation](#c-ingredient-optimisation)
- [D. Pricing and Ratings](#d-pricing-and-ratings)
- [Key Findings](#key-findings)
- [Assumptions and Limitations](#assumptions-and-limitations)
- [What I Learned](#what-i-learned)

---

## Business Task

Danny wants to understand how Pizza Runner is performing and identify opportunities to improve the business.

The analysis focuses on:

- Customer ordering behaviour
- Successful and cancelled deliveries
- Runner performance
- Delivery speed and distance
- Pizza customisations
- Ingredient usage
- Revenue and profitability

---

## Dataset

The database contains the following tables:

| Table | Description |
|---|---|
| `runners` | Runner registration information |
| `customer_orders` | Customer pizza orders and customisations |
| `runner_orders` | Delivery, distance, duration and cancellation information |
| `pizza_names` | Pizza names |
| `pizza_recipes` | Standard ingredients for each pizza |
| `pizza_toppings` | Pizza topping names |

---

## Entity Relationship Diagram

The entity relationship diagram will be added here.

---

## Data Cleaning and Transformation

The original dataset contains inconsistent values such as:

- Blank values
- Text values containing `null`
- Distances containing `km`
- Durations containing `minutes`, `minute` or `mins`
- Columns stored using unsuitable data types

The data must be cleaned before completing the analysis.

---

# A. Pizza Metrics

Questions and solutions related to pizza orders will be added here.

---

# B. Runner and Customer Experience

Questions and solutions related to runners, deliveries and customer experience will be added here.

---

# C. Ingredient Optimisation

Questions and solutions related to recipes, exclusions, extras and topping usage will be added here.

---

# D. Pricing and Ratings

Questions and solutions related to revenue, costs, ratings and profitability will be added here.

---

## Key Findings

Key findings will be added after completing the analysis.

---

## Assumptions and Limitations

Assumptions and data limitations will be documented here.

---

## What I Learned

The technical and analytical skills demonstrated in this project will be summarised here.

---

## Source

The original case study and dataset were created by [Data With Danny](https://8weeksqlchallenge.com/case-study-2/).

All SQL queries, explanations and interpretations in this repository are part of my own analysis.
