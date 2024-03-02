# 🐟Case Study #6: Clique Bait

<p align='center'>
<img width='400px' src="https://8weeksqlchallenge.com/images/case-study-designs/6.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-6/).


## 📚Table of Contents 
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [A. Digital Analysis](#a-digital-analysis)
- [B. Product Funnel Analysis](#b-product-funnel-analysis)
- [C. Campaigns Analysis](#c-campaigns-analysis)


***

## Business Task 
Clique Bait is an online seafood store. 

The task is to support Danny's vision, analyze the datasets, and develop creative solutions to calculate funnel fallout rates for the store. 

The entity relationship diagram of the database is as follows: 

![CS6_ERD](https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/66e93a55-8c9c-425f-b0e0-7fd30e0acd6a)

*** 

## Questions and Solutions
### A. Digital Analysis 
#### 1. How many users are there?

```mysql
SELECT 
    COUNT(DISTINCT user_id) AS user_count
FROM
    users; 
```

**Answer:**
|user_count|
|----------|
|500|

#### 2. How many cookies does each user have on average?

```mysql
WITH cookie_count AS (
	SELECT 
		COUNT(cookie_id) AS cookie_count
	FROM 
		users 
	GROUP BY 	
		user_id
)

SELECT 
	ROUND(AVG(cookie_count), 2) AS avg_cookie_count
FROM 
	cookie_count;
```

**Answer:**
|avg_cookie_count|
|----------------|
|3.56|

#### 3. What is the unique number of visits by all users per month? 

```mysql
SELECT 
	MONTHNAME(event_time) AS month_name, 
	COUNT(DISTINCT visit_id) AS visit_num
FROM 
	events
GROUP BY 
	MONTH(event_time)
ORDER BY 
	MONTH(event_time); 
```

**Answer:**
|month_name|visit_num|
|----------|---------|
|January|876|
|February|1488|
|March|916|
|April|248|
|May|36|

*I have disabled the `ONLY_FULL_GROUP_BY` mode with the following code.* 

```mysql
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```

#### 4. What is the number of events for each event type? 

```mysql
SELECT 
	ed.event_name, 
    e.event_type, 
    COUNT(*) AS event_num
FROM 
	events e 
JOIN event_identifier ed ON e.event_type = ed.event_type
GROUP BY ed.event_name
ORDER BY e.event_type; 
```

**Answer:**
|event_name|event_type|event_num|
|----------|----------|---------|
|Page View|1|20928|
|Add to Cart|2|8451|
|Purchase|3|1777|
|Ad Impression|4|876|
|Ad Click|5|702|

#### 5. What is the percentage of visits which have a purchase event? 

```mysql
SELECT 
    ROUND(100 * COUNT(DISTINCT visit_id) / (SELECT 
                    COUNT(DISTINCT visit_id)
                FROM
                    events), 2) AS purchase_event_percent
FROM
    events
WHERE
    event_type = 3; 
```

**Answer:**
|purchase_event_percent|
|----------------------|
|49.86|

#### 6. What is the percentage of visits which view the checkout page but do not have a purchase event?

```mysql
WITH checkout_CTE AS (
	SELECT 
		SUM(IF(p.page_name = 'Checkout', 1, 0)) AS checkout_num, 
        SUM(IF(p.page_name = 'Confirmation', 1, 0)) AS confirmation_num
	FROM 
		events e
	JOIN page_hierarchy p ON p.page_id = e.page_id
)

SELECT
	ROUND(100*(checkout_num - confirmation_num)/checkout_num, 2) AS no_purchase_percent
FROM 
	checkout_CTE; 
```

**Answer:**
|no_purchase_percent|
|-------------------|
|15.50|

#### 7. What are the top 3 pages by number of views?

```mysql
SELECT 
    p.page_id, p.page_name, COUNT(*) AS views_num
FROM
    page_hierarchy p
        JOIN
    events e ON p.page_id = e.page_id
GROUP BY p.page_id
ORDER BY views_num DESC
LIMIT 3; 
```

**Answer:**
|page_id|page_name|views_num|
|-------|---------|---------|
|2|All Products|4752|
|9|Lobster|2515|
|10|Crab|2513|

#### 8. What is the number of views and cart adds for each product category?

```mysql
SELECT 
    p.product_category,
    SUM(IF(e.event_type = 1, 1, 0)) AS page_views,
    SUM(IF(e.event_type = 2, 1, 0)) AS cart_adds_count
FROM
    events e
        JOIN
    page_hierarchy p ON e.page_id = p.page_id
WHERE
    p.product_category IS NOT NULL
GROUP BY p.product_category
ORDER BY p.product_category; 
```

**Answer:**
|product_category|page_veiws|cart_adds_count|
|----------------|----------|---------------|
|Fish|4633|2789|
|Luxury|3032|1870|
|Shellfish|6204|3792|

#### 9. What are the top 3 products by purchase? 

```mysql
SELECT 
    p.page_name,
    SUM(IF(e.event_type = 2, 1, 0)) AS total_purchase
FROM
    events e
        JOIN
    page_hierarchy p ON e.page_id = p.page_id
WHERE
    p.product_id IS NOT NULL
GROUP BY p.page_name
ORDER BY total_purchase DESC
LIMIT 3; 
```

**Answer:**
|page_name|total_purchase|
|---------|--------------|
|Lobster|968|
|Crab|949|
|Russian Caviar|946|

### B. Product Funnel Analysis 

Using a single SQL query - create a new output table which has the following details:

How many times was each product viewed?
How many times was each product added to cart?
How many times was each product added to a cart but not purchased (abandoned)?
How many times was each product purchased?
Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

```mysql
DROP TABLE IF EXISTS product_info; 
CREATE TABLE product_info (
WITH product_info AS (
	SELECT 
		e.visit_id,
        e.event_type, 
        p.page_name AS product_name, 
        p.product_category, 
        p.product_id
	FROM 
		events e
	JOIN page_hierarchy p ON p.page_id = e.page_id       
), 

page_cart AS (
	SELECT 
		product_id, 
        product_name, 
        product_category, 
        CASE
			WHEN event_type = 1 THEN visit_id 
        END AS views_id, 
        CASE
			WHEN event_type = 2 THEN visit_id 
        END AS cart_id 
	FROM 
		product_info
), 

purchase_cte AS (
	SELECT 
		visit_id AS purchase_id 
	FROM events 
	WHERE event_type = 3
)

SELECT
	product_id, 
    product_name, 
    product_category, 
    COUNT(views_id) AS page_views, 
    COUNT(cart_id) AS added_to_cart, 
    COUNT(cart_id) - COUNT(purchase_id) AS abandoned,
    COUNT(purchase_id) AS purchase_count
FROM 	
	page_cart pc
LEFT JOIN purchase_cte p ON pc.cart_id = p.purchase_id
WHERE product_id IS NOT NULL
GROUP BY product_id
ORDER BY product_id); 
```

**Answer:**
|product_id|product_name|product_catetory|page_views|added_to_cart|abandoned|purchase_count|
|----------|------------|----------------|----------|-------------|---------|---------------|
|1|Salmon|Fish|1559|938|227|711|
|2|Kingfish|Fish|1559|920|213|707|
|3|Tuna|Fish|1515|931|234|697|
|4|Russian Caviar|Luxury|1563|946|249|697|
|5|Black Truffle|Luxury|1469|924|217|707|
|6|Abalone|Shellfish|1525|932|233|699|
|7|Lobster|Shellfish|1547|968|214|754|
|8|Crab|Shellfish|1564|949|230|719|
|9|Oyster|Shellfish|1568|943|217|726|








