# 🪙Case Study #4: Data Bank

<p align='center'>
<img width='400px' src="https://8weeksqlchallenge.com/images/case-study-designs/4.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-4/)


## 📚Table of Contents 
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [A. Customer Nodes Exploration](#a-customer-nodes-exploration)
- [B. Customer Transactions](#b-customer-transactions)


*** 

## Busines Task 
The management team at Data Bank wants to increase their total customer base and needs help tracking how much data storage their customers will need. 

The team has provided an entity relationship diagram of the database as below: 
<p align='center'>
<img width="431" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/d7ecac7f-101e-48c7-b5cc-3060ba05abd1">
</p>

*** 

## Questions and Solutions 
### A. Customer Nodes Exploration
#### 1. How many unique nodes are there on the Data Bank system? 

```mysql
SELECT 
	COUNT(DISTINCT node_id) AS node_count 
FROM 
	customer_nodes;
```

**Answer:** 
|node_count|
|----------|
|5         |


#### 2. What is the number of nodes per region?

```mysql
SELECT 
    r.region_name,
    r.region_id,
    COUNT(DISTINCT c.node_id) AS node_count
FROM
    regions r
        JOIN
    customer_nodes c ON r.region_id = c.region_id
GROUP BY r.region_id
ORDER BY r.region_id;
```

**Answer:**
|region_name|region_id|node_count|
|-----------|---------|----------|
|Australia  |1        |5         |
|America    |2        |5         |
|Africa     |3        |5         |
|Asia       |4        |5         |
|Europe     |5        |5         |


#### 3. How many customers are allocated to each region?

```mysql
SELECT 
	r.region_name, 
    r.region_id, 
    COUNT(DISTINCT c.customer_id) AS customer_count
FROM 
	regions r
JOIN 
	customer_nodes c ON r.region_id = c.region_id
GROUP BY r.region_id
ORDER BY r.region_id; 
```

**Answer:** 
|region_name|region_id|customer_count|
|-----------|---------|--------------|
|Australia  |1|110|
|America|2|105|
|Africa|3|102|
|Asia|4|95|
|Europe|5|88|


#### 4. How many days on average are customers reallocated to a different node?

```mysql
-- To check if start_date and end_date are correct
SELECT DISTINCT
    start_date
FROM
    customer_nodes
ORDER BY start_date DESC;

SELECT DISTINCT
    end_date
FROM
    customer_nodes
ORDER BY end_date DESC;

-- calculate the average number of days 
SELECT 	
	FLOOR(AVG(DATEDIFF(end_date, start_date))) AS avg_days
FROM 
	customer_nodes
WHERE YEAR(end_date) != 9999; 
```

**Answer:** 
|avg_days|
|--------|
|14|


#### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?


### B. Customer Transactions 
#### 1. What is the unique count and total amount for each transaction type?

```mysql
SELECT 
    txn_type,
    COUNT(*) AS count,
    SUM(txn_amount) AS total
FROM
    customer_transactions
GROUP BY txn_type; 
```

**Answer:**
|txn_type|count|total|
|--------|-----|-----|
|deposit|2671|1359168|
|withdrawal|1580|793003|
|purchase|1617|806537|


#### 2. What is the average total historical deposit counts and amounts for all customers?

```mysql
WITH cte AS (
SELECT 
	COUNT(*) AS deposit_count, 
    SUM(txn_amount) AS amount
FROM 
	customer_transactions
WHERE txn_type = "deposit"
GROUP BY customer_id)

SELECT
	AVG(deposit_count) AS avg_count, 
    AVG(amount) AS avg_amount 
FROM cte; 
```

**Answer:** 
|avg_count|avg_amount|
|---------|----------|
|5.3420|2718.3360|


#### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

```mysql
WITH cte AS (
	SELECT 
		customer_id, 
		MONTHNAME(txn_date) AS month_name,
        MONTH(txn_date) AS month_id, 
        SUM(IF(txn_type = "deposit", 1, 0)) AS deposit_count, 
        SUM(IF(txn_type = "purchase", 1, 0)) AS purchase_count, 
        SUM(IF(txn_type = "withdrawal", 1, 0)) AS withdrawal_count
	FROM customer_transactions
    GROUP BY customer_id, month_name
)

SELECT 
	month_name, 
    COUNT(*) AS count 
FROM cte
WHERE deposit_count > 1 AND (purchase_count = 1 OR withdrawal_count = 1) 
GROUP BY month_name
ORDER BY month_id; 
```

**Answer:**
|month_name|count|
|----------|-----|
|January   |115|
|February  |108|
|March     |113|
|April|50| 


#### 4. What is the closing balance for each customer at the end of the month?

```mysql
WITH cte AS (
	SELECT 
		customer_id, 
        MONTHNAME(txn_date) AS month_name,
        MONTH(txn_date) AS month_id, 
        SUM(IF(txn_type = "deposit", txn_amount, -1 * txn_amount)) AS amount 
	FROM customer_transactions 
    GROUP BY customer_id, MONTHNAME(txn_date)
)

SELECT 
	customer_id, 
    month_id, 
	month_name, 
    SUM(amount) OVER(PARTITION BY customer_id ORDER BY month_id) AS total_amount 
FROM cte; 
```

**Answer:**
|customer_id|month_id|month_name|total_amount|
|-----------|--------|----------|------------|
|1|1|January|312|
|1|3|March|-640|
|2|1|January|549|
|2|3|March|610|
|3|1|January|144|
|3|2|February|-821|
|...||||
|500|1|January|1594|
|500|2|February|2981|
|500|3|March|2251|


#### 5. What is the percentage of customers who increase their closing balance by more than 5%? 

```mysql
WITH cte AS (
	SELECT 
		customer_id, 
        MONTHNAME(txn_date) AS month_name, 
        MONTH(txn_date) AS month_id, 
        SUM(IF(txn_type = "deposit", txn_amount, -1 * txn_amount)) AS amount 
	FROM customer_transactions 
    GROUP BY customer_id, MONTHNAME(txn_date) 
), 

total AS (
	SELECT 
		customer_id, 
        month_id,
        month_name, 
        SUM(amount) OVER(PARTITION BY customer_id ORDER BY month_id) AS total_amount 
	FROM cte
),

balance AS (
	SELECT 
		*,
        LAG(total_amount) OVER(PARTITION BY customer_id ORDER BY month_id) AS start_b,
        total_amount AS end_b
	FROM total
),

percent AS (
	SELECT 
		customer_id, 
		((end_b - start_b)*100)/start_b AS change_percent 
	FROM balance 
) 

SELECT 
	(COUNT(DISTINCT customer_id) * 100) /(SELECT COUNT(DISTINCT customer_id) FROM customer_transactions) AS customer_percent 
FROM percent 
WHERE change_percent > 5; 
```

**Answer:**
|customer_percent|
|----------------|
|75.80|



