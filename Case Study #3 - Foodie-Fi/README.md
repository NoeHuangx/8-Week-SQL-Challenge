# 🥑Case Study #3: Foodie Fi 

<p align='center'>
<img width="400px" src ="https://8weeksqlchallenge.com/images/case-study-designs/3.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-3/).


## 📚Table of Contents
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [A. Customer Journey](#a-customer-journey)
- [B. Data Analysis Questions](#b-data-analysis-questions)
- [C. Challenge Payment Question](#c-challenge-payment-question)

*** 

## Business Task 
Danny wants to make all the future investment decisions and create new features using data. 

Danny has provided an entity relationship diagram of his database as below: 
<p align="center">
<img width="514" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/73e003a9-6464-4ec7-9de4-31a6ac4bfe54">
</p>

*** 

## Questions and Solutions
### A. Customer Journey 
#### Based on the 8 sample customers provided in the sample from the subscriptions table, write a brief description of each customer’s onboarding journey.

**Table: `subscriptions` sample**
<p align="center">
<img width="209" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/f0788f49-318c-4632-9996-29485b5bd293">
</p>

```mysql
SELECT 
    s.customer_id, s.start_date, p.plan_name
FROM
    subscriptions s
        JOIN
    plans p ON s.plan_id = p.plan_id
    WHERE customer_id in (1,2,11,13); 
```

**Answer:**
|customer_id|start_date|plan_name| 
|-----------|----------|---------|
|1          |2020-08-01|trail    |
|1          |2020-08-08|basic monthly|
|2          |2020-09-20|trial    |
|2          |2020-09-27|pro annual| 
|11         |2020-11-19|trial    |
|11         |2020-11-26|churn    | 
|13         |2020-12-15|trial    |
|13         |2020-12-22|basic monthly|
|13         |2021-03-29|pro monthly| 

- All customers from the sample start by joining the 7-day free trial, and then, are changed to one of the paid subscription plans. Some customers might also cancel right after the free trial ends.
- `customer_id` with 1, 2, 11, and 13 are selected as examples. 


### B. Data Analysis Questions 
#### 1. How many customers has Foodie-Fi ever had?

```mysql
SELECT 
    COUNT(DISTINCT customer_id) AS customer_num
FROM
    subscriptions; 
```

**Answer:** 
|customer_num|
|------------|
|1000|


#### 2. What is the monthly distribution of trial plan start_date values for our dataset? - use the start of the month as the group by value. 

```mysql
SELECT 
    MONTHNAME(start_date) AS months,
    COUNT(customer_id) AS trial_count
FROM
    subscriptions
WHERE
    plan_id = 0
GROUP BY months
ORDER BY FIELD(months,
        'Janurary',
        'February',
        'March',
        'April',
        'May',
        'June',
        'July',
        'August',
        'September',
        'October',
        'November',
        'December');
```

**Answer:**
|months  |trial_count| 
|--------|-----------|
|January |88|
|February|68|
|March|94|
|April|81|
|May|88|
|June|79|
|July|89|
|August|88|
|September|87|
|October|79|
|November|75|
|December|84|


#### 3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name.

```mysql
SELECT 
    p.plan_name, p.plan_id, COUNT(*) AS count
FROM
    subscriptions s
        JOIN
    plans p ON p.plan_id = s.plan_id
        AND YEAR(s.start_date) > 2020
GROUP BY p.plan_id
ORDER BY p.plan_id; 
```

**Answer:**
|plan_name|plan_id|count|
|---------|-------|-----|
|basic monthly|1|8|
|pro monthly|2|60|
|pro annual|3|63|
|churn|4|71|

**Note:** I have disabled the `ONLY_FULL_GROUP_BY` mode for the session running queries with the following code: 
```mysql
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```


#### 4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

```mysql
WITH customers AS 
(SELECT 
	COUNT(DISTINCT customer_id) AS total_customer_count, 
	SUM(
    CASE
		WHEN plan_id = 4 THEN 1
        ELSE 0
    END) AS churn_count 
FROM subscriptions) 

SELECT 
	churn_count, 
    ROUND((churn_count/total_customer_count)*100, 1) AS churn_percent
FROM 
	customers;
```

**Answer:**
|churn_count|churn_percent|
|-----------|-------------|
|307|30.7|


#### 5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

```mysql
WITH churn_cte AS (
SELECT 
	customer_id,
    GROUP_CONCAT(plan_id) AS plans
FROM subscriptions
GROUP BY customer_id), 

count_cte AS
(SELECT 
	SUM(
    CASE
		WHEN plans = '0,4' THEN 1 
        ELSE 0 
    END
    ) AS churn_count, 
    COUNT(*) as total_customer 
FROM churn_cte)

SELECT
    ROUND((churn_count/total_customer) *100, 0) AS churn_percentage
FROM count_cte; 
```

**Answer:** 
|churn_percentage|
|----------------|
|9               |


#### 6. What is the number and percentage of customer plans after their initial free trial?

```mysql
WITH next_plan_cte AS (
	SELECT 
		s.customer_id,
        s.plan_id,
        p.plan_name, 
        LAG(s.plan_id) OVER(PARTITION BY s.customer_id) AS last_plan
	FROM 
		subscriptions s 
	JOIN plans p 
    ON s.plan_id = p.plan_id)
    
SELECT 
	plan_name, 
    COUNT(*) as customer_count,
    ROUND((COUNT(*)/(SELECT COUNT(DISTINCT customer_id) FROM subscriptions))*100, 2) AS percentage
FROM next_plan_cte
WHERE last_plan = 0 
GROUP BY plan_name; 
```

**Answer:**
|plan_name|customer_count|percentage|
|---------|--------------|----------|
|basic monthly|546|54.60|
|pro annual|37|3.70|
|pro monthly|325|32.50|
|churn|92|9.20|


#### 7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

```mysql
WITH plan_cte AS (
SELECT 
    s.plan_id, 
    p.plan_name, 
    s.start_date, 
    LEAD(start_date) OVER(PARTITION BY customer_id) AS next_date
FROM subscriptions s
JOIN plans p
ON s.plan_id = p.plan_id) 

SELECT 
	plan_id, 
    plan_name, 
    COUNT(*) AS count,
    ROUND((COUNT(*)/(SELECT COUNT(DISTINCT customer_id) FROM subscriptions)* 100), 2) AS percentage 
FROM plan_cte 
WHERE (start_date <= '2020-12-31') AND (next_date > '2020-12-31' OR next_date IS NULL)
GROUP BY plan_id
ORDER BY plan_id;
```

**Answer:**
|plan_id|plan_name|count|percentage|
|-------|---------|-----|----------|
|0|trial|19|1.90|
|1|basic monthly|224|22.40|
|2|pro monthly|326|32.60|
|3|pro annual|195|19.50|
|4|churn|236|23.60|


#### 8. How many customers have upgraded to an annual plan in 2020?

```mysql
SELECT 
    COUNT(*) AS num_customer
FROM
    subscriptions
WHERE
    plan_id = 3 AND YEAR(start_date) = 2020; 
```

**Answer:** 
|num_customer|
|------------|
|195|


#### 9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

```mysql
WITH annual_plan AS (
	SELECT 
		customer_id, 
        start_date AS annual_date
	FROM subscriptions 
    WHERE plan_id = 3
), 

trial_plan AS (
	SELECT 
		customer_id, 
        start_date AS trial_date 
	FROM subscriptions 
    WHERE plan_id = 0 
)

SELECT 
	ROUND(AVG(DATEDIFF(annual_date, trial_date)), 0) AS avg_days 
FROM annual_plan ap 
JOIN trial_plan tp 
ON ap.customer_id = tp.customer_id; 
```

**Answer:**
|avg_days|
|--------|
|105|


#### 10. Can you further breakdown this average value into 30-day periods (i.e. 0-30 days, 31-60 days, etc)?

```mysql
WITH annual_plan AS (
	SELECT 
		customer_id, 
        start_date AS annual_date
	FROM subscriptions 
    WHERE plan_id = 3
), 
trial_plan AS (
	SELECT 
		customer_id, 
        start_date AS trial_date 
	FROM subscriptions 
    WHERE plan_id = 0 
)

SELECT 
	CONCAT(FLOOR(DATEDIFF(annual_date, trial_date)/30)*30, '-', FLOOR(DATEDIFF(annual_date, trial_date)/30)*30 + 30) AS '30-day range', 
    COUNT(*) AS count 
FROM annual_plan ap 
JOIN trial_plan tp 
ON ap.customer_id = tp.customer_id 
GROUP BY FLOOR(DATEDIFF(annual_date, trial_date)/30)*30
ORDER BY FLOOR(DATEDIFF(annual_date, trial_date)/30)*30; 
```

**Answer:**
|30-day range|count|
|------------|-----|
|0-30|48|
|30-60|25|
|60-90|33|
|90-120|35|
|120-150|43|
|150-180|35|
|180-210|27|
|210-240|4|
|240-270|5|
|270-300|1|
|300-330|1|
|330-360|1|


#### 11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

```mysql
WITH next_cte AS (
SELECT 
	*,
    LEAD(plan_id) OVER(PARTITION BY customer_id) AS next_plan 
FROM subscriptions
WHERE YEAR(start_date) = 2020)

SELECT 
	COUNT(DISTINCT customer_id) AS num_customer 
FROM next_cte n
WHERE plan_id = 2 AND next_plan = 1; 
```

**Answer:**
|num_customer|
|------------|
|0|


### C. Challenge Payment Question
#### The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:
- **monthly payments always occur on the same day of the month as the original start_date of any monthly paid plan**
- **upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately**
- **upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period**
- **once a customer churns they will no longer make payments**

```mysql
DROP TEMPORARY TABLE IF EXISTS paid_plans; 
CREATE TEMPORARY TABLE paid_plans(
	SELECT 
		s.customer_id, 
        s.plan_id, 
        p.plan_name, 
        CASE
			WHEN s.plan_id = 1 THEN 9.90
            WHEN s.plan_id = 2 THEN 19.90
            WHEN s.plan_id = 3 THEN 199.90
            ELSE 0
        END AS amount, 
        s.start_date,
        LEAD(plan_name) OVER(PARTITION BY s.customer_id ORDER BY s.start_date) AS next_plan 
	FROM 
		subscriptions s
	JOIN 
		plans p 
	ON p.plan_id = s.plan_id 
    WHERE s.plan_id != 0 AND 
    year(s.start_date) = 2020 
);

DROP TEMPORARY TABLE IF EXISTS payment_end_date; 
CREATE TEMPORARY TABLE payment_end_date(
	SELECT *, 
		start_date AS payment_date, 
		CASE
			-- for monthly plan 
			WHEN next_plan IS NULL AND plan_id != 3 THEN '2020-12-31'
			-- If customer upgrades from pro monthly to pro annual, monthly price ends the month before the annual plan start date 
			WHEN plan_id = 2 AND next_plan = 'pro annual' THEN DATE_SUB(LEAD(start_date) OVER(PARTITION BY customer_id ORDER BY start_date), INTERVAL 1 MONTH)
			-- if customer churns or upgrade plans, change start_date 
			WHEN next_plan = 'churn' OR next_plan = 'pro monthly' OR next_plan = 'pro annual' THEN LEAD(start_date) OVER(PARTITION BY customer_id ORDER BY start_date)
			-- if customer upgrades to pro annual after trial 
			WHEN plan_id = 3 THEN start_date
		END AS end_date
	FROM 
		paid_plans
);

DROP TABLE IF EXISTS payments;
CREATE TABLE payments AS
WITH RECURSIVE date_cte AS(
	SELECT
		customer_id,
        plan_id, 
        plan_name, 
        amount,
        start_date,
        payment_date,
        next_plan,
        end_date
	FROM payment_end_date 
    
    UNION ALL 
    
	SELECT 
		d.customer_id, 
        d.plan_id, 
        d.plan_name, 
        d.amount, 
        d.start_date, 
        DATE_ADD(d.payment_date, INTERVAL 1 MONTH), 
        d.next_plan, 
        d.end_date
	FROM date_cte d
    WHERE DATE_ADD(d.payment_date, INTERVAL 1 MONTH) <= d.end_date
    
), 

final_cte AS(
	SELECT 
		customer_id, 
        plan_id, 
        plan_name, 
        payment_date, 
        CASE
			WHEN LAG(plan_id) OVER(PARTITION BY customer_id ORDER BY start_date) < plan_id
            AND 
            LAG(plan_id) OVER(PARTITION BY customer_id ORDER BY start_date) != 3
            AND
            MONTH(LAG(payment_date) OVER(PARTITION BY customer_id ORDER BY payment_date)) = MONTH(payment_date) 
            THEN amount - LAG(amount) OVER(PARTITION BY customer_id ORDER BY payment_date)
            ELSE amount
        END AS amount, 
        ROW_NUMBER() OVER(PARTITION BY customer_id) AS payment_order
	FROM date_cte
)

SELECT 
	*
FROM final_cte
WHERE plan_id != 4
ORDER BY customer_id, payment_date; 
```

For some samples of the `payments` table: 
```mysql
SELECT 
    *
FROM
    payments
WHERE
    customer_id IN (1, 2, 13, 15, 16, 19);
```

**Answer:** 
|customer_id|plan_id|plan_name|payment_date|amount|payment_order|
|-----------|-------|---------|------------|------|-------------|
|1|1|basic monthly|2020-08-08|9.90|1|
|1|1|basic monthly|2020-09-08|9.90|2|
|1|1|basic monthly|2020-10-08|9.90|3|
|1|1|basic monthly|2020-11-08|9.90|4|
|1|1|basic monthly|2020-12-08|9.90|5|
|2|3|pro annual|2020-09-27|199.90|1|
|13|1|basic monthly|2020-12-22|9.90|1|
|15|2|pro monthly|2020-03-24|19.90|1|
|15|2|pro monthly|2020-04-24|19.90|2|
|16|1|basic monthly|2020-06-07|9.90|1|
|16|1|basic monthly|2020-07-07|9.90|2|
|16|1|basic monthly|2020-08-07|9.90|3|
|16|1|basic monthly|2020-09-07|9.90|4|
|16|1|basic monthly|2020-10-07|9.90|5|
|16|3|pro annual|2020-10-21|190.00|6|
|19|2|pro monthly|2020-06-29|19.90|1|
|19|2|pro monthly|2020-07-29|19.90|2|
|19|3|pro annual|2020-08-29|199.90|3|
















