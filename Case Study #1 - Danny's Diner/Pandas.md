# 🍜 Case Study #1: Danny's Diner 
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Setup for Pandas using sqlalchemy](#setup-for-pandas-using-sqlalchemy)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***
## Setup for Pandas using SQLAlchemy
Using SQLAlchemy, we can query the local postgresql database named '8 WeekSQLChallenge', to extract all three tables - `sales`, `menu`, and `members` - 
from dannys_diner into three Pandas Dataframes. We will use these dataframes to answer all of the following case study questions.

```python
import pandas as pd
from sqlalchemy import create_engine

connection_string = f"postgresql+psycopg://user:pass@localhost:5432/8 WeekSQLChallenge"        
engine = create_engine(connection_string)

sales = pd.read_sql(
    sql="SELECT * FROM dannys_diner.sales",
    con=engine.connect(),
	parse_dates='order_date'
)
menu = pd.read_sql(
    sql="SELECT * FROM dannys_diner.menu",
    con=engine.connect()
)
members = pd.read_sql(
    sql="SELECT * FROM dannys_diner.members",
    con=engine.connect(),
	parse_dates='join_date'
)
```

***

## Question and Solution

### 📌 1. What is the total amount each customer spent at the restaurant?

````python
sales_menu = sales.merge(menu, on='product_id')

sales_menu = sales_menu.groupby('customer_id').agg(total_spent=('price', 'sum'))     
display(sales_menu)
````

#### Steps:
- Use **MERGE** to join `dannys_diner.sales` and `dannys_diner.menu` tables on `product_id` as we need the price information for this.
- Groupby `customer_id` and aggregate by getting the **SUM** of `price` in a column labeled `total_spent`

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

````python
days_visited = sales.groupby('customer_id').agg(days_visited=('order_date', 'nunique'))
display(days_visited)
````

#### Steps:
- To determine the unique number of visits for each customer, utilize `nunique`.
- Groupby `customer_id` and aggregate using `nunique` on `order_date` in a column labeled `days_visited`

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

````python
sales_menu = sales.merge(menu, on='product_id')

sales_menu['order_date_rank'] = sales_menu.groupby('customer_id')['order_date'].rank(method='dense')
first_purchased_items = sales_menu.loc[sales_menu['order_date_rank'] == 1, ['customer_id', 'product_name']].drop_duplicates()
display(first_purchased_items)
````

#### Steps:
- Use `merge` to join `sales` and `menu` on `product_id`
- In order to respect duplicates(where more than one order took place on the same day) we will utilize the `rank` function.
- We `groupby` `customer_id`, calculate the `dense rank` on `ascending order date`, and save into a new column `order_date_rank`.
- Then we simply filter the rows for where `order_date_rank = 1` and just select the `customer_id` and `product_name` columns.

#### Answer:
| customer_id | product_name | 
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

````python
sales_menu = sales.merge(menu, on='product_id')

most_purchased_item = sales_menu.value_counts('product_name').head(1).reset_index().rename(columns={'count':'total_purchased'})
display(most_purchased_item)
````

#### Steps:
- Use `merge` to join `sales` and `menu` on `product_id`
- Here we utilize the `value_counts` function on the `product_name` column
- By default, `value_counts` automatically sorts in descending order. Therefore, we simply use `head(1)` to get the most purchased item. 

#### Answer:
| product_name | total_purchased | 
| ----------- | ----------- |
| ramen       | 8 |


- Most purchased item on the menu is ramen which is 8 times. Yum!

### 📍 What is the most purchased item on the menu and how many times was it purchased total by all customers and by each customer?
#### Alternatively, I provide an answer for the same question but a little more involved.

````python
most_purchased_id = sales.value_counts('product_id').index[0]

x = (sales.loc[sales['product_id'] == most_purchased_id]
     .merge(menu, on='product_id')
     .groupby(['customer_id', 'product_name']).agg(total_purchased=('product_name', 'count')).reset_index())
x.loc['Total'] = {'customer_id':'All Customers', 'product_name':x.iat[0,1], 'total_purchased':x['total_purchased'].sum()}
display(x)
````

#### Answer:
|-| customer_id | product_name | total_purchased | 
|- | ----------- | ----------- | --- |
|- | A           | ramen        | 3 |
|- | B           | ramen        | 2 |
|- | C           | ramen        | 3 |
|Total| All Customers       | ramen        | 8 |

***

### 📌 5. Which item was the most popular for each customer?

````python
item_total_purchases = sales.groupby(['customer_id', 'product_id']).agg(total_purchased=('product_id', 'count')).reset_index()
item_total_purchases['total_purchased_rank'] = item_total_purchases.groupby('customer_id')['total_purchased'].rank(method='dense', ascending=False)

item_total_purchases = item_total_purchases.merge(menu, on='product_id').loc[item_total_purchases['total_purchased_rank'] == 1, ['customer_id', 'product_name', 'total_purchased']]
display(item_total_purchases)
````

*Each user may have more than 1 most ordered item.*

#### Steps:
- First we get the total purchases of each product for each customer. Groupby `'customer_id', 'product_id'` and count.
- Next, we create `total_purchased_rank` column by using a descending dense rank on `total_purchased` grouped by `customer_id`.
- Finally we merge the resulting frame with menu, and simply filter for where `item_total_purchases['total_purchased_rank'] == 1`.

#### Answer:
| customer_id | most_ordered | order_count |
| ----------- | ---------- |------------  |
| A           | ramen        |  3   |
| B           | sushi        |  2   |
| B           | curry        |  2   |
| B           | ramen        |  2   |
| C           | ramen        |  3   |

- Customer A and C's favourite item is ramen.
- Customer B enjoys all items on the menu.

***

### 📌 6. Which item was purchased first by the customer after they became a member?

```python

```

#### Steps:
- Create a CTE named `joined_as_member` and within the CTE, select the appropriate columns and calculate the row number using the **ROW_NUMBER()** window function. The **PARTITION BY** clause divides the data by `members.customer_id` and the **ORDER BY** clause orders the rows within each `members.customer_id` partition by `sales.order_date`.
- Join tables `dannys_diner.members` and `dannys_diner.sales` on `customer_id` column. Additionally, apply a condition to only include sales that occurred *after* the member's `join_date` (`sales.order_date > members.join_date`).
- In the outer query, join the `joined_as_member` CTE with the `dannys_diner.menu` on the `product_id` column.
- In the **WHERE** clause, filter to retrieve only the rows where the row_num column equals 1, representing the first row within each `customer_id` partition.
- Order result by `customer_id` in ascending order.

#### Answer:
| customer_id | product_name |
| ----------- | ---------- |
| A           | ramen        |
| B           | sushi        |

- Customer A's first order as a member is ramen.
- Customer B's first order as a member is sushi.

***

### 📌 7. Which item was purchased just before the customer became a member?

````python

````

#### Steps:
- Create a CTE called `purchased_prior_member`. 
- In the CTE, select the appropriate columns and calculate the rank using the **ROW_NUMBER()** window function. The rank is determined based on the order dates of the sales in descending order within each customer's group.
- Join `dannys_diner.members` table with `dannys_diner.sales` table based on the `customer_id` column, only including sales that occurred *before* the customer joined as a member (`sales.order_date < members.join_date`).
- Join `purchased_prior_member` CTE with `dannys_diner.menu` table based on `product_id` column.
- Filter the result set to include only the rows where the rank is 1, representing the earliest purchase made by each customer before they became a member.
- Sort the result by `customer_id` in ascending order.

#### Answer:
| customer_id | product_name |
| ----------- | ---------- |
| A           | sushi        |
| B           | sushi        |

- Both customers' last order before becoming members are sushi.

***

### 📌 8. What is the total items and amount spent for each member before they became a member?

```python

```

#### Steps:
- Select the columns `sales.customer_id` and calculate the count of `sales.product_id` as total_items for each customer and the sum of `menu.price` as total_sales.
- From `dannys_diner.sales` table, join `dannys_diner.members` table on `customer_id` column, ensuring that `sales.order_date` is earlier than `members.join_date` (`sales.order_date < members.join_date`).
- Then, join `dannys_diner.menu` table to `dannys_diner.sales` table on `product_id` column.
- Group the results by `sales.customer_id`.
- Order the result by `sales.customer_id` in ascending order.

#### Answer:
| customer_id | total_items | total_sales |
| ----------- | ---------- |----------  |
| A           | 2 |  25       |
| B           | 3 |  40       |

Before becoming members,
- Customer A spent $25 on 2 items.
- Customer B spent $40 on 3 items.

***

### 📌 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?

```python

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

- Total points for Customer A is $860.
- Total points for Customer B is $940.
- Total points for Customer C is $360.

***

### 📌 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?

```python

```

#### Assumptions:
- On Day -X to Day 1 (the day a customer becomes a member), each $1 spent earns 10 points. However, for sushi, each $1 spent earns 20 points.
- From Day 1 to Day 7 (the first week of membership), each $1 spent for any items earns 20 points.
- From Day 8 to the last day of January 2021, each $1 spent earns 10 points. However, sushi continues to earn double the points at 20 points per $1 spent.

#### Steps:
- Create a CTE called `dates_cte`. 
- In `dates_cte`, calculate the `valid_date` by adding 6 days to the `join_date` and determine the `last_date` of the month by subtracting 1 day from the last day of January 2021.
- From `dannys_diner.sales` table, join `dates_cte` on `customer_id` column, ensuring that the `order_date` of the sale is after the `join_date` (`dates.join_date <= sales.order_date`) and not later than the `last_date` (`sales.order_date <= dates.last_date`).
- Then, join `dannys_diner.menu` table based on the `product_id` column.
- In the outer query, calculate the points by using a `CASE` statement to determine the points based on our assumptions above. 
    - If the `product_name` is 'sushi', multiply the price by 2 and then by 10. For orders placed between `join_date` and `valid_date`, also multiply the price by 2 and then by 10. 
    - For all other products, multiply the price by 10.
- Calculate the sum of points for each customer.

#### Answer:
| customer_id | total_points | 
| ----------- | ---------- |
| A           | 1020 |
| B           | 320 |

- Total points for Customer A is 1,020.
- Total points for Customer B is 320.

***

## BONUS QUESTIONS

### 📌 Join All The Things

**Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)**

```python

```
 
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

```python

```

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
