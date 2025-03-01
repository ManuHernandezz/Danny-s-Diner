# Danny-s-Diner
  
  
 ## 1. What is the total amount each customer spent at the restaurant?

```sql

SELECT s.customer_id, SUM(m.price) AS total_spent
FROM sales s
INNER JOIN menu m
	ON s.product_id = m.product_id
GROUP BY s.customer_id
       
```

| customer_id | total_spent|
| :---        |    :----:   |
|A|	76 |
|B|	74|
|C|	36|


## 2. How many days has each customer visited the restaurant?


```sql

SELECT s.customer_id, COUNT(DISTINCT s.order_date) AS visit_days
FROM sales s
GROUP BY s.customer_id;
   
```

|customer_id	|visit_days|
| :---        |    :----:   |
|A	|4
|B	|6
|C	|2

## 3. What was the first item from the menu purchased by each customer?
```sql

WITH first_item AS (
  	SELECT s.customer_id,
  		   s.product_id,
  		   ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS row
  	FROM sales s
)
  
SELECT fi.customer_id, m.product_name
FROM first_item fi
INNER JOIN menu m
	ON fi.product_id = m.product_id
WHERE row = 1;
       

```
|customer_id	|product_name
| :---        |    :----:   |
|A	|sushi
|B	|curry
|C	|ramen


## 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql

SELECT TOP 1 m.product_name,
	COUNT(s.customer_id) AS Count_sales
FROM sales s
INNER JOIN menu m ON m.product_id = s.product_id
GROUP BY m.product_name
ORDER BY Count_sales DESC;

```
|product_name	|Count_sales
| :---        |    :----:   |
|ramen|	8




## 5. Which item was the most popular for each customer?

```sql

WITH counts AS (
  	SELECT 
        customer_id, 
        product_id, 
        COUNT(*) AS sales_count, 
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY COUNT(*) DESC) AS row_number
	FROM sales
    GROUP BY customer_id, product_id
)

SELECT customer_id, product_id, sales_count
FROM counts
WHERE row_number = 1;
       
```
|customer_id	|product_id	|sales_count|
| :---        |    :----:   |    :----:   |
|A	|3	|3
|B	|1	|2
|C	|3	|3


## 6. Which item was purchased first by the customer after they became a member?
 ```sql
 
 WITH first_purchase AS (
    SELECT m.join_date,
   			s.customer_id, 
  			s.order_date, 
  			me.product_name, 
  			ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date ASC) AS row_number
FROM members m
INNER JOIN sales s
	ON m.customer_id = s.customer_id
INNER JOIN menu me
	ON s.product_id = me.product_id
WHERE join_date <= order_date
   )
SELECT join_date, customer_id, order_date, product_name
FROM first_purchase
WHERE row_number = 1
     

```
| join_date	|customer_id	|order_date	|product_name
 | :---        |    :----:   |    :----:   |    :----:   |
|2021-01-07	|A	|2021-01-07	|curry
|2021-01-09	|B	|2021-01-11	|sushi


## 7. Which item was purchased just before the customer became a member?

```sql
 WITH first_purchase AS (
    SELECT m.join_date,
   			s.customer_id, 
  			s.order_date, 
  			me.product_name, 
  			ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date ASC) AS row_number
FROM members m
INNER JOIN sales s
	ON m.customer_id = s.customer_id
INNER JOIN menu me
	ON s.product_id = me.product_id
WHERE join_date > order_date
   )
SELECT join_date, customer_id, order_date, product_name
FROM first_purchase
ORDER BY customer_id     


```
|join_date	|customer_id	|order_date	|product_name
 | :---        |    :----:   |    :----:   |    :----:   |
|2021-01-07	|A	|2021-01-01	|sushi
|2021-01-07	|A	|2021-01-01	|curry
|2021-01-09	|B	|2021-01-01	|curry
|2021-01-09	|B	|2021-01-02	|curry
|2021-01-09	|B	|2021-01-04	|sushi


## 8. What is the total items and amount spent for each member before they became a member?

```sql

WITH first_purchase AS (
    SELECT m.join_date,
   			s.customer_id, 
  			s.order_date, 
  			me.product_name, 
   			me.price,
  			ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date ASC) AS row_number
FROM members m
INNER JOIN sales s
	ON m.customer_id = s.customer_id
INNER JOIN menu me
	ON s.product_id = me.product_id
WHERE join_date > order_date
   )
SELECT  customer_id, COUNT(*) AS total_items_before_join, SUM (price) AS total_spent
FROM first_purchase
GROUP BY customer_id
ORDER BY customer_id      


```

|customer_id	|total_items_before_join	|total_spent
 | :---        |    :----:   |    :----: |
|A	|2	|25
|B	|3	|40


## 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?


```sql

SELECT S.customer_id, 
		SUM (CASE 
        	WHEN m.product_name LIKE '%sushi%' THEN (m.price * 2*10 )
            ELSE (m.price * 10)
        END) AS total_points
FROM sales s
INNER JOIN menu m
	ON m.product_id = s.product_id
GROUP BY customer_id
ORDER BY total_points DESC      


```
|customer_id	|total_points
 | :---        |    :----:   | 
|B	|940
|A	|860
|C	|360



## 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

```sql
WITH first_week_sales AS (
    SELECT 
        m.customer_id,
        s.order_date,
        me.price
    FROM 
        members m
    INNER JOIN 
        sales s ON m.customer_id = s.customer_id
    INNER JOIN 
        menu me ON s.product_id = me.product_id
    WHERE 
        s.order_date >= m.join_date
        AND s.order_date < (m.join_date + INTERVAL '7 days') 
),
subsequent_sales AS (
    SELECT 
        m.customer_id,
        s.order_date,
        me.price
    FROM 
        members m
    INNER JOIN 
        sales s ON m.customer_id = s.customer_id
    INNER JOIN 
        menu me ON s.product_id = me.product_id
    WHERE 
        s.order_date >= (m.join_date + INTERVAL '7 days') 
        AND s.order_date <= '2021-01-31'
)

SELECT 
    customer_id,
    SUM(first_points) AS total_points
FROM (
    SELECT 
        customer_id,
        SUM(price * 2 * 10) AS first_points
    FROM 
        first_week_sales
    WHERE 
        customer_id IN ('A', 'B') 
    GROUP BY 
        customer_id

    UNION ALL

    SELECT 
        customer_id,
        SUM(price * 10) AS first_points
    FROM 
        subsequent_sales
    WHERE 
        customer_id IN ('A', 'B')
    GROUP BY 
        customer_id
) AS combined_sales
GROUP BY 
    customer_id
ORDER BY 
    total_points DESC;
```
|customer_id	|total_points
 | :---        |    :----:   | 
|A	|1020
|B	|320


       


```

