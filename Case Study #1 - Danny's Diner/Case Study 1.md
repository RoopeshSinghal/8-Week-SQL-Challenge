
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

## 2. Number of Days Each Customer Visited the Restaurant

```sql
SELECT
 customer_id,
 COUNT(DISTINCT order_date)
FROM dannys_diner.sales AS visits_cnt
GROUP BY 1
ORDER BY 2 DESC;
```

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
