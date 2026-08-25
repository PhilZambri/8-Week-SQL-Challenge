# 🍜 Case Study #1: Danny's Diner 
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Setup for PySpark using Databricks tables](#setup-for-pyspark-using-databricks-tables)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***
## Setup for PySpark using Databricks tables
- First we create the directory on `Databricks` using the `catalog.schema.tablename` notation. 
- `sql_challenge` is the name of the catalog, and `dannys_diner` is the name of the schema.
- Using the provided scripts, we created the `sales, menu and members` tables within `dannys_diner`.
- Now that the tables are all setup we create our `SparkSession`.
- Then we simply use the `spark.read_table()` function using `sql_challenge.dannys_diner.table_name` to read each table into a dataframe.
- Finally we use `.show()` and `.printSchema` to verify both the data and the datatypes.

```python
from pyspark.sql import SparkSession
from pyspark.sql.window import Window
import pyspark.sql.functions as sf

spark = SparkSession.builder.getOrCreate()

sales = spark.read.table("sql_challenge.dannys_diner.sales")
menu = spark.read.table("sql_challenge.dannys_diner.menu")
members = spark.read.table("sql_challenge.dannys_diner.members")

sales.show()    ; sales.printSchema()
menu.show()     ; menu.printSchema()
members.show()  ; members.printSchema()
```

***

## Question and Solution

### 📌 1. What is the total amount each customer spent at the restaurant?

````python
total_spent = (sales.join(menu, on='product_id')
                .groupBy('customer_id')
                .agg(sf.sum('price').alias('total_spent'))
                .sort('customer_id'))
                
total_spent.show()
````

#### Steps:
- Join `sales` with the `menu` table on `product_id`.
- GroupBy `customer_id` and calculate the `sum` of `price` into a column named `total_spent`.
- Finally, we sort by `customer_id`.

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
days_visited = (sales.groupBy('customer_id')
                .agg(sf.countDistinct('order_date').alias('days_visited'))
                .sort('customer_id'))

days_visited.show()
````

#### Steps:
- Group_by `customer_id`.
- Create the `days_visited` column by using the `countDistinct` function on the `order_date` column.
- Sort by `customer_id`.

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
first_order = (sales.join(menu, on='product_id')
               .withColumn('rank', sf.dense_rank().over(Window.partitionBy('customer_id').orderBy('order_date')))
               .filter(sf.col('rank') == 1)
               .select('customer_id', sf.col('product_name').alias('first_order'))
               .distinct())

first_order.show()
````

#### Steps:
- Join `sales` and `menu` on `product_id`.
- In order to respect duplicates(where more than one order took place on the same day) we will utilize the `dense_rank` function.
- We calculate the `dense_rank` on ascending `order_date`, grouped by `customer_id` and save into a new column named `rank`.
- Then we filter the rows for where `rank = 1` and select just the unique `customer_id` and `product_name` columns.

#### Answer:
| customer_id | first_order | 
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
most_purchased = (sales.join(menu, on='product_id')
                  .groupBy('product_name')
                  .agg(sf.count('product_name').alias('total_purchased'))
                  .sort(sf.desc('total_purchased')))

most_purchased.show(1)
````

#### Steps:
- Join `sales` and `menu` on `product_id`.
- Group_by `product_name` and use `count()` to count the number of each product purchased as a column named `total_purchased`.
- We sort by descending `total_purchased` and choose the first row. 

#### Answer:
| product_name | total_purchased | 
| ----------- | ----------- |
| ramen       | 8 |


- Most purchased item on the menu is ramen which is 8 times. Yum!

### 📍 What is the most purchased item on the menu and how many times was it purchased total by all customers and by each customer?
#### Alternatively, I provide an answer for the same question but a little more involved.

#### Steps:
- First we obtain the `product_id` of the most purchased item and store into `most_purcahsed_item`.
- Then we filter for the most purchased item, groupBy `customer_id` and use `count` to get the total ordered for each customer.
- Finally we create the overall totals row by creating a new Dataframe, and appending to `most_purchased`.

````python
most_purchased_item = (sales
                       .groupBy('product_id')
                       .agg(sf.count('product_id').alias('total_purchased'))
                       .sort(sf.desc('total_purchased'))
                       .first()['product_id'])

most_purchased = (sales
                  .filter(sf.col('product_id') == most_purchased_item)
                  .join(menu, on='product_id')
                  .groupBy('customer_id', 'product_name')
                  .agg(sf.count('product_name').alias('total_purchased'))
                  .sort('customer_id'))

total = spark.createDataFrame([('All Customers', 
                                most_purchased.first()['product_name'], 
                                most_purchased.select(sf.sum('total_purchased')).collect()[0][0])], 
                              schema="customer_id string, product_name string, total_purchased long")

most_purchased.union(total).show()
````

#### Answer:
| customer_id | product_name | total_purchased | 
| ----------- | ----------- | --- |
| A           | ramen        | 3 |
| B           | ramen        | 2 |
| C           | ramen        | 3 |
| All Customers       | ramen        | 8 |

***

### 📌 5. Which item was the most popular for each customer?

````python
most_purchased = (sales
                  .join(menu, on='product_id')
                  .groupBy('customer_id', 'product_name')
                  .agg(sf.count('product_name').alias('total_purchased'))
                  .withColumn('rank', sf.dense_rank().over(Window.partitionBy('customer_id').orderBy(sf.desc('total_purchased'))))
                  .filter(sf.col('rank') == 1)
                  .drop('rank')
                  .sort('customer_id'))

most_purchased.show()
````

*Each user may have more than 1 most ordered item.*

#### Steps:
- Join `sales` and `menu` on `product_id`.
- Group_by `(customer_id, product_name)` and use `count()` to count the total orders for each product for each customer.
- We use a dense rank on descending `total_purchased` partitioned by `customer_id` to obtain the `rank` column.
- Then we filter for where `rank == 1`, select the columns we want and sort by `customer_id`.

#### Answer:
| customer_id | product_name | total_purchased |
| ----------- | ---------- |------------  |
| A           | ramen        |  3   |
| B           | sushi        |  2   |
| B           | curry        |  2   |
| B           | ramen        |  2   |
| C           | ramen        |  3   |

- Customer A and C's favorite item is ramen.
- Customer B enjoys all items on the menu.

***

### 📌 6. Which item was purchased first by the customer after they became a member?

```python
first_purchase_after_member = (
    sales.join(members, on='customer_id')
    .filter(sf.col('order_date') >= sf.col('join_date'))
    .withColumn('rank', sf.rank().over(Window.partitionBy('customer_id').orderBy(sf.asc('order_date'))))
    .filter(sf.col('rank') == 1)
    .join(menu, on='product_id')
    .select('customer_id', 'product_name', 'order_date', 'join_date')
    .sort('customer_id'))

first_purchase_after_member.show()
```

#### Steps:
- Join `sales`, `menu`, and `members`.
- Filter for where `order_date` >= `join_date`.
- We then calculate the `rank` column by getting the dense rank on ascending `order_date`, partitioned by `customer_id`.
- Filter for where `rank == 1`.
- Select the desired columns.

#### Answer:
| customer_id | product_name | order_date | join_date |
| ----------- | ---------- | ----------- | ---------- |
| A           | curry        | 2021-01-07 | 2021-01-07 |
| B           | sushi        | 2021-01-11 | 2021-01-09 |

- Customer A's first order as a member was curry.
- Customer B's first order as a member was sushi.

***

### 📌 7. Which item was purchased just before the customer became a member?

````python

````
```python

```

#### Steps:
- Same as the previous question with two small changes.
- Filter for where `order_date` < `join_date` instead of  `order_date` >= `join_date`.
- We perform a descending dense rank on `order_date` instead of ascending.

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

```python

```
```python

```

#### Steps:
- Join `sales`, `menu`, and `members`.
- Filter for where `order_date` < `join_date`.
- Group_by `customer_id`.
- Calculate the count using `len()` and `total_spent` by getting the sum of `price`.

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

```python

```
```python

```

#### Steps:
Let's break down the question to understand the point calculation for each customer's purchases.
- Join `sales` and `menu` on `product_id`
- We utilize `pl.when().then().otherwise()` structure to create the points column based on the product and price.
  - when `product_id == 1` (sushi) then `price * 20` otherwise `price * 10`.
- Group_by `customer_id` and sum `points` to get the total points earned by each customer. 

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

```python

```
```python

```

#### Assumptions:
- each $1 spent equates to 10 points and sushi still has a 2x points multiplier
- the first week after a customer joins the program (including their join date) they earn 2x points on all items

#### Steps:
- Join `sales`, `menu`, and `members`.
- Filter for the month of January. `order_date` between 2021-1-1 and 2021-2-1.
- We utilize `pl.when().then().otherwise()` structure to create the points column based on the product, order_date and price.
  - when `product_id == 1` (sushi), or `order_date` is within 7 days of the `join_date` then `price * 20` otherwise `price * 10`.
-  Group_by `customer_id` and sum `points` to get the total points earned by each customer.


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

```python

```
````python

````

#### Steps:
- Join the tables `sales`, `menu`, `members`, ensuring we use `how='left'` on members as to not exclude non-members.
- Calculate the `member` column - if `order_date` >= `join_date` then 'Y' else 'N'
- Finally, select only the columns we want.

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
```python

```

#### Steps:
- Join the tables `sales`, `menu`, `members`, ensuring we use `how='left'` on members as to not exclude non-members.
- Calculate the `member` column - if `order_date` >= `join_date` then 'Y' else 'N'.
- Calculate the `ranking` column - if `member` == `'N'` then `None`, otherwise group_by `(customer_id, member)` and get the dense rank on ascending `order_date`.
- Finally, select only the columns we want.

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
