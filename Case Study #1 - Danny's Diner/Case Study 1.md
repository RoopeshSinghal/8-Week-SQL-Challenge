
# Danny's Diner SQL Analysis

## 1. Total Amount Spent by Each Customer

```sql
SELECT 
 customer_id,
 SUM(price) AS ttl_amt
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
USING(product_id)
GROUP BY 1
ORDER BY 2 DESC;
```
#### Result set:

| customer_id | ttl_amt |
| ----------- | ------- |
| A           | 76      |
| B           | 74      |
| C           | 36      |

## 2. Number of Days Each Customer Visited the Restaurant

```sql
SELECT
 customer_id,
 COUNT(DISTINCT order_date)
FROM dannys_diner.sales AS visits_cnt
GROUP BY 1
ORDER BY 2 DESC;
```
#### Result set:

| customer_id | count |
| ----------- | ----- |
| B           | 6     |
| A           | 4     |
| C           | 2     |

## 3. First Item Purchased by Each Customer

```sql
SELECT DISTINCT customer_id, product_name
FROM (
  SELECT *,
  RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS r
  FROM dannys_diner.sales s
  INNER JOIN dannys_diner.menu m
  USING(product_id)
) a
WHERE r = 1
GROUP BY 1, 2;
```
#### Result set:

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

## 4. Most Purchased Item on the Menu

```sql
SELECT
 m.product_name,
 COUNT(s.product_id) AS purchase_count
FROM dannys_diner.menu m
INNER JOIN dannys_diner.sales s
USING(product_id)
GROUP BY 1
ORDER BY 2 DESC
LIMIT 1;
```
#### Result set:

| product_name | purchase_count |
| ------------ | --------------- |
| ramen        | 8               |

## 5. Most Popular Item for Each Customer

```sql
WITH cte AS (
  SELECT
    m.product_name,
    s.customer_id,
    COUNT(s.product_id) AS purchase_count,
    RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) AS rnk
  FROM dannys_diner.menu m
  INNER JOIN dannys_diner.sales s
  USING(product_id)
  GROUP BY 1, 2
)
SELECT customer_id, product_name
FROM cte
WHERE rnk = 1;
```
#### Result set:

| customer_id | product_name |
| ----------- | ------------- |
| A           | ramen         |
| B           | ramen         |
| B           | sushi         |
| B           | curry         |
| C           | ramen         |

## 6. First Item Purchased After Customer Became a Member

```sql
SELECT
  customer_id,
  product_name
FROM (
  SELECT
    me.customer_id,
    m.product_name,
    s.order_date,
    me.join_date,
    DENSE_RANK() OVER(PARTITION BY me.customer_id ORDER BY s.order_date) rnk
  FROM dannys_diner.members me
  INNER JOIN dannys_diner.sales s
  USING(customer_id)
  INNER JOIN dannys_diner.menu m
  USING(product_id)
  WHERE s.order_date >= me.join_date
) a
WHERE rnk = 1;
```
#### Result set:

| customer_id | product_name |
| ----------- | ------------- |
| A           | curry         |
| B           | sushi         |

## 7. Item Purchased Just Before Customer Became a Member

```sql
SELECT
  customer_id,
  product_name
FROM (
  SELECT
    me.customer_id,
    m.product_name,
    s.order_date,
    me.join_date,
    DENSE_RANK() OVER(PARTITION BY me.customer_id ORDER BY s.order_date) rnk
  FROM dannys_diner.members me
  INNER JOIN dannys_diner.sales s
  USING(customer_id)
  INNER JOIN dannys_diner.menu m
  USING(product_id)
  WHERE s.order_date < me.join_date
) a
WHERE rnk = 1;
```
#### Result set:

| customer_id | product_name |
|-------------|--------------|
| A           | sushi        |
| A           | curry        |
| B           | curry        |

## 8. Total Items and Amount Spent Before Membership

```sql
SELECT
  me.customer_id,
  COUNT(m.product_id) AS total_items,
  SUM(m.price) AS amount_spent
FROM dannys_diner.members me
INNER JOIN dannys_diner.sales s
USING(customer_id)
INNER JOIN dannys_diner.menu m
USING(product_id)
WHERE s.order_date < me.join_date
GROUP BY 1
ORDER BY 1;
```
#### Result set:

| customer_id | total_items | amount_spent |
|-------------|-------------|--------------|
| A           | 2           | 25           |
| B           | 3           | 40           |

## 9. Points Calculation with Sushi Multiplier

```sql
SELECT
 s.customer_id,
 SUM(CASE
      WHEN m.product_name = 'sushi' THEN m.price * 2 * 10
      ELSE m.price * 10
    END) AS points
FROM dannys_diner.sales s
INNER JOIN dannys_diner.menu m
USING(product_id)
GROUP BY 1
ORDER BY 1;
```
#### Result set:

| customer_id | points |
| ----------- | ------ |
| A           | 860    |
| B           | 940    |
| C           | 360    |

## 10. Points Earned by Customers A and B at the End of January

```sql
WITH cte AS (
  SELECT join_date,
         DATE_ADD(join_date, INTERVAL '6 DAY') AS program_last_date,
         customer_id
  FROM   dannys_diner.members
)
SELECT s.customer_id,
       SUM(CASE
             WHEN order_date BETWEEN join_date AND program_last_date THEN
               price * 10 * 2
             WHEN order_date NOT BETWEEN join_date AND program_last_date
                  AND product_name = 'sushi' THEN price * 10 * 2
             WHEN order_date NOT BETWEEN join_date AND program_last_date
                  AND product_name != 'sushi' THEN price * 10
           END) AS customer_points
FROM   dannys_diner.menu AS m
INNER JOIN dannys_diner.sales s USING(product_id)
INNER JOIN cte ON s.customer_id = cte.customer_id
AND order_date <= '2021-01-31'  AND order_date >= join_date
GROUP BY 1
ORDER BY 1;
```
#### Result set:


| customer_id | customer_points |
| ----------- | --------------- |
| A           | 1020            |
| B           | 320             |

##BONUS QUES

## i. Join All The Things
### Create basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL. Fill Member column as 'N' if the purchase was made before becoming a member and 'Y' if the after is amde after joining the membership.

```sql
SELECT   
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE WHEN s.order_date >= mem.join_date THEN 'Y' ELSE 'N' END AS member
FROM dannys_diner.members mem
RIGHT JOIN dannys_diner.sales s USING (customer_id)
INNER JOIN dannys_diner.menu m USING (product_id)
ORDER BY 1, 2;
```
#### Result set:

| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | ------------ | ----- | ------ |
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

## Rank All The Things

### Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

```sql
WITH cte as
(
SELECT   
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE WHEN s.order_date >= mem.join_date THEN 'Y' ELSE 'N' END AS member
FROM dannys_diner.members mem
RIGHT JOIN dannys_diner.sales s USING (customer_id)
INNER JOIN dannys_diner.menu m USING (product_id)
ORDER BY 1, 2
)
SELECT
  *,
  CASE WHEN member = 'N' THEN NULL ELSE DENSE_RANK() OVER(PARTITION BY customer_id,member ORDER BY order_date) END AS ranking
FROM cte;
```

#### Result set:

| customer_id | order_date | product_name | price | member | ranking |
| ----------- | ---------- | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01 | sushi        | 10    | N      | NULL    |
| A           | 2021-01-01 | curry        | 15    | N      | NULL    |
| A           | 2021-01-07 | curry        | 15    | Y      | 1       |
| A           | 2021-01-10 | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01 | curry        | 15    | N      | NULL    |
| B           | 2021-01-02 | curry        | 15    | N      | NULL    |
| B           | 2021-01-04 | sushi        | 10    | N      | NULL    |
| B           | 2021-01-11 | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16 | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01 | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
| C           | 2021-01-07 | ramen        | 12    | N      | NULL    |
