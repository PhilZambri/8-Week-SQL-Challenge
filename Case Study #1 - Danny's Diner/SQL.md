# 🍜 Case Study #1: Danny's Diner 
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***

## Question and Solution

### 📌 1. What is the total amount each customer spent at the restaurant?

````sql
SELECT
  customer_id,
  SUM(price) AS total_spent
FROM dannys_diner.sales
    JOIN dannys_diner.menu USING (product_id)
GROUP BY customer_id
ORDER BY total_spent DESC;
````

#### Steps:
- Use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` tables as `sales.customer_id` and `menu.price` are from both tables.
- Use **SUM** to calculate the total sales contributed by each customer.
- Group the aggregated results by `sales.customer_id`. 

#### Answer:
| customer_id | total_spent |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

- Customer A spent $76.
- Customer B spent $74.
- Customer C spent $36.

***

### 📌 2. How many days has each customer visited the restaurant?

````sql
SELECT
  customer_id,
  COUNT(DISTINCT order_date) AS days_visited
FROM dannys_diner.sales
GROUP BY customer_id;
````

#### Steps:
- To determine the unique number of visits for each customer, utilize **COUNT(DISTINCT `order_date`)**.
- It's important to apply the **DISTINCT** keyword while calculating the visit count to avoid duplicate counting of days. For instance, if Customer A visited the restaurant twice on '2021–01–07', counting without **DISTINCT** would result in 2 days instead of the accurate count of 1 day.

#### Answer:
| customer_id | days_visited |
| ----------- | -----------  |
| A           | 4            |
| B           | 6            |
| C           | 2            |

- Customer A visited 4 times.
- Customer B visited 6 times.
- Customer C visited 2 times.

***

### 📌 3. What was the first item from the menu purchased by each customer?

````sql
WITH ranked_customers AS (
    SELECT 
        *,
        RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS rank
    FROM dannys_diner.sales
    JOIN dannys_diner.menu USING (product_id)
)
SELECT DISTINCT customer_id, product_name AS first_purchased_item
FROM ranked_customers
WHERE rank = 1;
````

````sql
SELECT DISTINCT
    customer_id,
    product_name AS first_purchased_item
FROM sql_challenge.dannys_diner.sales
JOIN sql_challenge.dannys_diner.menu USING (product_id)
QUALIFY RANK() OVER(PARTITION BY customer_id ORDER BY order_date) = 1;
````

#### Steps:
- We notice there can be more than one order on the first date that each customer purchased
- Since there is no finer time detail to determine what was ordered first, we will list all unique menu items purchased on the first day
- We answer this in two different ways with the same results: 1st with a CTE, and 2nd with QUALIFY. 

#### Answer:
| customer_id | first_purchased_item | 
| ----------- | ----------- |
| A           | sushi        |
| A           | curry        |
| B           | curry        | 
| C           | ramen        |

- Customer A's first order was sushi and curry.
- Customer B's first order was curry.
- Customer C's first order was ramen.

***

### 📌 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

````sql
SELECT 
    product_name, 
    COUNT(product_name) AS total_purchased
FROM dannys_diner.sales
	JOIN dannys_diner.menu USING (product_id)
GROUP BY product_name
ORDER BY total_purchased DESC
LIMIT 1;
````

#### Steps:
- Perform a **COUNT** aggregation on the `product_name` column and **ORDER BY** the result in descending order using `total_purchased` field.
- Apply the **LIMIT** 1 clause to filter and retrieve the highest number of purchased items.

#### Answer:
| product_name | total_purchased | 
| ----------- | ----------- |
| ramen       | 8 |


- Most purchased item on the menu is ramen which is 8 times. Yum!

### 📍 What is the most purchased item on the menu and how many times was it purchased total by all customers and by each customer?
#### Alternatively, I provide an answer for the same question but a little more involved.


````sql
WITH most_purchased_item AS(
    SELECT 
        product_id, 
        COUNT(product_id) AS total_purchased
    FROM dannys_diner.sales
    GROUP BY product_id
    ORDER BY total_purchased DESC
    LIMIT 1
)
SELECT 
    COALESCE(customer_id, 'Total') AS customer_id, 
    product_name, 
    COUNT(product_name) AS total_purchases
FROM dannys_diner.sales
	JOIN dannys_diner.menu USING (product_id)
WHERE product_id = (SELECT product_id FROM most_purchased_item)
GROUP BY ROLLUP(customer_id), product_name
ORDER BY customer_id;
````

#### Answer:
| customer_id | product_name | total_purchases | 
| ----------- | ----------- | --- |
| A           | ramen        | 3 |
| B           | ramen        | 2 |
| C           | ramen        | 3 |
| Total       | ramen        | 8 |

***

### 📌 5. Which item was the most popular for each customer?

````sql
WITH ranked AS (
    SELECT 
        customer_id, 
        product_name, 
        COUNT(product_name) AS number_of_orders,
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(product_name) DESC) AS most_ordered_rank
    FROM dannys_diner.sales
        JOIN dannys_diner.menu USING (product_id)
    GROUP BY customer_id, product_name
)

SELECT 
    customer_id, 
    product_name AS most_ordered_product, 
    number_of_orders
FROM ranked
WHERE most_ordered_rank = 1;
````
````sql
SELECT 
    customer_id, 
    product_name, 
    COUNT(product_name) AS number_of_orders 
FROM sql_challenge.dannys_diner.sales
    JOIN sql_challenge.dannys_diner.menu USING (product_id)
GROUP BY customer_id, product_name
QUALIFY RANK() OVER(PARTITION BY customer_id ORDER BY number_of_orders DESC) = 1
ORDER BY customer_id, product_name;
````

*Each user may have more than 1 most ordered item.*

#### Steps:
- Create a CTE named `ranked` and within the CTE, join the `menu` table and `sales` table using the `product_id` column.
- Group results by `sales.customer_id` and `menu.product_name` and calculate the count of `menu.product_id` occurrences for each group. 
- Utilize the **DENSE_RANK()** window function to calculate the ranking of each `sales.customer_id` partition based on the count of orders **COUNT(`sales.customer_id`)** in descending order.
- In the outer query, select the appropriate columns and apply a filter in the **WHERE** clause to retrieve only the rows where the rank column equals 1, representing the rows with the highest order count for each customer.
- Additionally we answer this question another way using QUALIFY

#### Answer:
| customer_id | most_ordered | order_count |
| ----------- | ---------- |------------  |
| A           | ramen        |  3   |
| B           | sushi        |  2   |
| B           | curry        |  2   |
| B           | ramen        |  2   |
| C           | ramen        |  3   |

- Customer A and C's favorite item is ramen.
- Customer B enjoys all items on the menu equally.

***

### 📌 6. Which item was purchased first by the customer after they became a member?

```sql
WITH rankings AS (
    SELECT 
        *,
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS ranked_dates
    FROM dannys_diner.sales
        JOIN dannys_diner.members USING (customer_id)
    WHERE order_date >= join_date
)
SELECT 
    customer_id, 
    product_name, 
    order_date, 
    join_date
FROM rankings
    JOIN dannys_diner.menu USING(product_id)
WHERE ranked_dates = 1
ORDER BY customer_id;
```

````sql
SELECT 
    customer_id,
    product_name,
    order_date,
    join_date
FROM sql_challenge.dannys_diner.sales
    JOIN sql_challenge.dannys_diner.members USING (customer_id)
    JOIN sql_challenge.dannys_diner.menu USING (product_ID)
WHERE order_date >= join_date
QUALIFY DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) = 1;
````

*- We are assuming that an order on the same date as `join_date` is considered to be after becoming a member.*

#### Steps:
- Create a CTE named `rankings` and within the CTE, select the appropriate columns and calculate the rank using the **DENSE_RANK()** window function. The **PARTITION BY** clause divides the data by `members.customer_id` and the **ORDER BY** clause orders the rows within each `members.customer_id` partition by `sales.order_date`.
- Join tables `dannys_diner.members` and `dannys_diner.sales` on `customer_id` column. Additionally, apply a condition to only include sales that occurred *after* the member's `join_date` (`sales.order_date >= members.join_date`).
- In the outer query, join the `joined_as_member` CTE with the `dannys_diner.menu` on the `product_id` column.
- In the **WHERE** clause, filter to retrieve only the rows where the row_num column equals 1, representing the first row within each `customer_id` partition.
- Order result by `customer_id` in ascending order.
- Additionally we answer again using QUALIFY.

#### Answer:
| customer_id | product_name | order_date | join_date |
| ----------- | ---------- | ----------- | ---------- |
| A           | curry        | 2021-01-07 | 2021-01-07 |
| B           | sushi        | 2021-01-11 | 2021-01-09 |

- Customer A's first order as a member is curry.
- Customer B's first order as a member is sushi.

***

### 📌 7. Which item was purchased just before the customer became a member?

````sql
WITH rankings AS (
    SELECT 
        *,
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS ranked_dates
    FROM dannys_diner.sales
        JOIN dannys_diner.members USING (customer_id)
    WHERE order_date < join_date
)
SELECT 
    customer_id, 
    product_name, 
    order_date, 
    join_date
FROM rankings
    JOIN dannys_diner.menu USING(product_id)
WHERE ranked_dates = 1
ORDER BY customer_id;
````

````sql
SELECT 
    customer_id,
    product_name,
    order_date,
    join_date
FROM sql_challenge.dannys_diner.sales
    JOIN sql_challenge.dannys_diner.members USING (customer_id)
    JOIN sql_challenge.dannys_diner.menu USING (product_ID)
WHERE order_date < join_date
QUALIFY DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date DESC) = 1;
````

#### Steps:
- We use the same query from last the last question with two small modifications.
- Changed `WHERE order_date >= join_date` to `WHERE order_date < join_date`
- Within the window function we changed `ORDER BY order_date` to `ORDER BY order_date DESC`

#### Answer:
| customer_id | product_name | order_date | join_date |
| ----------- | ---------- | ----------- | ---------- |
| A           | sushi        | 2021-01-01 | 2021-01-07 |
| A           | curry        | 2021-01-01 | 2021-01-07 |
| B           | sushi        | 2021-01-04 | 2021-01-09 |

- Customer A ordered both sushi and curry on the first date before becoming a member.
- Customer B ordered sushi on the first date before becoming a member.

***

### 📌 8. What is the total items and amount spent for each member before they became a member?

```sql
SELECT 
    customer_id, 
    COUNT(product_name) AS total_items, 
    SUM(price) AS total_spent
FROM dannys_diner.sales
    JOIN dannys_diner.members USING (customer_id)
    JOIN dannys_diner.menu USING (product_ID)
WHERE order_date < join_date
GROUP BY customer_id
ORDER BY customer_id;
```

#### Steps:
- Join the tables: `dannys_diner.sales`, `dannys_diner.members`, `dannys_diner.menu`.
- Filter for where  `sales.order_date < members.join_date`
- Group the results by `sales.customer_id`.
- Select `customer_id`, and aggregate using `COUNT(product_name)`, and `SUM(price)`
- Order the result by `sales.customer_id` in ascending order.

#### Answer:
| customer_id | total_items | total_spent |
| ----------- | ---------- |----------  |
| A           | 2 |  25       |
| B           | 3 |  40       |

Before becoming members,
- Customer A spent $25 on 2 items.
- Customer B spent $40 on 3 items.

***

### 📌 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?

```sql
WITH get_points AS (
    SELECT
        customer_id,
        CASE WHEN product_id = 1 THEN price * 20  ELSE price * 10 END AS points
    FROM dannys_diner.sales
        JOIN dannys_diner.menu USING (product_ID)
)
SELECT 
    customer_id, 
    SUM(points) AS total_points
FROM get_points
GROUP BY customer_id
ORDER BY customer_id;
```

#### Steps:
Let's break down the question to understand the point calculation for each customer's purchases.
- Each $1 spent = 10 points. However, `product_id` 1 sushi gets 2x points, so each $1 spent = 20 points.
- Here's how the calculation is performed using a conditional CASE statement:
	- If product_id = 1, multiply every $1 by 20 points.
	- Otherwise, multiply $1 by 10 points.
- Then, calculate the total points for each customer.

#### Answer:
| customer_id | total_points | 
| ----------- | ---------- |
| A           | 860 |
| B           | 940 |
| C           | 360 |

- Total points for Customer A is 860.
- Total points for Customer B is 940.
- Total points for Customer C is 360.

***

### 📌 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?

```sql
WITH get_points AS (
    SELECT 
        customer_id,
        CASE
			WHEN product_id = 1 OR (order_date >= join_date AND order_date < join_date + 7) THEN price * 20
			ELSE price * 10
		END AS points
    FROM dannys_diner.sales
        JOIN dannys_diner.menu USING (product_ID)
        JOIN dannys_diner.members USING (customer_id)
    WHERE order_date >= '2021-01-01' AND order_date < '2021-02-01'
)
SELECT 
    customer_id, 
    SUM(points) AS total_points
FROM get_points
GROUP BY customer_id
ORDER BY customer_id;
```

#### Assumptions:
- each $1 spent equates to 10 points and sushi still has a 2x points multiplier
- the first week after a customer joins the program (including their join date) they earn 2x points on all items

#### Steps:
- Join the tables `sales`, `menu`, `members`
- Filter for `order_date` in January
- Calculate the total points based on the requirements by using a case statement: `product_id = 1` or `(order_date >= join_date AND order_date < join_date + 7`
- Place this query into a CTE called `get_points`
- In the main query, we `GROUP BY customer_id` and `SUM(points) AS total_points`

#### Answer:
| customer_id | total_points | 
| ----------- | ---------- |
| A           | 1370 |
| B           | 820 |

- Total points for Customer A is 1,370.
- Total points for Customer B is 820.

***

## BONUS QUESTIONS

### 📌 Join All The Things

**Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)**

```sql
SELECT 
    customer_id, order_date, product_name, price,
    CASE WHEN order_date >= join_date THEN 'Y' ELSE 'N' END AS member
FROM dannys_diner.sales
    JOIN dannys_diner.menu USING (product_ID)
    LEFT JOIN dannys_diner.members USING (customer_id)
ORDER BY customer_id, order_date;
```
#### Steps:
- Join the tables `sales`, `menu`, `members`, ensuring we use a `LEFT JOIN` on members as to not exclude non-members
- Use a CASE statment to generate the members column `CASE WHEN order_date >= join_date THEN 'Y' ELSE 'N' END AS member`

#### Answer: 
| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | -------------| ----- | ------ |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

***

### 📌 Rank All The Things

**Danny also requires further information about the ```ranking``` of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ```ranking``` values for the records when customers are not yet part of the loyalty program.**

```sql
WITH is_member AS (
    SELECT 
        customer_id, order_date, product_name, price,
        CASE WHEN order_date >= join_date THEN 'Y' ELSE 'N' END AS member
    FROM dannys_diner.sales
        JOIN dannys_diner.menu USING (product_ID)
        LEFT JOIN dannys_diner.members USING (customer_id)
)
SELECT 
    *,
    CASE 
        WHEN member = 'N' THEN NULL 
        ELSE DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date) 
    END AS ranking
FROM is_member
ORDER BY customer_id, order_date;
```

#### Steps:
- We wrap the previous Query from the first Bonus Question into a CTE named `is_member`
- In the main query we create another case statement that uses a window function the generate the rankings: `WHEN member = 'N' THEN NULL` `ELSE DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)`

#### Answer: 
| customer_id | order_date | product_name | price | member | ranking | 
| ----------- | ---------- | -------------| ----- | ------ |-------- |
| A           | 2021-01-01 | sushi        | 10    | N      | NULL
| A           | 2021-01-01 | curry        | 15    | N      | NULL
| A           | 2021-01-07 | curry        | 15    | Y      | 1
| A           | 2021-01-10 | ramen        | 12    | Y      | 2
| A           | 2021-01-11 | ramen        | 12    | Y      | 3
| A           | 2021-01-11 | ramen        | 12    | Y      | 3
| B           | 2021-01-01 | curry        | 15    | N      | NULL
| B           | 2021-01-02 | curry        | 15    | N      | NULL
| B           | 2021-01-04 | sushi        | 10    | N      | NULL
| B           | 2021-01-11 | sushi        | 10    | Y      | 1
| B           | 2021-01-16 | ramen        | 12    | Y      | 2
| B           | 2021-02-01 | ramen        | 12    | Y      | 3
| C           | 2021-01-01 | ramen        | 12    | N      | NULL
| C           | 2021-01-01 | ramen        | 12    | N      | NULL
| C           | 2021-01-07 | ramen        | 12    | N      | NULL

***
