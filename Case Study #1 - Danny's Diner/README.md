# CASE STUDY #1: Danny's Diner  
<img src='https://8weeksqlchallenge.com/images/case-study-designs/1.png' width='500' height='500'/>
Note that all information on the case study can be found here: https://8weeksqlchallenge.com/case-study-1/

---
### Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

---
### Entity Relationship Diagram
<img src='https://miro.medium.com/v2/resize:fit:720/format:webp/1*PIhL73m7boVx6RJqd7TRjg.png' width='470' height='300'/>

---
# Solutions
Queries were executed using PostgreSQL on pgAdmin.
#### 1. What is the total amount each customer spent at the restaurant?
```
SELECT customer_id, SUM(m.price) AS total_sales
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY customer_id
ORDER BY customer_id;
```
#### Output:
|customer_id|total_sales|
|-----------|---|
|A          |76 |
|B          |74 |
|C          |36 |
- Customer A spent $76
- Customer B spent $74
- Customer C spent $36
---
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
- Customer A visited 4 days
- Customer B visited 6 days
- Customer C visited 2 days
---
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
- Customer A ordered both curry and sushi at the same time, making both the first order
- Customer B ordered curry first
- Customer C ordered ramen first
---
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
- Ramen is the most purchased item, being purchased a total of 8 times
---
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
- Customer A's most popular item was ramen, being purchased 3 times
- Customer B has a three way tie his most popular item: sushi, curry, and ramen, each purchased twice
- Customer C's most popular item was ramen, being purchased 3 times
---
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
- Customer A's first purchase after becoming a member was ramen
- Customer B's first purchase after becoming a member was sushi
---
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
- Customer A purchased sushi right before becoming a member
- Customer B purchased sushi right before becoming a member
---
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
- Customer A spent $25 on two items before becoming a member
- Customer B spent $40 on three items before becoming a member
---
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
- Customer A had 860 points
- Customer B had 940 points
- Customer C had 360 points
---
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
- Customer A would have 1370 points at the end of January
- Customer B would have 820 points at the end of January
---
# BONUS QUESTIONS
The following questions are related creating basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL.

Recreate the following table output using the available data:
|customer_id|order_date|product_name|price|member|
|-|-|-|-|-|
|A|2021-01-01|sushi|10|N|
|A|2021-01-01|curry|15|N|
|A|2021-01-07|curry|15|Y|
|A|2021-01-10|ramen|12|Y|
|A|2021-01-11|ramen|12|Y|
|A|2021-01-11|ramen|12|Y|
|B|2021-01-01|curry|15|N|
|B|2021-01-02|curry|15|N|
|B|2021-01-04|sushi|10|N|
|B|2021-01-11|sushi|10|Y|
|B|2021-01-16|ramen|12|Y|
|B|2021-02-01|ramen|12|Y|
|C|2021-01-01|ramen|12|N|
|C|2021-01-01|ramen|12|N|
|C|2021-01-07|ramen|12|N|
#### Solution
```
SELECT s.customer_id, order_date, product_name, price,
CASE
	WHEN order_date >= join_date THEN 'Y'
	ELSE 'N'
END AS member
FROM sales s
INNER JOIN menu mu
ON s.product_id = mu.product_id
LEFT JOIN members m
ON s.customer_id = m.customer_id
ORDER BY customer_id, order_date;
```
---
Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.  
|customer_id|order_date|product_name|price|member|ranking|
|-|-|-|-|-|-|
|A|2021-01-01|sushi|10|N|null|
|A|2021-01-01|curry|15|N|null|
|A|2021-01-07|curry|15|Y|1|
|A|2021-01-10|ramen|12|Y|2|
|A|2021-01-11|ramen|12|Y|3|
|A|2021-01-11|ramen|12|Y|3|
|B|2021-01-01|curry|15|N|null|
|B|2021-01-02|curry|15|N|null|
|B|2021-01-04|sushi|10|N|null|
|B|2021-01-11|sushi|10|Y|1|
|B|2021-01-16|ramen|12|Y|2|
|B|2021-02-01|ramen|12|Y|3|
|C|2021-01-01|ramen|12|N|null|
|C|2021-01-01|ramen|12|N|null|
|C|2021-01-07|ramen|12|N|null|
#### Solution
```
WITH customer_data AS (
	SELECT s.customer_id, order_date, product_name, price,
	CASE
		WHEN order_date >= join_date THEN 'Y'
		ELSE 'N'
	END AS member
	FROM sales s
	INNER JOIN menu mu
	ON s.product_id = mu.product_id
	LEFT JOIN members m
	ON s.customer_id = m.customer_id
	ORDER BY customer_id, order_date
)

SELECT *,
CASE
	WHEN member = 'N' THEN NULL
	ELSE RANK() OVER(
	PARTITION BY customer_id, member
	ORDER BY order_date
	)
END AS ranking
FROM customer_data
```
