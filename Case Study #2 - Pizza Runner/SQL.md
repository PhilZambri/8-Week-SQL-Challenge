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

## 📰 Table: customer_orders
Starting with the `customer_orders` table, let's take a look at the data.
- All the datatypes look good - no changes needed.
- Looking at the data, its clear there are various values of null, empty strings, and possibly blank spaces in the `exclusions` and `extras` columns.
- We know that `COUNT(column_name)` skips over `NULL` values. Therefore, when we count, we would expect to get less than the 14 total rows in the table. However, we get 14 and 13 rows each, telling us something else is going on here.
- For further information, we get the `char_length` of each column. We notice a character length of 4 for the null values. This means almost all the nulls we see in the data are not real nulls but actually 4-char length strings - 'null'. 

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

Now we have to decide what we should do with these values. Should we transform them all into nulls? into empty strings? into a particular value like Unknown, None, or 0?
- since both columns are strings, any of these options will work.
- I am opting for the `NULL` values option.
- `NULL` values take minimal to no space, and more importantly `NULL` values have the best query performance.
- We can always transform the `NULL` values after computations for a cleaner, more user-friendly output display.

Let's proceed with the cleaning process:
- We could use temporary tables, normal tables, or materialized views.
- If this were a production environment, new orders and raw data would be arriving everyday. It would make sense for the cleaned tables to be refreshed every so often - probably daily.
- Therefore, We will use materialized views to store the cleaned data.
- We will utilize `TRIM()`, `replace()`, and `NULLIF()` functions.
- `TRIM()` will remove any leading and trailing white spaces. (There aren't any in the current data, but could be if data gets added.)
- We use `replace()` to transform the 4-character strings 'null' with the empty string ''.
- `NULLIF()` transforms all the empty strings to `NULL` values.

As I was working through the questions, I ran into an additional problem that I will address and solve here.
- There is no way to uniquely identify when a customer orders the exact same pizza more than once in the same order.
- In our example, we have duplicate rows at row 5 and 6. The customer ordered 2 of the same pizza.
- Ordering 2 or more of the same pizza is very common, so we don't want to remove duplicates.
- What we can do is combine them into a `quantity` column. This removes the duplicate rows without losing data.
- This will allow us to get the full list of topping names for each order without problems in a later question.

```SQL
-- Cleaned customer_orders table

CREATE TEMP TABLE customer_orders_temp AS
WITH customer_orders_cte AS (
    SELECT 
        order_id, 
        customer_id, 
        pizza_id,
        NULLIF(replace(TRIM(exclusions), 'null', ''), '') AS exclusions,
        NULLIF(replace(TRIM(extras), 'null', ''), '') AS extras,
        order_time
    FROM pizza_runner.customer_orders
)

SELECT *, COUNT(*) AS quantity
FROM customer_orders_cte
GROUP BY 1, 2, 3, 4, 5, 6
ORDER BY order_id, pizza_id;
```
```SQL
-- View and verify the data
SELECT *, char_length(exclusions) AS exclusions_length, char_length(extras) AS extras_length
FROM pizza_runner.customer_orders_temp;

-- Verify the nulls
SELECT COUNT(exclusions) AS exclusions_count, COUNT(extras) AS extras_count
FROM pizza_runner.customer_orders_temp;
```
<img width="1309" height="364" alt="image" src="https://github.com/user-attachments/assets/bc0e8279-f294-495a-b3ce-0c89dae2f240" />
<img width="373" height="54" alt="image" src="https://github.com/user-attachments/assets/14dff559-37af-47b0-84fb-a1f92831d039" />

***

## 📰 Table: runner_orders
Now, let's take a look at the `runner_orders` table.
- lala

```SQL
-- View the data
SELECT * FROM pizza_runner.runner_orders;

-- Check the data-types
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'runner_orders';

-- Count the non-nulls
SELECT 
    COUNT(pickup_time) AS time_count, 
    COUNT(distance) AS distance_count, 
    COUNT(duration) AS duration_count, 
    COUNT(cancellation) AS cancel_count 
FROM pizza_runner.runner_orders;
```
<img width="874" height="286" alt="image" src="https://github.com/user-attachments/assets/af5b6cc4-a142-4eb4-aded-db07a2a112a9" />
<img width="317" height="185" alt="image" src="https://github.com/user-attachments/assets/3039b5c5-5f92-4d1e-b274-e30c04e7b3f3" />
<img width="668" height="55" alt="image" src="https://github.com/user-attachments/assets/99473115-55be-42ac-82b3-e345d6eb25b5" />

more speaky here
- lala
```SQL
-- Cleaned runner_orders table

CREATE TEMP TABLE runner_orders_temp AS
SELECT 
    order_id,
    runner_id,
    NULLIF(pickup_time, 'null')::timestamp AS pickup_time,
    NULLIF(regexp_replace(distance, '[^0-9.]', '', 'g'), '')::FLOAT AS "distance(km)",
    NULLIF(regexp_replace(duration, '[^0-9]', '', 'g'), '')::INT AS "duration(minutes)",
    NULLIF(replace(TRIM(cancellation), 'null', ''), '')::VARCHAR(50) AS cancellation
FROM pizza_runner.runner_orders;
```
```SQL
-- View the cleaned data
SELECT * FROM pizza_runner.runner_orders_temp;

-- Check the data-types
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'runner_orders_temp';

-- Count the non-nulls
SELECT 
    COUNT(pickup_time) AS time_count, 
    COUNT("distance(km)") AS distance_count, 
    COUNT("duration(minutes)") AS duration_count, 
    COUNT(cancellation) AS cancel_count 
FROM pizza_runner.runner_orders_temp;
```
<img width="969" height="285" alt="image" src="https://github.com/user-attachments/assets/9c9d6c4e-ef60-41e2-aaf8-d5f2ff41d3ba" />
<img width="394" height="184" alt="image" src="https://github.com/user-attachments/assets/7341fec2-aea5-42e8-b6ab-cc74b0c35146" />
<img width="668" height="54" alt="image" src="https://github.com/user-attachments/assets/2162bd5f-203a-49c1-94c1-bef973a25ec0" />




