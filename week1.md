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