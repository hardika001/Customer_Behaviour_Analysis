# Customer_Behaviour_Analysis - MySQL WORLBENCH

This project is based on a case study that can be accessed by this [link](https://8weeksqlchallenge.com/case-study-1/). It focuses on examining patterns, trends, and factors influencing customer spending to gain insights into their preferences, purchasing habits, and potential areas for improvement in menu offerings or marketing strategies in a dining establishment.

## Introduction

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent, and which menu items are their favorite. This deeper connection with his customers will help him deliver a better and more personalized experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program -additionally, he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

## Entity Relationship Diagram

![Screenshot 2023-12-11 014238](https://github.com/hardika001/Customer_Behaviour_Analysis/assets/141905140/269088c9-bff2-4ea8-97db-2d86bd149153)

## Skills Applied

- Window Functions
- CTEs
- Aggregations
- JOINs
- Write scripts to generate basic reports that can be run every period

## Questions Explored

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

## SQL Queries of above questions

```sql
-- 1. What is the total amount each customer spent at the restaurant?
  select s.customer_id, sum(price) as amount_spent from sales s
  join menu m 
  on s.product_id = m.product_id
  group by customer_id;
```


```sql
-- 2. How many days has each customer visited the restaurant?
select CUSTOMER_ID, count(distinct order_date) as no_of_times_visited from sales
group by customer_id;
```


```sql
-- 3. What was the first item from the menu purchased by each customer?
with customer_first_purchase as 
 (select s.customer_id, min(s.order_date) as first_purchase_date
 from sales s
 group by s.customer_id)

select cfp.customer_id, cfp.first_purchase_date, m.product_name
from customer_first_purchase as cfp
join sales s on 
s.customer_id = cfp.customer_id
and cfp.first_purchase_date = s.order_date
join menu m on m.product_id = s.product_id;
```


```sql
-- 4. What is the most purchased item on the menu and how many times was it purchased by all
-- customers?
select m.product_name, count(*) as most_purchased
from sales s
join menu m 
on s.product_id = m.product_id
group by product_name
order by most_purchased desc
limit 1;
```


```sql
-- 5. What item was the most popular for each customer?
with customer_popularity as 
(select s.customer_id, m.product_name,count(*) as product_count,
row_number() over (partition by s.customer_id order by count(*) desc) as rnk
from sales s 
join menu m
on s.product_id = m.product_id
group by s.customer_id, m.product_name)

select cp.customer_id, cp.product_name, cp.product_count
from customer_popularity cp
where rnk = 1;
```


```sql
-- 6. Which item was purchased first by the customer after they became a member?
with first_purchase_after_membership as
(select s.customer_id, min(s.order_date) as first_purchase, m.join_date
from sales s 
join members m
on s.customer_id = m.customer_id
and s.order_date >= m.join_date
group by s.customer_id, m.join_date)

select fpm.customer_id, mu.product_name
from first_purchase_after_membership fpm
join sales s 
on s.customer_id = fpm.customer_id
and s.order_date = fpm.first_purchase
join menu mu
on mu.product_id = s.product_id
order by fpm.customer_id;
```


```sql
-- 7. Which item was purchased just before the customer became a member?
with item_purchased_before_joindate as
(select mu.product_name, s.order_date,s.customer_id, m.join_date,
rank() over (partition by customer_id order by order_date desc) as rnk
from sales s
join members m 
on s.customer_id = m.customer_id
join menu mu
on s.product_id = mu.product_id
where s.order_date < m.join_date)

select ip.product_name, ip.customer_id
from item_purchased_before_joindate as ip
where rnk = 1;
```


```sql
-- 8. What is the total items and amount spent for each member before they became a 
-- member?

select s.customer_id, sum(mu.price) as total_amount, count(s.product_id) as total_items
from sales s
join members m 
on s.customer_id = m.customer_id
join menu mu
on s.product_id = mu.product_id
where s.order_date < m.join_date
group by s.customer_id
order by customer_id;
```



```sql
-- 9. if each $1 spent equates to 10 points and sushi has a 2x points multiplier-
-- how many points would each customer have?

 select s.customer_id, sum(
      case
         when m.product_name = 'sushi' then m.price*20
         else m.price*10 end) as total_points
from sales s 
join menu m ON 
s.product_id = m.product_id
group by s.customer_id;
```

```sql
-- 10. In the first week after a customer joins the program (including their 
-- join date)they earn 2x points on all items, not just sushi - 
-- how many points do customer A and B have at the end of January?
  
select s.customer_id,
sum(
    case when s.order_date between m.join_date and date_add(join_date, interval 7 day)
          then mu.price*20
          when mu.product_name = "sushi" then mu.price*20
          else mu.price*10 end) as total_points
from sales s 
join menu mu 
on s.product_id = mu.product_id
join  members m on 
s.customer_id = m.customer_id
where s.customer_id in ('A', 'B') and s.order_date <= '2021-01-31'
group by s.customer_id
order by customer_id;
```


```sql
-- BONUS QUESTIONS
-- 1.
select s.customer_id, s.order_date, m.product_name, m.price,
case
when s.customer_id = mb.customer_id and s.order_date >= mb.join_date then "Y"
else "N" end as member
from sales s 
join menu m on
s.product_id = m.product_id
left join members mb on 
s.customer_id = mb.customer_id;
```


```sql
-- 2. 
with ranking_query as 
(select s.customer_id, s.order_date, m.product_name, m.price,
case
when s.customer_id = mb.customer_id and s.order_date >= mb.join_date then "Y"
else "N" end as member
from sales s 
join menu m on
s.product_id = m.product_id
left join members mb on 
s.customer_id = mb.customer_id)

select *,
case
when member = 'N' then null
else rank() over(partition by customer_id, member order by order_date) end
as ranking
from ranking_query;
```


## Insights

- Customer B is the most frequent visitor with 6 visits in Jan 2021.
- Customer A has spent most followed by Customer B
- Danny’s Diner’s most popular item is ramen, followed by curry and sushi.
- Customer A loves ramen, Customer C loves only ramen whereas Customer B seems to enjoy sushi, curry and ramen equally.
- The last items ordered by Customers A and B before they became members are sushi and curry.
  
