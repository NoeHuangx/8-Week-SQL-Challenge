# 🍜Case Study #1: Danny's Diner

<p align="center">
<img width="400px"  src="https://8weeksqlchallenge.com/images/case-study-designs/1.png" />
</p>

Please view the detailed case study info <a href="https://8weeksqlchallenge.com/case-study-1/">here</a>. 


## 📚Table of Contents
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially their visiting patterns, how much money they’ve spent, and which menu items are their favorites.

Danny has shared 3 key datasets:
- sales
- menu
- members

The entity relationship diagram is as below: 
<p align="center">
<img width="548" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/d3e68cca-c81c-4dcb-8e54-af5e99371c95">
</p>

***

## Questions and Solutions 
### 1. What is the total amount each customer spent at the restaurant? 

```mysql
SELECT 
    s.customer_id,
    SUM(m.price) AS total_spent
FROM
    sales s
        JOIN
    menu m ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```  

**Answer:**
|customer_id | total_spent |
|------------|-------------|
|A           | 76          |
|B           | 74          |
|C           | 36          |


### 2. How many days has each customer visited the restaurant?

```mysql
SELECT 
    customer_id,
    COUNT(DISTINCT order_date) AS visit_count
FROM
    sales
GROUP BY customer_id;
```

**Answer:**
| customer_id | visit_count|
|-------------|------------|
|A            | 4          |
|B            | 6          |
|C            | 2          |


### 3. What was the first item from the menu purchased by each customer?

```mysql
WITH order_rank AS (
	SELECT 
		s.customer_id, s.order_date, m.product_name,
		DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS rnk
	FROM
		sales s
			JOIN
		menu m ON s.product_id = m.product_id)
    
SELECT customer_id, product_name, order_date FROM order_rank WHERE rnk = 1
GROUP BY customer_id, product_name; 
```

**Answer:** 
|customer_id |product_name |order_date|
|------------|-------------|----------|
|A           |sushi        |2021-01-01|
|A           |curry        |2021-01-01|
|B           |curry        |2021-01-01|
|C           |ramen        |2021-01-01|


### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```mysql
SELECT 
    m.product_name,
    COUNT(*) AS order_count
FROM
    sales s
        JOIN
    menu m ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY order_count DESC
LIMIT 1; 
```

**Answer:** 
|product_name|order_count|
|------------|-----------|
|ramen       |8          |


### 5. Which item was the most popular for each customer?

```mysql
WITH units_sold AS (
SELECT 
    s.customer_id,
    m.product_name,
    COUNT(m.product_name) AS order_count,
    DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.customer_id) DESC) AS rnk
FROM
    sales s
        JOIN
    menu m ON s.product_id = m.product_id
GROUP BY customer_id, product_name) 

SELECT customer_id, product_name, order_count
FROM units_sold WHERE rnk = 1; 
```

**Answer:**
|customer_id|product_name|order_count|
|-----------|------------|-----------|
|A          |ramen       |3          |
|B          |curry       |2          |
|B          |sushi       |2          |
|B          |ramen       |2          |
|C          |ramen       |3          |


### 6. Which item was purchased first by the customer after they became a member?

```mysql
WITH order_after_member AS (
SELECT 
	s.customer_id, s.order_date, mm.join_date, m.product_name,
	DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS rnk 
FROM sales s JOIN members mm ON s.customer_id = mm.customer_id
JOIN menu m ON s.product_id = m.product_id
WHERE s.order_date >= DATE(mm.join_date)
ORDER BY s.customer_id)

SELECT customer_id, product_name, order_date, join_date
FROM order_after_member WHERE rnk = 1;
```

**Answer:**
|customer_id|product_name|order_date|join_date |
|-----------|------------|----------|--------- |
|A          |curry       |2021-01-07|2021-01-07|
|B          |sushi       |2021-01-11|2021-01-09|


## 7. Which item was purchased just before the customer became a member?

```mysql
WITH order_before_member AS (
SELECT 
	s.customer_id, s.order_date, mm.join_date, m.product_name,
	DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS rnk 
FROM sales s JOIN members mm ON s.customer_id = mm.customer_id
JOIN menu m ON s.product_id = m.product_id
WHERE s.order_date < DATE(mm.join_date)
ORDER BY s.customer_id) 

SELECT customer_id, product_name, order_date, join_date
FROM order_before_member WHERE rnk = 1;
```

**Answer:**
|customer_id|product_name|order_date|join_date |
|-----------|------------|----------|--------- |
|A          |sushi       |2021-01-01|2021-01-07|
|A          |curry       |2021-01-01|2021-01-07|
|B          |sushi       |2021-01-04|2021-01-09|


### 8. What is the total items and amount spent for each member before they became a member?

```mysql
WITH before_member AS ( 
SELECT s.customer_id, s.order_date, s.product_id, mm.join_date
FROM sales s JOIN members mm ON s.customer_id = mm.customer_id
WHERE s.order_date < mm.join_date)

SELECT b.customer_id, COUNT(*) AS items_count, 
	SUM(m.price) AS total_amount_spent 
FROM before_member b JOIN menu m ON b.product_id = m.product_id
GROUP BY b.customer_id
ORDER BY b.customer_id;
```

**Answer:**
|customer_id|items_count|total_amount_spent|
|-----------|-----------|------------------|
|A          |2          | 25               |
|B          |3          | 40               |


### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

```mysql
SELECT 
    s.customer_id,
    SUM(CASE
        WHEN m.product_id = 1 THEN m.price * 10 * 2
        ELSE m.price * 10
    END) AS points_collected
FROM
    menu m
        JOIN
    sales s ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id; 
```

**Answer:**
| customer_id |points_collected|
|-------------|----------------|
|A            | 860            |
|B            | 940            |
|C            | 360            |


### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customers A and B have at the end of January?

```mysql
WITH promo_week AS (
SELECT 
    customer_id, 
    join_date, 
    DATE_ADD(join_date, INTERVAL 6 DAY) AS valid_date
FROM members) 

SELECT
    p.customer_id, 
    SUM(CASE
		WHEN s.order_date BETWEEN p.join_date AND p.valid_date THEN price*10*2
        WHEN m.product_name = 'sushi' THEN m.price*10*2
        ELSE price*10 
    END) AS points_collected 
FROM sales s JOIN promo_week p ON s.customer_id = p.customer_id
    JOIN menu m ON s.product_id = m.product_id
      AND s.order_date <= '2021-01-31'
    GROUP BY p.customer_id
    ORDER BY p.customer_id;
```

**Answer:** 
| customer_id |points_collected|
|-------------|----------------|
|A            | 1370           |
|B            | 820            |


### BONUS QUESTIONS 

**Join All The Things**

```mysql
SELECT 
	s.customer_id, 
        s.order_date, 
        m.product_name, 
        m.price, 
        CASE
	    WHEN s.order_date < mm.join_date THEN 'N' 
            WHEN s.order_date >= mm.join_date THEN 'Y' 
            WHEN mm.join_date IS NULL THEN 'N'
        END AS 'member'
FROM sales s LEFT JOIN members mm
        ON s.customer_id = mm.customer_id 
    LEFT JOIN menu m 
        ON s.product_id = m.product_id;
```


 **Rank All The Things** 

 ```mysql
 WITH loyalty_program AS (
 SELECT 
	s.customer_id, 
        s.order_date, 
        m.product_name, 
        m.price, 
        CASE
	    WHEN s.order_date < mm.join_date THEN 'N' 
            WHEN s.order_date >= mm.join_date THEN 'Y' 
            WHEN mm.join_date IS NULL THEN 'N'
        END AS 'member'
FROM sales s LEFT JOIN members mm
          ON s.customer_id = mm.customer_id 
    LEFT JOIN menu m
          ON s.product_id = m.product_id) 

SELECT 
    customer_id, 
    order_date, 
    product_name, 
    price, 
    member, 
    IF(member = 'Y', RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date),
                                              'Null') AS ranking
FROM loyalty_program; 
```


















