# CASE STUDY #1: Danny's Diner
#### 1. What is the total amount each customer spent at the restaurant?
```
SELECT customer_id, SUM(m.price)
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY customer_id;
```
#### Output:
|customer_id|sum|
|-----------|---|
|B          |74 |
|C          |36 |
|A          |76 |
