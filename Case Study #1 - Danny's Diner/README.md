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
