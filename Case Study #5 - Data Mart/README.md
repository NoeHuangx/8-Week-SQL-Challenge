# 🧺Case Study #5: Data Mart 

<p align='center'>
<img width='400px' src="https://8weeksqlchallenge.com/images/case-study-designs/5.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-5/).


## 📚Table of Contents 
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [A. Data Cleansing Steps](#a-data-cleansing-steps)
- [B. Data Exploration](#b-data-exploration)
- [C. Before & After Analysis](#c-before-and-after-analysis)
- [D. Bonus Question](#d-bonus-question)


*** 

## Business Task
Danny's latest venture is Data Mart, which is an international online supermarket that specializes in fresh produce. In June 2020, large-scale supply changes were made at Data Mart. All Data Mart products now use sustainable packaging methods in every single step from the farm to the customers. 

The key business questions Data Mart's team needs help with are the following: 
- What was the quantifiable impact of the changes introduced in June 2020?
- Which platform, region, segment, and customer types were impacted most by this change?
- What can we do about the future introduction of similar sustainability updates to the business to minimize the impact on sales?

The entity relationship diagram of the database is as below: 
<p align='center'>
<img width="226" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/d46f7120-8d6c-4f0a-95c0-3d7cc9d8d710">
</p>

*** 

## Questions and Solutions
### A. Data Cleansing Steps
In a single query, perform the following operations and generate a new table in the `data_mart` schema named `clean_weekly_sales`: 

- Convert the `week_date` to a `DATE` format
- Add a `week_number` as the second column for each `week_date` value, for example, any value from the 1st of January to the 7th of January will be 1, the 8th to 14th will be 2, etc.
- Add a `month_number` with the calendar month for each `week_date` value as the 3rd column
- Add a `calendar_year` column as the 4th column containing either 2018, 2019 or 2020 values
- Add a new column called `age_band` after the original `segment` column using the following mapping on the number inside the `segment` value

|segment|age_band|
|-------|--------|
|1|Young Adults|
|2|Middle Aged|
|3 or 4| Retirees|

- Add a new `demographic` column using the following mapping for the first letter in the `segment` values:

|segment|demographic|
|-------|-----------|
|C|Couples|
|F|Families|

- Ensure all `null` string values with an `unknown` string value in the original `segment` column as well as the new `age_band` and `demographic` columns
- Generate a new `avg_transaction` column as the `sales` value divided by `transactions` rounded to 2 decimal places for each record 

```mysql
DROP TABLE IF EXISTS clean_weekly_sales; 
CREATE TABLE clean_weekly_sales (
	SELECT 
		STR_TO_DATE(week_date, '%d/%m/%Y') AS week_date,
		WEEKOFYEAR(STR_TO_DATE(week_date, '%d/%m/%Y')) AS week_number, 
		MONTH(STR_TO_DATE(week_date, '%d/%m/%Y')) AS month_number, 
		YEAR(STR_TO_DATE(week_date, '%d/%m/%Y')) AS calendar_year, 
		region, 
		platform, 
		CASE 
			WHEN segment LIKE '%null%' THEN NULL 
			ELSE segment 
		END AS segment, 
		CASE 
			WHEN RIGHT(segment, 1) = '1' THEN 'Young Adults'
			WHEN RIGHT(segment, 1) = '2' THEN 'Middle Aged'
            WHEN RIGHT(segment, 1) in ('3', '4') THEN 'Retirees'
			ELSE NULL
		END AS age_band, 
        CASE 
			WHEN LEFT(segment, 1) = 'C' THEN 'Couples'
            WHEN LEFT(segment, 1) = 'F' THEN 'Families'
            ELSE NULL
        END AS demographic, 
		customer_type, 
        transactions, 
        sales, 
        ROUND(sales/transactions, 2) AS avg_transaction 
	FROM 
		weekly_sales
);

DESC clean_weekly_sales; 
```

**Answer:**

<img width="280" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/3a2e9f27-dda2-46df-a97b-16ef57f30605">

<img width="766" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/8d5d0553-f8ca-422c-a7e1-97619727a176">


### B. Data Exploration 
#### 1. What day of the week is used for each `week_date` value? 

```mysql
SELECT DISTINCT
    (DAYNAME(week_date)) AS week_day
FROM
    clean_weekly_sales
GROUP BY week_date; 
```

**Answer:** 
|week_day|
|--------|
|Monday|

#### 2. What range of week numbers are missing from the dataset? 

```mysql
WITH RECURSIVE week_cte AS (
	SELECT 1 AS pos 
    UNION ALL 
	SELECT pos + 1 FROM week_cte
    WHERE pos+1 <= 52) 

SELECT DISTINCT(w.pos), 
	c.week_number
FROM week_cte w 
LEFT JOIN clean_weekly_sales c 
ON w.pos = c.week_number
WHERE c.week_number IS NULL; 
```

**Answer:** 

_I'm only posting the results of 8 rows here. The actual result I retrieved is 28 rows._

|pos|week_number|
|---|-----------|
|1|NULL|
|2|NULL|
|3|NULL|
|4|NULL|
|...|....|
|49|NULL|
|50|NULL|
|51|NULL|
|52|NULL|

- The range of `week_number` missing from the dataset is from week 1 to week 12 and week 37 to week 52. 

#### 3. How many total transactions were there for each year in the dataset? 

```mysql
SELECT 
	calendar_year, 
	SUM(transactions) AS total_transactions
FROM 
	clean_weekly_sales 
GROUP BY 
	calendar_year
ORDER BY 
	calendar_year; 
```

**Answer:**
|calendar_year|total_transactions|
|-------------|------------------|
|2018|346406460|
|2019|365639285|
|2020|375813651|

#### 4. What is the total sales for each region for each month?

```mysql
SELECT 
	region, 
    month_number, 
    SUM(sales) AS total_sales 
FROM 
	clean_weekly_sales
GROUP BY region, month_number
ORDER BY region;
```

**Answer:**

*I'm only presenting the results of Africa for each month.*

|region|month_number|total_sales|
|------|------------|-----------|
|AFRICA|3|567767480|
|AFRICA|4|1911783504|
|AFRICA|5|1647244738|
|AFRICA|6|1767559760|
|AFRICA|7|1960219710|
|AFRICA|8|1809596890|
|AFRICA|9|276320987|

#### 5. What is the total count of transactions for each platform?

```mysql
SELECT 
	platform, 
    SUM(transactions) AS total_transactions 
FROM 
	clean_weekly_sales 
GROUP BY 
	platform; 
```

**Answer:**
|platform|total_transactions|
|--------|------------------|
|Retail|1081934227|
|Shopify|5925169|

#### 6. What is the percentage of sales for Retail vs Shopify for each month?

```mysql
SELECT 
	calendar_year, 
    month_number, 
    ROUND(((SUM(CASE WHEN platform = "Retail" THEN sales ELSE 0 END)) * 100 /SUM(sales)), 2) AS retail_percentage, 
	ROUND(((SUM(CASE WHEN platform = "Shopify" THEN sales ELSE 0 END)) * 100 /SUM(sales)), 2) AS shopify_percentage
FROM 
	clean_weekly_sales
GROUP BY 
	calendar_year, month_number
ORDER BY 
	calendar_year, month_number; 
```

**Answer:**

*I'm only posting the results of each month in 2018.*

|calendar_year|month_number|retail_percentage|shopify_percentage|
|-------------|------------|-----------------|---------|
|2018|3|97.92|2.08|
|2018|4|97.93|2.07|
|2018|5|97.73|2.27|
|2018|6|97.76|2.24|
|2018|7|97.75|2.25|
|2018|8|97.71|2.29|
|2018|9|97.68|2.32|

#### 7. What is the percentage of sales by demographic for each year in the dataset?

```mysql
SELECT 
	calendar_year, 
    SUM(sales) AS total_sales, 
    ROUND((SUM(IF(demographic = 'Couples', sales, 0))* 100/ SUM(sales)), 2) AS couples_percent, 
    ROUND((SUM(IF(demographic = 'Families', sales, 0))* 100/ SUM(sales)), 2) AS families_percent, 
    ROUND((SUM(IF(demographic IS NULL, sales, 0))*100/ SUM(sales)), 2) AS no_category_percent
FROM 
	clean_weekly_sales 
GROUP BY calendar_year
ORDER BY calendar_year; 
```

**Answer:** 
|calendar_year|total_sales|couples_percent|families_percent|no_category_percent|
|-------------|-----------|---------------|----------------|-------------------|
|2018|12897380827|26.38|31.99|41.63|
|2019|13746032500|27.28|32.47|40.25|
|2020|14100220900|28.72|32.73|38.55|

#### 8. Which `age_band` and `demographic` values contribute the most to Retail sales? 

```mysql
SELECT 
    demographic,
    age_band,
    SUM(sales) AS total_sales,
    RANK() OVER(ORDER BY SUM(sales) DESC) AS sales_overall_rank
FROM 
	clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY demographic, age_band
ORDER BY sales_overall_rank; 
```

**Answer**
|demographic|age_band|total_sales|sales_overall_rank|
|-----------|--------|-----------|------------------|
|NULL|NULL|16067285533|1|
|Families|Retirees|6634686916|2|
|Couples|Retirees|6370580014|3|
|Families|Middle Aged|4354091554|4|
|Couples|Young Adults|2602922797|5|
|Couples|Middle Aged|1854160330|6|
|Families|Young Adults|1770889293|7|

- The highest retail sales are contributed by unknown `age_band` and `demographic`. After that, the retired family contributed the second most to the sales.

#### 9. Can we use the `avg_transaction` column to find the average transaction size for each year for Retail vs Shopify? If not, how would you calculate it instead? 

```mysql
SELECT 
	calendar_year, 
    platform, 
    ROUND((SUM(sales)/SUM(transactions)), 2) AS avg_transaction_group,
    ROUND(AVG(avg_transaction), 2) AS avg_transaction_row
FROM 
	clean_weekly_sales 
GROUP BY 	
	calendar_year, platform
ORDER BY 
	calendar_year, platform; 
```

**Answer:**
|calendar_year|plaform|avg_transaction_group|avg_transaction_row|
|-------------|-------|---------------------|----------|
|2018|Retail|36.56|42.91|
|2018|Shopify|192.48|188.28|
|2019|Retail|36.83|41.97|
|2019|Shopify|183.36|177.56|
|2020|Retail|36.56|40.64|
|2020|Shopify|179.03|174.87|

- The `avg_transaction_row` used `avg_transaction` to find the average, which is the average of each row's average transaction.
- And, the `avg_transaction_group` is the average of all the transactions for each platform per year. This value should be used to answer this question.

### C. Before And After Analysis 

The `week_date` value of `2020-06-15` is used as the start of the period **after** the change and the previous `week_date` values would be `before`. 

To find the `week_number` of `2020-06-15`, the following query is used, and the result is **25**. 

```mysql
SELECT 
	week_date, 
    week_number
FROM 
	clean_weekly_sales 
WHERE	 
	week_date = '2020-06-15'; 
```

#### 1. What is the total sales for the 4 weeks before and after `2020-06-15`? What is the growth or reduction rate in actual values and percentage of sales? 

```mysql
WITH package_sales AS (
	SELECT
		week_date, 
        week_number, 
		SUM(sales) AS total_sales 
	FROM 
		clean_weekly_sales 
	WHERE 
		(week_number BETWEEN 21 AND 28) AND
		(calendar_year = '2020')
	GROUP BY 
		week_date, week_number
), 

before_after AS (
	SELECT 
		SUM(IF(week_number BETWEEN 21 AND 24, total_sales, 0)) AS before_change, 
        SUM(IF(week_number BETWEEN 25 AND 28, total_sales, 0)) AS after_change
	FROM 
		package_sales
)

SELECT 
	before_change, 
    after_change, 
    after_change - before_change AS sales_diff, 
    ROUND(100*(after_change - before_change)/before_change, 2) AS change_percentage
FROM 
	before_after; 
```

**Answer:**
|before_change|after_change|sales_diff|Change_percentage|
|-------------|------------|----------|-----------------|
|2345878357|2318994169|-26884188|-1.15|

#### 2. What about the entire 12 weeks before and after? 

```mysql
WITH change_sales AS (
	SELECT 
		week_date, 
        week_number, 
        SUM(sales) AS total_sales 
	FROM 
		clean_weekly_sales 
	WHERE 
		(week_number BETWEEN 13 AND 36) AND 
		(calendar_year = '2020')
	GROUP BY 	
		week_date, week_number
),

before_and_after AS (
	SELECT 
		SUM(IF(week_number BETWEEN 13 AND 24, total_sales, 0)) AS before_sales, 
        SUM(IF(week_number BETWEEN 25 AND 36, total_sales, 0)) AS after_sales
	FROM 
		change_sales
)

SELECT 
	before_sales, 
    after_sales, 
    after_sales - before_sales AS sales_diff, 
    ROUND(100*(after_sales - before_sales)/before_sales, 2) AS change_percentage
FROM 
	before_and_after; 
```

**Answer:**
|before_sales|after_sales|sales_diff|change_percentage|
|------------|-----------|----------|-----------------|
|7126273147|6973947753|-152325394|-2.14|

#### 3. How do the sales metrics for these 2 periods before and after compare with the previous years in 2018 and 2019? 

```mysql
WITH sales_package AS (
	SELECT 
		week_date, 
        week_number, 
        calendar_year,
        SUM(sales) AS total_sales 
	FROM 
		clean_weekly_sales 
	WHERE week_number BETWEEN 21 and 28 
    GROUP BY week_date
), 

before_and_after AS (
	SELECT 
		calendar_year, 
        SUM(IF(week_number BETWEEN 21 AND 24, total_sales, 0)) AS WK21_to_WK24_sales,
        SUM(IF(week_number BETWEEN 25 AND 28, total_sales, 0)) AS WK25_to_WK28_sales
	FROM 
		sales_package
	GROUP BY calendar_year
)

SELECT
	*, 
    WK25_to_WK28_sales - WK21_to_WK24_sales AS sales_diff, 
    ROUND(100*(WK25_to_WK28_sales - WK21_to_WK24_sales)/WK21_to_WK24_sales, 2) AS change_percentage
FROM 
	before_and_after
ORDER BY calendar_year; 
```

**Answer:**
|calendar_year|WK21_to_WK24_sales|WK25_to_WK28_sales|sales_diff|change_percentage|
|-------------|------------------|------------------|----------|-----------------|
|2018|2125140809|2129242914|4102105|0.19|
|2019|2249989796|2252326390|2336594|0.10|
|2020|2345878357|2318994169|-26884188|-1.15|


### D. Bonus Question
Which areas of the business have the highest negative impact on sales metrics performance in 2020 for the 12-week before and after period? 
- region
- platform
- age_band
- demographic
- customer_type
Do you have any further recommendations for Danny's team at Data Mart or any interesting insights based on this analysis?

#### By region: 
```mysql
WITH sales_package AS (
	SELECT 
		week_date, 
        week_number, 
        region, 
        SUM(sales) AS total_sales 
	FROM 
		clean_weekly_sales 
	WHERE 
		(week_number BETWEEN 13 AND 36) AND 
		(calendar_year = '2020')
	GROUP BY 	
		week_date, week_number, region 
), 

before_after AS (
	SELECT 
		region, 
        SUM(IF(week_number BETWEEN 13 AND 24, total_sales, 0)) AS before_sales, 
        SUM(IF(week_number BETWEEN 25 AND 36, total_sales, 0)) AS after_sales
	FROM 
		sales_package
	GROUP BY 
		region
)

SELECT 
	*, 
    after_sales - before_sales AS sales_diff, 
    ROUND(100*(after_sales - before_sales)/before_sales, 2) AS change_percentage
FROM 
	before_after; 
```

**Answer:**
|region|before_sales|after_sales|sales_diff|change_percentage|
|------|------------|-----------|----------|-----------------|
|ASIA  |1637244466  |1583807621|-53436845|-3.26|
|USA|677013558|666198715|-10814843|-1.60|
|EUROPE|108886567|114038959|5152392|4.73|
|AFRICA|17095377105|1700390294|-9146811|-0.54|
|CANADA|426438454|418264441|-8174013|-1.92|
|OCEANIA|2354116790|2282795690|-71321100|-3.03|
|SOUTH AMERICA|213036207|208452033|-4584174|-2.15|








