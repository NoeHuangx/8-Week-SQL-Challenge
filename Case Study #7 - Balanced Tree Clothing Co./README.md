# 🌲Case Study #7: Balanced Tree Clothing Co. 

<p align='center'>
<img width='400px' src="https://8weeksqlchallenge.com/images/case-study-designs/7.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-7/).


## 📚Table of Contents
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [High Level Sales Analysis](#high-level-sales-analysis)
- [Transaction Analysis](#transaction-analysis)
- [Product Analysis](#product-analysis)
- [Bonus Challenge](#bonus-challenge)

***

## Business Task 
Balanced Tree Clothing Company provides a wide range of clothing and lifestyle wear to modern adventurers. 

The task is to analyze the sales performance and generate a basic financial report to share with the stakeholders. 

THe entity relationship diagram of the database is as follows: 

![CS7_ERD](https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/da0c9786-f996-40b1-85e3-2db33ab00828)

*** 

## Questions and Solutions 
### High Level Sales Analysis 
#### 1. What was the total quantity sold for all products? 

```mysql
SELECT 
	SUM(qty) AS total_qty 
FROM
	sales;
```

**Answer:** 
|total_qty|
|---------|
|45216|

#### 2. What is the total generated revenue for all products before discounts?

```mysql
SELECT 
	SUM(qty*price) AS revenue_before_discount
FROM 
	sales; 
```

**Answer:** 
|revenue_before_discount|
|-----------------------|
|1289453|

#### 3. What was the total discount amount for all products? 

```mysql
SELECT 
    ROUND(SUM((qty * price) * (discount / 100)), 2) AS total_discount
FROM
    sales; 
```

**Answer:**
|total_discount|
|--------------|
|156229.14|

### Transaction Analysis 
#### 1. How many unique transactions were there?

```mysql
SELECT 
	COUNT(DISTINCT txn_id) AS transactions_count
FROM 
	sales; 
```

**Answer:**
|transactions_count|
|------------------|
|2500|

#### 2. What is the average unique product purchased in each transaction? 

```mysql
WITH qty_cte AS (
	SELECT 
		txn_id, 
        COUNT(DISTINCT prod_id) AS item_qty
	FROM 
		sales
	GROUP BY 
		txn_id
)

SELECT
	AVG(item_qty) AS avg_unique_prodct 
FROM 
	qty_cte; 
```

**Answer:** 
|avg_unique_product|
|------------------|
|6.0380|

#### 3. What are the 25th, 50th and 75th percentile values for the revenue per transaction?

#### 4. What is the average discount value per transaction?

```mysql
WITH discount_cte AS (
	SELECT
		txn_id,
		SUM((qty*price)*(discount/100)) AS discount 
	FROM 
		sales 
	GROUP BY 
		txn_id
)

SELECT 
	ROUND(AVG(discount), 2) AS avg_discount
FROM 
	discount_cte; 
```

**Answer:** 
|avg_discount|
|------------|
|62.49|

#### 5. What is the percentage split of all transactions for members vs non-members?

```mysql
WITH count_cte AS (
	SELECT
		txn_id, 
		IF(member = 't', txn_id, NULL) AS members, 
        IF(member = 'f', txn_id, NULL) AS non_members 
	FROM 
		sales
)

SELECT 	
	ROUND(COUNT(DISTINCT members)/COUNT(DISTINCT txn_id)*100, 2) AS members_percent, 
    ROUND(COUNT(DISTINCT non_members)/COUNT(DISTINCT txn_id)*100,2) AS non_members_percent 
FROM 
	count_cte; 
```

**Answer:**
|members_percent|non_members_percent|
|---------------|-------------------|
|60.20|39.80|

#### 6. What is the average revenue for member transactions and non-member transactions?

```mysql
WITH transaction_cte AS (
	SELECT 
		txn_id, 
		member, 
        SUM(price*qty) AS total 
	FROM 
		sales 
	GROUP BY txn_id
)

SELECT 
	member, 
	ROUND(AVG(total), 2) AS avg_revenue 
FROM 
	transaction_cte
GROUP BY member; 
```

**Answer:** 
|member|avg_revenue|
|------|-----------|
|t|516.27|
|f|515.04|

### Product Analysis 
#### 1. What are the top 3 products by total revenue before discount? 

```mysql
SELECT 
	s.prod_id, 
    pd.product_name, 
    SUM(s.price*s.qty) AS total_revenue
FROM 
	sales s
JOIN product_details pd ON s.prod_id = pd.product_id
GROUP BY s.prod_id
ORDER BY total_revenue DESC
LIMIT 3; 
```

**Answer:**
|prod_id|product_name|total_revenue|
|2a2353|Blue Polo Shirt - Mens|217683|
|9ec847|Grey Fashion Jacket - Womens|209304|
|5d267b|White Tee Shirt - Mens|152000|

#### 2. What is the total quantity, revenue and discount for each segment?

```mysql
SELECT 
	pd.segment_id, 
    pd.segment_name, 
    SUM(s.qty) AS total_qty, 
    SUM(s.price * s.qty) AS total_revenue, 
    ROUND(SUM((s.price * s.qty)*(s.discount/100)), 2) AS total_discount 
FROM 
	sales s
JOIN product_details pd 
ON s.prod_id = pd.product_id 
GROUP BY pd.segment_id
ORDER BY pd.segment_id; 
```

**Answer:** 
|segment_id|segment_name|total_qty|total_revenue|total_discount|
|----------|------------|---------|-------------|--------|
|3|Jeans|11349|208350|25343.97|
|4|Jacket|11385|366983|44277.46|
|5|Shirt|11265|406143|49594.27|
|6|Socks|11217|307977|37013.44|

#### 3. What is the top selling product for each segment?

```mysql
WITH rnk_cte AS (
	SELECT 
		pd.segment_name,
		s.prod_id, 
		pd.product_name,
		SUM(s.qty) AS total_qty, 
		RANK() OVER(PARTITION BY pd.segment_name ORDER BY SUM(s.qty) DESC) AS rnk
	FROM 
		sales s 
	JOIN product_details pd 
	ON s.prod_id = pd.product_id
	GROUP BY s.prod_id, pd.segment_name
)

SELECT 
	segment_name, 
	product_name, 
    total_qty
FROM 
	rnk_cte 
WHERE 
	rnk = 1; 
```

**Answer:**
|segment_name|product_name|total_qty|
|------------|------------|---------|
|Jacket|Grey Fashion Jacket - Womens|3876|
|Jeans|Navy Oversized Jeans - Womens|3856|
|Shirt|Blue Polo Shirt - Mens|3819|
|Socks|Navy Solid Socks - Mens|3792|

#### 4. What is the total quantity, revenue, and discount for each category? 

```mysql
WITH total_qty_cte AS (
	SELECT 
		pd.category_name, 
		pd.product_name,
		SUM(s.qty) AS total_quantity,
		RANK() OVER(PARTITION BY pd.category_name ORDER BY SUM(s.qty) DESC) AS rnk
	FROM 
		sales s 
	JOIN product_details pd ON s.prod_id = pd.product_id 
	GROUP BY pd.product_name, pd.category_name
  ) 
  
SELECT
	category_name, 
    product_name, 
    total_quantity
FROM
	total_qty_cte
WHERE rnk = 1; 
```

**Answer:** 
|category_name|product_name|total_quantity|
|-------------|------------|--------------|
|Mens|Blue Polo Shirt - Mens|3819|
|Womens|Grey Fashion Jacket - Womens|3876|

#### 6. What is the percentage split of revenue by product for each segment?

```mysql
WITH product_revenue_cte AS (
	SELECT 
		pd.segment_name, 
		pd.product_name, 
		SUM(s.price*s.qty) AS product_revenue
	FROM sales s 
	JOIN product_details pd ON s.prod_id = pd.product_id
    GROUP BY pd.segment_name, pd.product_name
)

SELECT 
	*, 
    ROUND(100*product_revenue/SUM(product_revenue) OVER(PARTITION BY segment_name), 2) AS segment_revenue_split
FROM product_revenue_cte; 
```

**Answer:** 

*I have only stated the results of 2 `segment_name`: Jacket & Jeans. The actual result from this query provides the data for all segments.*
|segment_name|product_name|product_revenue|segment_revenue_split|
|------------|------------|---------------|------------|
|Jacket|Indigo Rain Jacket - Womens|71383|19.45|
|Jacket|Khaki Suit Jacket - Womens|86296|23.51|
|Jacket|Grey Fashion Jacket - Womens|209304|57.03|
|Jeans|Navy Oversized Jeans - Womens|50128|24.06|
|Jeans|Cream Relaxed Jeans - Womens|37070|17.79|
|Jeans|Black Straight Jeans - Womens|121152|58.15|

#### 7. What is the percentage split of revenue by segment for each category? 

```mysql
WITH segment_revenue_cte AS (
	SELECT
		pd.category_name, 
		pd.segment_name, 
		SUM(s.price*s.qty) AS segment_revenue
	FROM 
		sales s 
	JOIN product_details pd 
    ON s.prod_id = pd.product_id 
    GROUP BY pd.segment_name, pd.category_name
)

SELECT 
	*, 
    ROUND(
		100* segment_revenue/SUM(segment_revenue) OVER(PARTITION BY category_name)
    ,2) AS category_revenue_split 
FROM 
	segment_revenue_cte; 
```

**Answer:**
|category_name|segment_name|segment_revenue|category_revenue_split|
|-------------|------------|---------------|-----------|
|Mens|Shirt|406143|56.87|
|Mens|Socks|307977|43.13|
|Womens|Jeans|208350|36.21|
|Womens|Jacket|366983|63.79|

#### 8. What is the percentage split of total revenue by category?

```mysql
SELECT 
	pd.category_name, 
    ROUND(
		100*SUM(s.qty*s.price)/(SELECT SUM(qty*price) FROM sales)
    ,2) AS revenue_percent
FROM 
	sales s 
JOIN product_details pd ON s.prod_id = pd.product_id 
GROUP BY pd.category_name; 
```

**Answer:** 
|category_name|revenue_percent|
|-------------|---------------|
|Womens|44.62|
|Mens|55.38|

#### 9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)


```mysql
WITH product_cte AS (
	SELECT 
		DISTINCT s.prod_id,
        pd.product_name, 
        COUNT(DISTINCT s.txn_id) AS product_txn, 
        (SELECT COUNT(DISTINCT txn_id) FROM sales) AS total_txn
	FROM 
		sales s 
	JOIN product_details pd 
    ON s.prod_id = pd.product_id 
    GROUP BY prod_id
)

SELECT 
	*,
    ROUND(100*product_txn/total_txn,2) AS penetration_rate
FROM 
	product_cte;
```

**Answer:**
|prod_id|product_name|product_txn|total_txn|penetration_rate|
|-------|------------|-----------|---------|------------|
|2a2353|Blue Polo Shirt - Mens|1268|2500|50.72|
|2feb6b|Pink Fluro Polkadot Socks - Mens|1258|2500|50.32|
|5d267b|White Tee Shirt - Mens|1268|2500|50.72|
|72f5d4|Indigo Rain Jacket - Womens|1250|2500|50.00|
|9ec847|Grey Fashion Jacket - Womens|1275|2500|51.00|
|b9a74d|White Striped Socks - Mens|1243|2500|49.72|
|c4a632|Navy Oversized Jeans - Womens|1274|2500|50.96|
|c8d436|Teal Button Up Shirt - Mens|1242|2500|49.68|
|d5e9a6|Khaki Suit Jacket - Womens|1247|2500|49.88|
|e31d39|Cream Releaxed Jeans - Womens|1243|2500|49.72
|e83aa3|Black Straight Jeans - Womens|1246|2500|49.84|
|f084eb|Navy Solid Socks - Mens|1281|2500|51.24|

#### 10. What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction? 

```mysql
WITH transactions_count_cte AS (
	SELECT
		s.txn_id,
		pd.product_id, 
		pd.product_name, 
		COUNT(pd.product_id) OVER(PARTITION BY s.txn_id) AS txn_count
	FROM 
		sales s 
	JOIN  product_details pd 
	ON s.prod_id = pd.product_id
),

combination_cte AS (
	SELECT
		txn_id,
		GROUP_CONCAT(product_id ORDER BY product_id SEPARATOR ",") AS product_ids, 
		GROUP_CONCAT(product_name ORDER BY product_id SEPARATOR " ,") AS product_names
	FROM 
		transactions_count_cte 
	WHERE txn_count = 3 
	GROUP BY txn_id
), 

combination_count_cte AS (
	SELECT 
		product_ids, 
		product_names, 
		COUNT(*) AS combination_count
	FROM 
		combination_cte
	GROUP BY product_ids, product_names
)

SELECT 
	*
FROM 
	combination_count_cte 
WHERE combination_count = (SELECT MAX(combination_count)
							FROM combination_count_cte); 
```

**Answer:** 
|product_ids|product_names|combination_count|
|-----------|-------------|-----------------|
|5d267b,c4a632,e31d39|White Tee Shirt - Mens, Navy Oversized Jeans - Womens, Cream Relaxed Jeans - Womens|3|
|b9a74d,c4a632,d5e9a6|White Striped Socks - Mens, Navy Oversized Jeans - Womens, Khaki Suit Jacket - Womens|3|
|2a2353,2feb6b,c4a632|Blue Polo Shirt - Mens, Pink Fluro Polkadot Socks - Mens, Navy Oversized Jeans - Womens|3|
|5d267b,c4a632,e83aa3|White Tee Shirt - Mens, Navy Oversized Jeans - Womens, Black Straight Jeans - Womens|3|
|c4a632,c8d436,e83aa3|Navy Oversized Jeans - Womens, Teal Button Up Shirt - Mens, Black Straight Jeans - Womens|3|

### Bonus Challenge 
#### Use a single SQL query to transform the `product_hierarchy` and `product_prices` datasets to the `product_details` table. 
#### Hint: You may want to consider using a recursive CTE to solve this problem. 

```mysql
SELECT 
	pp.product_id, 
    pp.price, 
    CONCAT(ph1.level_text, " ", ph2.level_text, " - ", ph3.level_text) AS product_name, 
    ph3.id AS category_id, 
    ph2.id AS segment_id, 
    ph1.id AS style_id, 
    ph3.level_text AS category_name, 
    ph2.level_text AS segment_name, 
    ph1.level_text AS style_name
FROM product_hierarchy ph1
JOIN product_prices pp ON ph1.id = pp.id
JOIN product_hierarchy ph2 ON ph1.parent_id = ph2.id 
JOIN product_hierarchy ph3 ON ph2.parent_id = ph3.id
ORDER BY style_id; 
```

**Answer:** 


<img width="605" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/229f18bb-b3fd-4d49-95aa-08e5a67de494">










