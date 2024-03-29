# 🍊Case Study #8: Fresh Segments

<p align='center'>
<img width='400px' src="https://8weeksqlchallenge.com/images/case-study-designs/8.png">
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-8/).

## 📚Table of Contents
- [Business Task](#business-task)
- [Questions And Solutions](#questions-and-solutions)
- [Data Exploration and Cleansing](#data-exploration-and-cleansing)
- [Interest Analysis](#interest-analysis)
- [Segment Analysis](#segment-analysis)
- [Index Analysis](#index-analysis)

***

## Business Task 
Fresh Segments is a digital marketing agency that helps other businesses analyze trends in online ad click behavior for their unique customer base. 

The task is to analyze the aggregated metrices and provide high levels insights for the given dataset. 

There are 2 datasets in this case study to answer the business questions as follows: 

<p align='center'>
<img width="458" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/6a54994d-51b5-4450-bdd9-515f66076cbf">
</p>


*** 

## Questions and Solutions 
### Data Exploration and Cleansing 
#### 1. Update the `interest_metrics` table by modifying the month_year column to be a date data type with the start of the month. 

```mysql
-- Drop month_year column
ALTER TABLE interest_metrics 
DROP COLUMN month_year;

-- Add new month_year column
ALTER TABLE interest_metrics
ADD COLUMN month_year DATE; 

-- Add data to the month_year column
UPDATE interest_metrics 
SET month_year = CAST(CONCAT(_year, "-", _month, "-01") AS DATE);

-- To check the data type of each column
DESC interest_metrics; 
```

**Answer:** 

<img width="286" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/176f814b-54b7-452c-af0f-1881d7353ef1">

#### 2. What is the count of records in the `interest_metrics` for each `month_year` value sorted in chronological order (earliest to latest) with the null values appearing first? 

```mysql
SELECT
	month_year, 
	COUNT(*) AS count
FROM interest_metrics
GROUP BY month_year
ORDER BY month_year;
```

**Answer:** 

<img width="113" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/ebb78e02-e4ce-4f78-a233-fb6d92764054">


#### 3. What do you think we should do with these null values in the `interest_metrics`? 

```mysql
-- To find the percentage of null values in this dataset
SELECT 
    100 * COUNT(*) / (SELECT 
            COUNT(*)
        FROM
            interest_metrics) AS null_percent
FROM
    interest_metrics
WHERE
    interest_id IS NULL; 
```

**Answer:** 

<img width="88" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/6efdd09c-7277-48d9-be0f-7d2e0fa83d29">

Since the data is meaningless without interest_id, _month, and _year info and the null values percentage is only around 8.35%, I suggest dropping these null values. 

```mysql
-- To delete null values
DELETE FROM interest_metrics 
WHERE interest_id IS NULL;

-- To check if all null values are deleted
SELECT 
    100 * COUNT(*) / (SELECT 
            COUNT(*)
        FROM
            interest_metrics) AS null_percent
FROM
    interest_metrics
WHERE
    interest_id IS NULL; 
```

<img width="83" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/a683deea-e64d-4b50-b376-42ec16b758d9">


#### 4. How many `interest_id` values exist in the `interest_metrics` table but not in the `interest_map` table? What about the other way around? 

```mysql
-- Outer Join two tables 
WITH combine_cte AS (
	SELECT * FROM interest_metrics mt LEFT JOIN interest_map mp ON mt.interest_id = mp.id 
    UNION
    SELECT * FROM interest_metrics mt RIGHT JOIN interest_map mp ON mt.interest_id = mp.id
)

SELECT 
	COUNT(DISTINCT interest_id) AS metrics_id_count, 
    COUNT(DISTINCT id) AS maps_id_count, 
    SUM(IF(interest_id IS NULL, 1, 0)) AS not_in_metrics, 
    SUM(IF(id IS NULL, 1, 0)) AS not_in_maps
FROM 
	combine_cte; 
```

**Answer:** 

<img width="292" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/35a77062-bfab-4112-904f-f9a3e37d1a40">


#### 5. Summarize the `id` values in the `interest_map` by its total record count in this table. 

```mysql
SELECT 
    id, interest_name, COUNT(*) AS record_count
FROM
    interest_map mp
        JOIN
    interest_metrics mt ON mt.interest_id = mp.id
GROUP BY mp.id
ORDER BY record_count DESC; 
```

**Answer:** 

*Only showing the partial result here. There are 1,000 results from this query.* 

<img width="298" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/4bc0088c-c2a9-4735-b3f7-f6e4f279bd76">

#### 6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where `interest_id = 21246` in your joined output and include all columns from `interest_metrics` and all columns from `interest_map` except the `id` column. 

The good old `INNER JOIN` should be used to connect two tables by matching `interest_id` & `id`. This `JOIN` method would allow us to have the data records that are available in both tables. 

```mysql
SELECT 
    mt.*,
    mp.interest_name,
    mp.interest_summary,
    mp.created_at,
    mp.last_modified
FROM
    interest_metrics mt
        JOIN
    interest_map mp ON mt.interest_id = mp.id
WHERE
    mt.interest_id = 21246
        AND mt._month IS NOT NULL;
```

**Answer:** 

<img width="937" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/5fa6b813-c9c0-42e6-8b87-ed3c69889b28">


#### 7. Are there any records in your joined table where the `month_year` value is before the `created_at` value from the `interest_map` table? Do you think these values are valid and why? 

```mysql
-- To check the data counts where the month_year is before the created_at value
SELECT
	COUNT(*) AS count
FROM 
	interest_metrics mt
JOIN interest_map mp ON mt.interest_id = mp.id 
WHERE mt.month_year < mp.created_at; 
```

**Answer:** 

<img width="60" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/c3329b36-10f3-49cd-8944-65912408b172">

Even though there are 188 records where the `month_year` value is before the `created_at` value, we also have to keep in mind how the `month_year` column is created. The `month_year` column was updated to have the data values at the start of each month. So, I would check again whether there are any values of `month_year` before `created_at` and not in the same month and same year as the `created_at`, and there are none in such condition as seen below. 

```mysql
SELECT 
    COUNT(*) AS ctn
FROM
    interest_metrics mt
        JOIN
    interest_map mp ON mt.interest_id = mp.id
WHERE
    mt.month_year < mp.created_at
        AND MONTH(mt.month_year) != MONTH(mp.created_at)
        AND YEAR(mt.month_year) != YEAR(mp.created_at); 
```

**Answer:** 

<img width="56" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/a2aa5650-a587-4286-97e2-3addff802c5d">

### Interest Analysis 
#### 1. Which interests have been present in all `month_year` dates in our dataset? 

```mysql
-- To check the unique number of interests and month_year
SELECT 
    COUNT(DISTINCT interest_id) AS interest_count,
    COUNT(DISTINCT month_year) AS month_year_count
FROM
    interest_metrics; 
```

**Answer:** 

<img width="172" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/73f6e031-f22a-42d5-9990-0a4d0a143206">



```mysql
--To check the interests count that present in all month_year 
WITH count_cte AS (
	SELECT 
		interest_id, 
        COUNT(DISTINCT month_year) AS total_months 
	FROM 
		interest_metrics 
	WHERE month_year IS NOT NULL
	GROUP BY 
		interest_id
)

SELECT 
	total_months,
	COUNT(DISTINCT interest_id) AS interest_count
FROM 
	count_cte 
WHERE total_months = 14; 
```

**Answer:** 

<img width="150" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/2b9ad4bd-a121-4461-9cd5-4c247278e229">

#### 2. Using this same `total_months` measure - calculate the cumulative percentage of all records starting at 14 months - which `total_months` value passes the 90% cumulative percentage value? 

```mysql
WITH month_count_cte AS (
	SELECT
		interest_id, 
        COUNT(DISTINCT month_year) AS total_months
	FROM 
		interest_metrics
	WHERE month_year IS NOT NULL
    GROUP BY interest_id
), 

interest_count AS (
	SELECT
		total_months, 
        COUNT(DISTINCT interest_id) AS interest_count 
	FROM 
		month_count_cte
	GROUP BY total_months
), 

percent_cte AS (
	SELECT *, 
		ROUND(100* SUM(interest_count) OVER(ORDER BY total_months DESC)/(SELECT COUNT(DISTINCT interest_id) FROM interest_metrics), 2) AS cumulative_percent
	FROM 
		interest_count
) 

SELECT 
	*
FROM 
	percent_cte 
WHERE cumulative_percent > 90; 
```

**Answer:** 

<img width="231" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/42f73d26-6293-4b09-90ec-4ae63b2a2f01">


#### 3. If we were to remove all `interest_id` values which are lower than the `total_months` value we found in the previous question - how many total data points would we be removing? 

```mysql
WITH month_cte AS (
	SELECT 
		COUNT(DISTINCT month_year) AS total_months, 
        interest_id
	FROM 
		interest_metrics 
	WHERE month_year IS NOT NULL 
    GROUP BY interest_id
    HAVING total_months <= 6
)

SELECT 
	COUNT(interest_id) AS interest_count 
FROM 
	month_cte; 
```

**Answer:** 

<img width="89" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/fbb8e3b9-079f-427c-82ee-dffb9c355464">

#### 4. Does this decision make sense to remove these data points from a business perspective? Use an example where there are all 14 months present to a removed `interest` example for your arguments - think about what it means to have less months present from a segment perspective. 















