# Danny-s-Diner
Case Study #1 - Danny's Diner

SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
  
  
 -- 1. What is the total amount each customer spent at the restaurant?
SELECT s.customer_id, SUM(m.price) AS total_spent
FROM sales s
INNER JOIN menu m
	ON s.product_id = m.product_id
GROUP BY s.customer_id


-- 2. How many days has each customer visited the restaurant?

SELECT s.customer_id, COUNT(DISTINCT s.order_date) AS visit_days
FROM sales s
GROUP BY s.customer_id;


-- 3. What was the first item from the menu purchased by each customer?

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

-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT m.product_name, COUNT(s.customer_id) AS Count_sales
FROM sales s
INNER JOIN menu m
	ON m.product_id = s.product_id
GROUP BY product_name
ORDER BY Count_sales DESC
LIMIT 1


-- 5. Which item was the most popular for each customer?

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

-- 6. Which item was purchased first by the customer after they became a member?
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

-- 7. Which item was purchased just before the customer became a member?

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

-- 8. What is the total items and amount spent for each member before they became a member?

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


-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

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


-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

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
