# CASE STUDY #1: Danny's Diner
#### 1. What is the total amount each customer spent at the restaurant?
```
SELECT customer_id, SUM(m.price) AS total_sales
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY customer_id;
```
#### Output:
|customer_id|total_sales|
|-----------|---|
|B          |74 |
|C          |36 |
|A          |76 |

#### 2. How many days has each customer visited the restaurant?
```
SELECT customer_id, COUNT(DISTINCT(order_date)) AS days_visited
FROM sales
GROUP BY customer_id;
```
#### Output:
|customer_id|days_visited|
|-|-|
|A|4|
|B|6|
|C|2|

#### 3. What was the first item from the menu purchased by each customer?
```
WITH sales_ordered AS (
	SELECT customer_id, order_date, product_name, RANK() OVER(
		PARTITION BY customer_id
		ORDER BY order_date
	) AS index
	FROM sales s
	INNER JOIN menu m
	ON s.product_id = m.product_id
)

SELECT customer_id, product_name AS first_purchase
FROM sales_ordered
WHERE index = 1
GROUP BY customer_id, product_name;
```
#### Output:
|customer_id|first_purchase|
|-|-|
|A|curry|
|A|sushi|
|B|curry|
|C|ramen|

#### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```
SELECT COUNT(s.product_id) AS purchases, product_name
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY product_name
LIMIT 1;
```
#### Output:
|purchases|product_name|
|-|-|
|8|ramen|

#### 5. Which item was the most popular for each customer?
```
WITH top_item AS(
	SELECT customer_id, product_name, COUNT(s.product_id) AS purchase_count, RANK() OVER(
		PARTITION BY customer_id
		ORDER BY COUNT(s.product_id) DESC
	)
	FROM sales s
	INNER JOIN menu m
	ON s.product_id = m.product_id
	GROUP BY customer_id, product_name
)

SELECT customer_id, product_name, purchase_count
FROM top_item
WHERE rank = 1;
```
#### Output:
|customer_id|product_name|purchase_count|
|-|-|-|
|A|ramen|3|
|B|sushi|2|
|B|curry|2|
|B|ramen|2|
|C|ramen|3|

#### 6. Which item was purchased first by the customer after they became a member?
```
WITH first_purchase AS (
	SELECT s.customer_id, product_name, RANK() OVER(
		PARTITION BY s.customer_id
		ORDER BY order_date
	) AS ranking
	FROM sales s
	INNER JOIN members m
	ON s.customer_id = m.customer_id
	INNER JOIN menu mu
	ON s.product_id = mu.product_id
	WHERE order_date > join_date
)

SELECT customer_id, product_name
FROM first_purchase
WHERE ranking = 1;
```
#### Output:
|customer_id|product_name|
|-|-|
|A|ramen|
|B|sushi|

#### 7. Which item was purchased just before the customer became a member?
```
WITH items_purchased_before AS (
	SELECT s.customer_id, order_date, product_name, ROW_NUMBER() OVER(
		PARTITION BY s.customer_id
		ORDER BY order_date DESC
	) AS ranking
	FROM sales s
	JOIN members m
	ON s.customer_id = m.customer_id
	JOIN menu mu
	ON s.product_id = mu.product_id
	WHERE order_date < join_date
	ORDER BY customer_id, order_date
)

SELECT customer_id, product_name
FROM items_purchased_before
WHERE ranking = 1;
```
#### Output:
|customer_id|product_name|
|-|-|
|A|sushi|
|B|sushi|

#### 8. What is the total items and amount spent for each member before they became a member?
```
SELECT s.customer_id, COUNT(product_name) AS items, SUM(price) AS amount_spent
FROM sales s
INNER JOIN members m
ON s.customer_id = m.customer_id
INNER JOIN menu mu
ON s.product_id = mu.product_id
WHERE order_date < join_date
GROUP BY s.customer_id
ORDER BY s.customer_id;
```
#### Output:
|customer_id|items|amount_spent|
|-|-|-|
|A|2|25|
|B|3|40|

#### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```
WITH points AS (
	SELECT customer_id, product_name,
	CASE
		WHEN product_name <> 'sushi' THEN price*10
		ELSE price*20
	END AS points
	FROM sales s
	INNER JOIN menu m
	ON s.product_id = m.product_id
)

SELECT customer_id, SUM(points) AS total_points
FROM points
GROUP BY customer_id
ORDER BY customer_id;
```
#### Output:
|customer_id|total_points|
|-|-|
|A|860|
|B|940|
|C|360|

#### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```
WITH points_january AS (
	SELECT s.customer_id,
	CASE
		WHEN order_date BETWEEN join_date AND join_date+6 THEN price*20
		WHEN order_date NOT BETWEEN join_date AND join_date+6 AND product_name = 'sushi' THEN price*20
		ELSE price*10
	END AS points
	FROM sales s
	INNER JOIN members m
	ON s.customer_id = m.customer_id
	INNER JOIN menu mu
	ON s.product_id = mu.product_id
	WHERE EXTRACT(MONTH FROM order_date) = 01
)

SELECT customer_id, SUM(points) AS total_points
FROM points_january
GROUP BY customer_id
ORDER BY customer_id;
```
#### Output:
|customer_id|total_points|
|-|-|
|A|1370|
|B|820|
