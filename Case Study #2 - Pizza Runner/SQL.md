# 🍕 Case Study #2 Pizza Runner

<img src="https://user-images.githubusercontent.com/81607668/127271856-3c0d5b4a-baab-472c-9e24-3c1e3c3359b2.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- Solution
  - [Data Cleaning and Transformation](#-data-cleaning--transformation)
  - [A. Pizza Metrics](#a-pizza-metrics)
  - [B. Runner and Customer Experience](#b-runner-and-customer-experience)
  - [C. Ingredient Optimisation](#c-ingredient-optimisation)
  - [D. Pricing and Ratings](#d-pricing-and-ratings)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-2/).

***

## Business Task
Danny is expanding his new Pizza Empire and at the same time, he wants to Uberize it, so Pizza Runner was launched!

Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers. 

## Entity Relationship Diagram

![Pizza Runner](https://github.com/katiehuangx/8-Week-SQL-Challenge/assets/81607668/78099a4e-4d0e-421f-a560-b72e4321f530)

*I have loaded all the data for this case study into a local postgresql database.*  
*All of the data querying and transformations will be through this local database.*

## 🧼 Data Cleaning & Transformation
This case study contains 6 tables - `customer_orders`, `pizza_names`, `pizza_recipes`, `pizza_toppings`, `runner_orders`, and `runners`.  
Before we start answering questions, the two tables `customer_orders` and `runner_orders` contain messy data that we will need to tidy-up.  
Let's get started!

### 🔨 Table: customer_orders
Starting with the `customer_orders` table, let's take a look at the data.
- All the datatypes look good - no changes needed.
- Looking at the data, its clear there are various values of null, empty strings, and possibly blank spaces in the `exclusions` and `extras` columns.
- Viewing the count of each column, we see it counted 14 and 13 rows each. That means it counted all rows except 1. We know that `COUNT(column_name)` skips over `NULL` values, therefore something else is going on here.
- Next, we get the `char_length` of each column. We notice a character length of 4 for the null values. This means almost all the nulls we see in the data are not real nulls but actually 4-char length strings - 'null'. 

```SQL
-- View the data
SELECT * FROM pizza_runner.customer_orders;

-- Check the data types
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'customer_orders';

-- Count the non-nulls
SELECT COUNT(exclusions), COUNT(extras)
FROM pizza_runner.customer_orders;

-- Verify nulls are actually 4-char strings
SELECT exclusions, char_length(exclusions) AS exclusions_length, extras, char_length(extras) AS extras_length
FROM pizza_runner.customer_orders;

```

<img width="827" height="390" alt="image" src="https://github.com/user-attachments/assets/ccfba1b2-eae9-4bde-acc1-c673a1fee33d" />
<img width="392" height="182" alt="image" src="https://github.com/user-attachments/assets/9c9c2cc8-e570-4854-bc49-d91385e77a5b" />
<img width="376" height="58" alt="image" src="https://github.com/user-attachments/assets/79a1881d-b2a2-4ce1-ada5-72bbe30c7c88" />
<img width="636" height="393" alt="image" src="https://github.com/user-attachments/assets/776930fc-7fc3-4573-8dfa-56225ef183c3" />


