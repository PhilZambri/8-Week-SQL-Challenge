# 🍜 Case Study #1: Danny's Diner 
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Setup for Polars using pl.read_database_uri()](#setup-for-polars-using-plread_database_uri)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***
## Setup for Polars using pl.read_database_uri()
Using pl.read_database_uri(), we can query the local postgresql database named '8 WeekSQLChallenge', to extract all three tables - sales, menu, and members - 
from dannys_diner into three Polars Dataframes. We will use these dataframes to answer all of the following case study questions.

```python
import polars as pl

uri="postgresql://user:pass@localhost:5432/8%20WeekSQLChallenge"

sales = pl.read_database_uri(
    query="SELECT * FROM dannys_diner.sales", 
    uri=uri)

menu = pl.read_database_uri(
    query="SELECT * FROM dannys_diner.menu", 
    uri=uri)

members = pl.read_database_uri(
    query="SELECT * FROM dannys_diner.members", 
    uri=uri)

sales_lazy = sales.lazy()
menu_lazy = menu.lazy()
members_lazy = members.lazy()
```

***

## Question and Solution

### 📌 1. What is the total amount each customer spent at the restaurant?

````python
# Eager

sales_menu = sales.join(menu, on='product_id')
sales_menu = (sales_menu
             .group_by('customer_id')
             .agg(total_spent=pl.col('price').sum())
             .sort('customer_id'))

print(sales_menu)
````
````python
# Lazy

q = (sales_lazy.join(menu_lazy, on='product_id')
    .group_by('customer_id')
    .agg(pl.col('price').sum().alias('total_spent'))
    .sort('customer_id'))

df = q.collect()
print(df)
````

#### Steps:
- Use **JOIN** to merge `sales` and `menu` on `product_id`.
- Groupby `customer_id` and calculate the `sum` of `price` into a column named `total_spent`.
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
# Eager

unique_days_visited = (sales
                       .group_by('customer_id')
                       .agg(days_visited = pl.col('order_date').n_unique())
                       .sort('customer_id'))

print(unique_days_visited)
````
````python
# Lazy

q = (sales_lazy
     .group_by('customer_id')
     .agg(days_visited = pl.col('order_date').n_unique())
     .sort('customer_id'))

df = q.collect()
print(df)
````

#### Steps:
- Group_by `customer_id`.
- Create the `days_visited` column by using the `n_unique` function on the `order_date` column.
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
# Eager

first_order = (sales.join(menu, on='product_id')
               .with_columns(rank=pl.col('order_date').rank(method='dense').over(partition_by='customer_id'))
               .filter(pl.col('rank') == 1)
               .select('customer_id', pl.col('product_name').alias('first_order'))
               .unique()
               .sort('customer_id'))

print(first_order)
````
````python
# Lazy

first_order = (sales_lazy.join(menu_lazy, on='product_id')
               .with_columns(rank=pl.col('order_date').rank(method='dense').over(partition_by='customer_id'))
               .filter(pl.col('rank') == 1)
               .select('customer_id', pl.col('product_name').alias('first_order'))
               .unique()
               .sort('customer_id'))

df = first_order.collect()
print(df)  
````
#### Steps:
- Join `sales` and `menu` on `product_id`.
- In order to respect duplicates(where more than one order took place on the same day) we will utilize the `rank` function.
- We calculate the `dense rank` on ascending `order_date`, grouped by `customer_id` and save into a new column named `rank`.
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
# Eager

most_purchased_item = (sales.join(menu, on='product_id')
                       .group_by('product_name')
                       .len(name='total_purchased')
                       .top_k(1, by='total_purchased'))

print(most_purchased_item)
````
````python
# Lazy

most_purchased_item = (sales_lazy.join(menu_lazy, on='product_id')
                       .group_by('product_name')
                       .len(name='total_purchased')
                       .top_k(1, by='total_purchased'))

df = most_purchased_item.collect()
print(df)
````

#### Steps:
- Join `sales` and `menu` on `product_id`.
- Group_by `product_name` and use `len()` to count the number of each product purchased as a column named `total_purchased`.
- We utilize the `top_k()` function on the `total_purchased` column to get the most purchased item.

#### Answer:
| product_name | total_purchased | 
| ----------- | ----------- |
| ramen       | 8 |


- Most purchased item on the menu is ramen which is 8 times. Yum!

### 📍 What is the most purchased item on the menu and how many times was it purchased total by all customers and by each customer?
#### Alternatively, I provide an answer for the same question but a little more involved.

````python
most_purchased_item = (sales
                       .group_by('product_id')
                       .len(name='total_purchased')
                       .top_k(1, by='total_purchased')[0, 'product_id'])

x = (sales.join(menu, on='product_id')
     .filter(pl.col('product_id') == most_purchased_item)
     .group_by('customer_id', 'product_name')
     .len(name='total_purchased')
     .sort('customer_id'))

y = pl.DataFrame({
    "customer_id": "All Customers",
    "product_name": x[0,'product_name'],
    "total_purchased": x.select(pl.col('total_purchased').sum())
})

result = pl.concat([x, y])
print(result )
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
# Eager

most_purchased_items = (sales.join(menu, on='product_id')
                        .group_by('customer_id', 'product_name').len(name='order_count')
                        .with_columns(rank=pl.col('order_count').rank(method='dense', descending=True).over(partition_by='customer_id'))
                        .filter(pl.col('rank') == 1)
                        .select('customer_id', 'product_name', 'order_count')
                        .sort('customer_id'))

print(most_purchased_items)
````
````python
# Lazy 

most_purchased_items = (sales_lazy.join(menu_lazy, on='product_id')
                        .group_by('customer_id', 'product_name').len(name='order_count')
                        .with_columns(rank=pl.col('order_count').rank(method='dense', descending=True).over(partition_by='customer_id'))
                        .filter(pl.col('rank') == 1)
                        .select('customer_id', 'product_name', 'order_count')
                        .sort('customer_id'))

df = most_purchased_items.collect()
print(df)
````

*Each user may have more than 1 most ordered item.*

#### Steps:
- Join `sales` and `menu` on `product_id`.
- Group_by `(customer_id, product_name)` and use `len()` to count the total orders for each product for each customer.
- We execute a dense rank on descending `order_count` partitioned by `customer_id` to obtain the `rank` column.
- Then we filter for where `rank == 1`, select the columns we want and sort by `customer_id`.

#### Answer:
| customer_id | product_name | order_count |
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
# Eager

first_purchase = (sales.join(menu, on='product_id').join(members, on='customer_id')
                  .filter(pl.col('order_date') >= pl.col('join_date'))
                  .with_columns(rank=pl.col('order_date').rank(method='dense').over(partition_by='customer_id'))
                  .filter(pl.col('rank') == 1)
                  .select('customer_id', 'product_name', 'order_date', 'join_date'))

print(first_purchase)
```
```python
# Lazy

first_purchase = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id')
                  .filter(pl.col('order_date') >= pl.col('join_date'))
                  .with_columns(rank=pl.col('order_date').rank(method='dense').over(partition_by='customer_id'))
                  .filter(pl.col('rank') == 1)
                  .select('customer_id', 'product_name', 'order_date', 'join_date'))

df = first_purchase.collect()
print(df)
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

- Customer A's first order as a member was ramen.
- Customer B's first order as a member was sushi.

***

### 📌 7. Which item was purchased just before the customer became a member?

````python
# Eager

first_purchase = (sales.join(menu, on='product_id').join(members, on='customer_id')
                  .filter(pl.col('order_date') < pl.col('join_date'))
                  .with_columns(rank=pl.col('order_date').rank(method='dense', descending=True).over(partition_by='customer_id'))
                  .filter(pl.col('rank') == 1)
                  .select('customer_id', 'product_name', 'order_date', 'join_date'))

print(first_purchase)
````
```python
# Lazy

first_purchase = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id')
                  .filter(pl.col('order_date') < pl.col('join_date'))
                  .with_columns(rank=pl.col('order_date').rank(method='dense', descending=True).over(partition_by='customer_id'))
                  .filter(pl.col('rank') == 1)
                  .select('customer_id', 'product_name', 'order_date', 'join_date'))

df = first_purchase.collect()
print(df)
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
# Eager

totals = (sales.join(menu, on='product_id').join(members, on='customer_id')
          .filter(pl.col('order_date') < pl.col('join_date'))
          .group_by('customer_id')
          .agg(total_items=pl.col('customer_id').len(), total_spent=pl.col('price').sum()))

print(totals)
```
```python
# Lazy

totals = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id')
          .filter(pl.col('order_date') < pl.col('join_date'))
          .group_by('customer_id')
          .agg(total_items=pl.col('customer_id').len(), total_spent=pl.col('price').sum()))

df = totals.collect()
print(df)
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
# Eager

total_points = (sales.join(menu, on='product_id')
                .select('customer_id', points=pl.when(pl.col('product_id') == 1)
                                                .then(pl.col('price') * 20)
                                                .otherwise(pl.col('price') * 10))
                .group_by('customer_id')
                .agg(total_points=pl.col('points').sum())
                .sort('customer_id'))

print(total_points)
```
```python
# Lazy

total_points = (sales_lazy.join(menu_lazy, on='product_id')
                .select('customer_id', points=pl.when(pl.col('product_id') == 1)
                                                .then(pl.col('price') * 20)
                                                .otherwise(pl.col('price') * 10))
                .group_by('customer_id')
                .agg(total_points=pl.col('points').sum())
                .sort('customer_id'))

df = total_points.collect()
print(df)
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
# Eager

condition = (pl.col('product_id') == 1) | ((pl.col('order_date') >= pl.col('join_date')) & (pl.col('order_date') < pl.col('join_date') + pl.duration(days=7)))
total_points = (sales.join(menu, on='product_id').join(members, on='customer_id')
                .filter((pl.col('order_date') >= pl.date(2021, 1, 1)) & (pl.col('order_date') < pl.date(2021, 2, 1)))
                .select('customer_id', points=pl.when(condition )
                                                .then(pl.col('price') * 20)
                                                .otherwise(pl.col('price') * 10))
                .group_by('customer_id')
                .agg(total_points=pl.col('points').sum())
                .sort('customer_id'))

print(total_points)
```
```python
# Lazy

condition = (pl.col('product_id') == 1) | ((pl.col('order_date') >= pl.col('join_date')) & (pl.col('order_date') < pl.col('join_date') + pl.duration(days=7)))
total_points = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id')
                .filter((pl.col('order_date') >= pl.date(2021, 1, 1)) & (pl.col('order_date') < pl.date(2021, 2, 1)))
                .select('customer_id', points=pl.when(condition )
                                                .then(pl.col('price') * 20)
                                                .otherwise(pl.col('price') * 10))
                .group_by('customer_id')
                .agg(total_points=pl.col('points').sum())
                .sort('customer_id'))

df = total_points.collect()
print(df)
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
# Eager

member = (sales.join(menu, on='product_id').join(members, on='customer_id', how='left')
          .select('customer_id', 
                  'order_date', 
                  'product_name', 
                  'price', 
                  member=pl.when(pl.col('order_date') >= pl.col('join_date')).then(pl.lit('Y')).otherwise(pl.lit('N'))))

print(member)
```
````python
# Lazy

member = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id', how='left')
          .select('customer_id', 
                  'order_date', 
                  'product_name', 
                  'price', 
                  member=pl.when(pl.col('order_date') >= pl.col('join_date')).then(pl.lit('Y')).otherwise(pl.lit('N'))))

df = member.collect()
print(df)
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
# Eager

condition = pl.col('order_date') >= pl.col('join_date')

member_ranking = (sales.join(menu, on='product_id').join(members, on='customer_id', how='left')
                  .with_columns(member=pl.when(condition).then(pl.lit('Y')).otherwise(pl.lit('N')))
                  .select('customer_id', 
                          'order_date', 
                          'product_name', 
                          'price', 
                          'member',
                          ranking=pl.when(pl.col('member') == 'N').then(None).otherwise(pl.col('order_date').rank(method='dense').over(partition_by=['customer_id', 'member']))))

print(member_ranking)
```
```python
# Lazy

condition = pl.col('order_date') >= pl.col('join_date')

member_ranking = (sales_lazy.join(menu_lazy, on='product_id').join(members_lazy, on='customer_id', how='left')
                  .with_columns(member=pl.when(condition).then(pl.lit('Y')).otherwise(pl.lit('N')))
                  .select('customer_id', 
                          'order_date', 
                          'product_name', 
                          'price', 
                          'member',
                          ranking=pl.when(pl.col('member') == 'N').then(None).otherwise(pl.col('order_date').rank(method='dense').over(partition_by=['customer_id', 'member']))))

df = member_ranking.collect()
print(df)
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
