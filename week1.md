# Week 1: Danny’s Diner case study

## Context
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

## Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

## Example Datasets
Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

Danny has shared with you 3 key datasets for this case study:

sales
menu
members
You can inspect the entity relationship diagram and example data below.

## Solutions by Thai Tran
### 1. What is the total amount each customer spent at the restaurant?

```sql
select customer_id, sum(price)
from dannys_diner.menu as a, dannys_diner.sales as b
where a.product_id= b.product_id
group by customer_id
```

### 2. How many days has each customer visited the restaurant?

```sql
select customer_id, count( DISTINCT order_date)
from dannys_diner.sales
group by customer_id
```

### 3. What was the first item from the menu purchased by each customer?

```sql
select DISTINCT a.customer_id, product_name
from dannys_diner.sales as a, (select customer_id, min(order_date) as mindate
from dannys_diner.sales
group by customer_id) as b, dannys_diner.menu as c
where a.customer_id=b.customer_id
and order_date = mindate
and c.product_id=a.product_id
```

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
select product_id, count(*) as numb
from dannys_diner.sales
group by product_id
ORDER BY 2 DESC
limit 1
```
### 5. Which item was the most popular for each customer?
```sql
select d.customer_id, d.product_id,maxnum
from (select customer_id, max(numb) as maxnum
      from (select customer_id, product_id, count(*) as numb
            from dannys_diner.sales
            group by customer_id, product_id) as a
            group by customer_id) as c, 
            (select customer_id, product_id, count(*) as numb
            from dannys_diner.sales
            group by customer_id, product_id) as d
      where c.customer_id=d.customer_id
      and maxnum = numb
```

Which item was purchased first by the customer after they became a member?


Which item was purchased just before the customer became a member?
What is the total items and amount spent for each member before they became a member?
If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
