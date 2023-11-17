# 🍕Case Study #2: Pizza Runner

<p align="center">
<img width="400px"  src="https://8weeksqlchallenge.com/images/case-study-designs/2.png" />
</p>

Please view the detailed case study info [here](https://8weeksqlchallenge.com/case-study-2/). 


## 📚Table of Contents
- [Business Task](#business-task)
- [Data Cleaning](#data-cleaning)
- [Questions and Solutions](#questions-and-solutions)
- [A. Pizza Metrics](#a-pizza-metrics)
- [B. Runner and Customer Experience](#b-runner-and-customer-experience)
- [C. Ingredient Optimisation](#c-ingredient-optimisation)
- [D. Pricing and Ratings](#d-pricing-and-ratings)
- [E. Bonus Questions](#e-bonus-questions)


***


## Business Task
Danny requires assistance to clean the data and apply some basic calculations so he can better direct his runners and optimise Pizza Runner’s operations.

Danny has provided an entity relationship diagram of his database as below: 
<p align="center">
<img width="605" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/cb6e6cd5-9f2f-4e8d-91b2-4ebf85d1ac94">
</p>


***


## Data Cleaning 

### Table: customer_orders

<p align="center">
<img width="400" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/558b0be8-6ceb-48fd-a8c7-ad70f3c0087b">
</p>

**Cleaning Process:**
- From `customer_orders` table as seen above, all the blank spaces and "null" strings in `exclusions` and `extras`columns are replaced as `NULL` value for data consistency in temporary table called `temp_customer_orders`. 

```mysql
DROP TEMPORARY TABLE IF EXISTS temp_customer_orders; 
CREATE TEMPORARY TABLE temp_customer_orders (
	SELECT 
		order_id, 
    customer_id, 
    pizza_id, 
    CASE
			WHEN exclusions = '' THEN NULL
			WHEN exclusions LIKE '%null%' THEN NULL
      ELSE exclusions
    END AS exclusions, 
    CASE
			WHEN extras = '' THEN NULL
			WHEN extras LIKE '%null%' THEN NULL
      ELSE extras
    END AS extras, 
        order_time
  FROM customer_orders
);
```

### Table: runner_orders

<p align="center">
<img width="500" alt="image" src="https://github.com/NoeHuangx/8-Week-SQL-Challenge/assets/148400128/9c8fbadd-e00d-432b-963f-3ce1212965cd">
</p>

**Cleaning Process:**
- From `runner_orders` table as seen above, all the "null" strings are replaced as `NULL` value across all columns in temporary table `temp_runner_orders`.
- From `distance` and `duration` columns, the texts are removed from the stringed numerical values, and the columns' names are updated to `distance_in_km` and `duration_in_minute`.
- Then, the data types of `pickup_time`, `distance_in_km` and `duration_in_minute` are modified to `TIMESTAMP`,`FLOAT` and `INT`. 

```mysql
DROP TEMPORARY TABLE IF EXISTS temp_runner_orders; 
CREATE TEMPORARY TABLE temp_runner_orders (
	SELECT 
		order_id, 
    runner_id, 
    CASE
			WHEN pickup_time LIKE '%null%' THEN NULL
      ELSE pickup_time
    END AS pickup_time, 
		CASE 
			WHEN distance LIKE '%null%' THEN NULL
      ELSE  TRIM('km' FROM distance)
    END AS distance_in_km, 
    CASE 
			WHEN duration LIKE '%null%' OR NULL THEN NULL
      WHEN duration LIKE '%minutes%' THEN TRIM('minutes' FROM duration)
      WHEN duration LIKE '%minute%' THEN TRIM('minute' FROM duration)
      WHEN duration LIKE '%mins%' THEN TRIM('mins' FROM duration)
      ELSE duration
    END AS duration_in_minute, 
    CASE
			WHEN cancellation = '' THEN NULL
			WHEN cancellation IS NULL THEN NULL
      WHEN cancellation LIKE '%null%' THEN NULL
      ELSE cancellation
    END AS cancellation
FROM runner_orders
);

ALTER TABLE temp_runner_orders 
MODIFY COLUMN pickup_time TIMESTAMP,
MODIFY COLUMN distance_in_km FLOAT, 
MODIFY COLUMN duration_in_minute INT; 
```


***


## Questions and Solutions
### A. Pizza Metrics 
#### 1. How many pizzas were ordered?

```mysql
SELECT 
    COUNT(*) AS pizza_count
FROM
    temp_customer_orders;
```

**Answer:** 
|pizza_count|
|-----------|
|14         |


#### 2. How many unique customer orders were made?

```mysql
SELECT 
    COUNT(DISTINCT order_id) AS unique_pizza_count
FROM
    temp_customer_orders; 
```

**Answer:**
|unique_pizza_count|
|------------------|
|10                |


#### 3. How many successful orders were delivered by each runner?

```mysql
SELECT 
    runner_id,
    COUNT(*) AS orders_completed
FROM
    temp_runner_orders
WHERE
    pickup_time IS NOT NULL
GROUP BY runner_id
ORDER BY runner_id;
```

**Answer:**
|runner_id|orders_completed|
|---------|----------------|
|1        |4               |
|2        |3               |
|3        |1               |


#### 4. How many of each type of pizza was delivered?

```mysql
SELECT 
    p.pizza_name, COUNT(c.pizza_id) AS count
FROM
    temp_customer_orders c
        JOIN
    pizza_names p ON p.pizza_id = c.pizza_id
        JOIN
    temp_runner_orders r ON r.order_id = c.order_id
WHERE
    r.pickup_time IS NOT NULL
GROUP BY p.pizza_name; 
```

**Answer:** 
|pizza_name|count|
|----------|-----|
|Meatlovers|9    |
|Vegetarian|3    |


#### 5. How many Vegetarian and Meatlovers were ordered by each customer?

```mysql
SELECT 
    c.customer_id, p.pizza_name, COUNT(*) AS order_count
FROM
    temp_customer_orders c
        JOIN
    pizza_names p ON p.pizza_id = c.pizza_id
GROUP BY c.customer_id , p.pizza_name
ORDER BY c.customer_id;
```

**Answer:**
|customer_id|pizza_name|order_count|
|-----------|----------|-----------|
|101        |Meatlovers|2          |
|101        |Vegetarian|1          |
|102        |Meatlovers|2          |
|102        |Vegetarian|1          |
|103        |Meatlovers|3          |
|103        |Vegetarian|1          |
|104        |Meatlovers|3          |
|105        |Vegetarian|1          |


#### 6. What was the maximum number of pizzas delivered in a single order?

```mysql
-- Q6: What was the maximum number of pizzas delivered in a single order?
SELECT 
    order_id, COUNT(*) AS count
FROM
    temp_customer_orders
GROUP BY order_id
ORDER BY pizza_count DESC
LIMIT 1;
```

**Answer:**
|order_id|count|
|--------|-----|
|4       |3    |


#### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```mysql
SELECT 
    c.customer_id,
    SUM(CASE
        WHEN CHAR_LENGTH(c.exclusions) > 0 THEN 1
        WHEN CHAR_LENGTH(c.extras) > 0 THEN 1
        ELSE 0
    END) AS pizza_count_with_change,
    SUM(IF(c.exclusions IS NULL AND c.extras IS NULL, 1,0)) AS pizza_count_with_no_change
FROM
    temp_customer_orders c
        JOIN
    temp_runner_orders r ON c.order_id = r.order_id
WHERE
    r.pickup_time IS NOT NULL
GROUP BY customer_id
ORDER BY customer_id; 
```

**Answer:**
|customer_id|pizza_count_with_change|pizza_count_with_no_change|
|-----------|-----------------------|--------------------------|
|101        |0                      |2                         |
|102        |0                      |3                         |
|103        |3                      |0                         |
|104        |2                      |1                         |
|105        |1                      |0                         |


#### 8. How many pizzas were delivered that had both exclusions and extras?

```mysql
SELECT 
    SUM(IF(CHAR_LENGTH(c.exclusions) > 0
            AND CHAR_LENGTH(c.extras) > 0,
        1,
        0)) AS with_exclusions_and_extras
FROM
    temp_customer_orders c
        JOIN
    temp_runner_orders r ON c.order_id = r.order_id
WHERE
    r.pickup_time IS NOT NULL; 
```

**Answer:**
|with_exclusions_and_extras|
|--------------------------|
|1                         |


#### 9. What was the total volume of pizzas ordered for each hour of the day?

```mysql
SELECT 
    HOUR(order_time) AS hour_of_the_day, COUNT(*) AS total_pizza
FROM
    temp_customer_orders
GROUP BY hour_of_the_day
ORDER BY hour_of_the_day;
```

**Answer:**
|hour_of_the_day|total_pizza|
|---------------|-----------|
|11             |1          |
|13             |3          |
|18             |3          |
|19             |1          |
|21             |3          |
|23             |3          |


#### 10. What was the volume of orders for each day of the week?

```mysql
SELECT 
    DAYNAME(order_time) AS weekday, COUNT(*) AS total_pizza
FROM
    temp_customer_orders
GROUP BY weekday
ORDER BY total_pizza DESC;
```

**Answer:**
| weekday |total_pizza|
|---------|-----------|
|Wednesday|5          |
|Saturday |5          |
|Thursday |3          |
|Friday   |1          |


### B. Runner and Customer Experience
#### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

```mysql
SELECT 
    WEEK(registration_date) AS week_num,
    COUNT(*) AS count
FROM
    runners
GROUP BY week_num;
```

**Answer:**
|week_num|count|
|--------|-----|
|0       |1    |
|1       |2    |
|2       |1    |


#### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

```mysql
SELECT 
    r.runner_id,
    MINUTE(AVG(TIMEDIFF(r.pickup_time, c.order_time))) AS avg_pickup_duration
FROM
    temp_customer_orders c
        JOIN
    temp_runner_orders r ON c.order_id = r.order_id
WHERE
    r.pickup_time IS NOT NULL
GROUP BY r.runner_id;
```

**Answer:** 
|runner_id|avg_pickup_duration|
|---------|-------------------|
|1        |15                 |
|2        |23                 |
|3        |10                 |


#### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

```mysql
WITH duration_cte AS (
SELECT 
    COUNT(c.pizza_id) AS num_of_pizza,
    MINUTE(TIMEDIFF(r.pickup_time, c.order_time)) AS time_before_pickup
FROM
    temp_customer_orders c
        JOIN
    temp_runner_orders r ON c.order_id = r.order_id
WHERE
    r.pickup_time IS NOT NULL
GROUP BY c.order_id)

SELECT 
	num_of_pizza, 
    AVG(time_before_pickup) AS avg_prep_time 
FROM duration_cte
GROUP BY num_of_pizza;
```

**Answer:**
|num_of_pizza|avg_prep_time|
|------------|-------------|
|1           |12.0000      |
|2           |18.0000      |
|3           |29.0000      |

It seems like the larger the number of pizzas order, the longer the prep time is.

**Note:** I have disabled the `ONLY_FULL_GROUP_BY` mode for the session running this query with the following code: 
```mysql
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```


#### 4. What was the average distance travelled for each customer?

```mysql
SELECT 
    c.customer_id,
    CAST(AVG(r.distance_in_km) AS DECIMAL (4 , 2 )) AS avg_distance_km
FROM
    temp_customer_orders c
        JOIN
    temp_runner_orders r ON c.order_id = r.order_id
WHERE
    r.pickup_time IS NOT NULL
GROUP BY c.customer_id;
```

**Answer:** 
|customer_id|avg_distance_km|
|-----------|---------------|
|101        |20.00          |
|102        |16.73          |
|103        |23.40          |
|104        |10.00          |
|105        |25.00          |


#### 5. What was the difference between the longest and shortest delivery times for all orders?

```mysql
SELECT 
    MAX(duration_in_minute) AS max_duration,
    MIN(duration_in_minute) AS min_duration,
    MAX(duration_in_minute) - MIN(duration_in_minute) AS time_difference
FROM
    temp_runner_orders; 
```

**Answer:** 
|max_duration|min_duration|time_difference|
|------------|------------|---------------|
|40          |10          |30             |


#### 6. What was the average speed for each runner for each delivery and do you notice any trend for these values?

```mysql
SELECT 
    runner_id,
    pickup_time,
    distance_in_km,
    duration_in_minute,
    ROUND((distance_in_km / (duration_in_minute / 60)),
            2) AS speed
FROM
    temp_runner_orders
WHERE
    pickup_time IS NOT NULL
ORDER BY runner_id;
```

**Answer:**

speed = distance in km / duration in hour

|runner_id|     pickup_time   |distance_in_km|duration_in_minute|speed|
|---------|-------------------|--------------|------------------|-----|
|1        |2020-01-01 18:15:34|20            |32                |37.5 |
|1        |2020-01-01 19:10:54|20            |27                |44.44|
|1        |2020-01-03 00:12:37|13.4          |20                |40.2 |
|1        |2020-01-11 18:50:20|10            |10                |60   |
|2        |2020-01-04 13:53:03|23.4          |40                |35.1 |
|2        |2020-01-08 21:30:45|25            |25                |60   |
|2        |2020-01-10 00:15:02|23.4          |15                |93.6 |
|3        |2020-01-08 21:10:57|10            |15                |40   |

I do not notice any trend. 


#### 7. What is the successful delivery percentage for each runner?

```mysql
WITH orders_cte AS (
SELECT 
	runner_id, 
    SUM(IF(pickup_time IS NOT NULL, 1, 0)) AS success_orders, 
    SUM(IF(pickup_time IS NULL, 1, 0))AS fail_orders,
    COUNT(*) AS total_orders
FROM temp_runner_orders
GROUP BY runner_id) 

SELECT runner_id, 
	(success_orders/total_orders)*100 AS success_percentage
FROM orders_cte
ORDER BY runner_id;
```

**Answer:**
|runner_id|success_percentage|
|---------|------------------|
|1        |100.0000          |
|2        |75.0000           |
|3        |50.0000           |


### C. Ingredient Optimisation

The following temporary tables are created from `pizza_recipes` and `temp_customer_orders` to answer the questions from section C thru E. 

**temp_pizza_recipes**
```mysql
DROP TEMPORARY TABLE IF EXISTS temp_pizza_recipes; 
CREATE TEMPORARY TABLE temp_pizza_recipes (
	SELECT pr.pizza_id, 
		TRIM(j.toppings) AS topping_id 
	FROM pizza_recipes pr
    JOIN JSON_TABLE(TRIM(REPLACE(JSON_ARRAY(pr.toppings), ',', '","')), 
		'$[*]' COLUMNS (toppings VARCHAR(50) PATH '$')) AS j
);

ALTER TABLE temp_pizza_recipes 
MODIFY COLUMN topping_id INT;
```

**temp_customer_orders_split**
```mysql
DROP TEMPORARY TABLE IF EXISTS temp_customer_orders_split; 
CREATE TEMPORARY TABLE temp_customer_orders_split (
	SELECT
		c.row_num,
		c.order_id, 
        c.customer_id, 
        c.pizza_id, 
        TRIM(j1.exclusions) AS exclusions, 
        TRIM(j2.extras) AS extras,
        c.order_TIME
	FROM
		(SELECT *, 
			ROW_NUMBER() OVER() AS row_num
		FROM temp_customer_orders) AS c 
    JOIN 
		JSON_TABLE(TRIM(REPLACE(JSON_ARRAY(c.exclusions), ',', '","')), 
        '$[*]' COLUMNS(exclusions VARCHAR(11) PATH '$')
    ) AS j1
    JOIN 
		JSON_TABLE(TRIM(REPLACE(JSON_ARRAY(c.extras), ',', '","')), 
        '$[*]' COLUMNS(extras VARCHAR(11) PATH '$')
    ) AS j2
);

ALTER TABLE temp_customer_orders_split
MODIFY COLUMN exclusions INT,
MODIFY COLUMN extras INT;
```

#### 1. What are the standard ingredients for each pizza? 

```mysql
SELECT 
    pn.pizza_name,
    GROUP_CONCAT(pt.topping_name) AS standard_ingredients
FROM
    temp_pizza_recipes pr
        JOIN
    pizza_names pn ON pn.pizza_id = pr.pizza_id
        JOIN
    pizza_toppings pt ON pt.topping_id = pr.topping_id
GROUP BY pn.pizza_name;
```

**Answer:**
|pizza_name |standard_ingredients                                          |
|-----------|--------------------------------------------------------------|
|Meatlovers |Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami|
|Vegetarian |Cheese,Mushrooms,Onions,Peppers,Tomatoes,Tomato Sauce         |


#### 2. What was the most commonly added extra?

```mysql
SELECT 
    pt.topping_name AS common_extra_toppings,
    COUNT(*) AS count
FROM
    temp_customer_orders_split c
        JOIN
    pizza_toppings pt ON c.extras = pt.topping_id
GROUP BY pt.topping_name;
```

 **Answer:**
 |common_extra_toppings|count|
 |---------------------|-----|
 |Bacon                |5    |
 |Chicken              |1    |
 |Cheese               |2    |


 #### 3. What was the most common exclusion?

 ```mysql
SELECT 
    pt.topping_name AS common_exclusion_toppings,
    COUNT(*) AS count
FROM
    temp_customer_orders_split c
        JOIN
    pizza_toppings pt ON c.exclusions = pt.topping_id
GROUP BY pt.topping_name; 
```

**Answer:**
|common_exclusion_toppings|count|
|-------------------------|-----|
|Cheese                   |5    |
|BBQ Sauce                |2    |
|Mushrooms                |2    |


#### 4. Generate an order item for each record in the customers_orders table in the format of one of the following:
  - Meat Lovers
  - Meat Lovers - Exclude Beef
  - Meat Lovers - Extra Bacon
  - Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

```mysql
SELECT 
    c.order_id,
    c.customer_id,
    c.pizza_id,
    c.order_time,
    GROUP_CONCAT(DISTINCT c.exclusions) AS exclusions,
    GROUP_CONCAT(DISTINCT c.extras) AS extras,
    CASE
        WHEN
            c.exclusions IS NULL
                AND c.extras IS NULL
        THEN
            pn.pizza_name
        WHEN
            c.exclusions IS NOT NULL
                AND c.extras IS NULL
        THEN
            CONCAT(pn.pizza_name,
                    ' - Exclude ',
                    pt.topping_name)
        WHEN
            c.exclusions IS NULL
                AND c.extras IS NOT NULL
        THEN
            CONCAT(pn.pizza_name,
                    ' - Extra ',
                    e.topping_name)
        ELSE CONCAT(pn.pizza_name,
                ' - Exclude ',
                GROUP_CONCAT(DISTINCT pt.topping_name SEPARATOR ', '),
                '  - Extra ',
                GROUP_CONCAT(DISTINCT e.topping_name SEPARATOR ', '))
    END AS pizza_info
FROM
    temp_customer_orders_split c
        JOIN
    pizza_names pn ON pn.pizza_id = c.pizza_id
        LEFT JOIN
    pizza_toppings pt ON pt.topping_id = c.exclusions
        LEFT JOIN
    pizza_toppings e ON e.topping_id = c.extras
GROUP BY c.row_num; 
```

**Answer:**
|order_id|customer_id|pizza_id|  order_time       |exclusions|extras|pizza_info                        |
|--------|-----------|--------|-------------------|----------|------|----------------------------------|
|1       |101        |1       |2020-01-01 18:05:02|NULL      |NULL  |Meatlovers                        |
|2       |101        |1       |2020-01-01 19:00:52|NULL      |NULL  |Meatlovers                        |
|3       |102        |1       |2020-01-02 23:51:23|NULL      |NULL  |Meatlovers                        |
|3       |102        |2       |2020-01-02 23:51:23|NULL      |NULL  |Vegetarian                        |
|4       |103        |1       |2020-01-04 13:23:46|4         |NULL  |Meatlovers -Exclude Cheese        |
|4       |103        |1       |2020-01-04 13:23:46|4         |NULL  |Meatlovers -Exclude Cheese        |
|4       |103        |2       |2020-01-04 13:23:46|4         |NULL  |Vegetarian -Exclude Cheese        |
|5       |104        |1       |2020-01-08 21:00:29|NULL      |1     |Meatlovers -Extra Bacon           |
|6       |101        |2       |2020-01-08 21:03:13|NULL      |NULL  |Vegetarian                        |
|7       |105        |2       |2020-01-08 21:20:29|NULL      |1     |Vegetarian -Extra Bacon           |
|8       |102        |1       |2020-01-09 23:54:33|NULL      |NULL  |Meatlovers                        |
|9       |103        |1       |2020-01-10 11:22:59|4         |1,5   |Meatlovers -Exclude Cheese -Extra Bacon, Chicken|
|10      |104        |1       |2020-01-11 18:34:49|NULL      |NULL  |Meatlovers                        |
|10      |104        |1       |2020-01-11 18:34:49|2,6       |1,4   |Meatlovers -Exclude BBQ Sauce, Mushrooms -Extra Bacon, Cheese|


#### 5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
- For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

```mysql
WITH ingridients AS (
	SELECT 
		c.*, 
        pn.pizza_name,
        CASE
			WHEN pr.topping_id IN (c.exclusions) THEN NULL
			WHEN pr.topping_id IN (c.extras) THEN CONCAT('2x ' , pt.topping_name)
            ELSE pt.topping_name
        END AS toppings 
	FROM temp_pizza_recipes pr
    JOIN pizza_names pn ON pn.pizza_id = pr.pizza_id
    JOIN pizza_toppings pt ON pr.topping_id = pt.topping_id
    JOIN temp_customer_orders_split c ON pr.pizza_id = c.pizza_id 
)

SELECT
    order_id,
    pizza_id,
    GROUP_CONCAT(DISTINCT exclusions) as exclusions, 
    GROUP_CONCAT(DISTINCT extras) as extras,
    CONCAT(pizza_name, ': ', GROUP_CONCAT(DISTINCT toppings SEPARATOR ', ')) AS pizza_info 
FROM ingridients
GROUP BY row_num;
```
  
**Answer:**
|order_id|pizza_id|exclusions|extras|pizza_info|
|--------|--------|----------|------|----------|
|1       |1       |NULL      |NULL  |Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|2       |1       |NULL      |NULL  |Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|3       |1       |NULL      |NULL  |Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|3       |2       |NULL      |NULL  |Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes|
|4       |1       |4        |NULL   |Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami|
|4       |1       |4        |NULL   |Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami|
|4       |2       |4        |NULL   |Vegetarian: Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes|
|5       |1       |NULL     |1      |Meatlovers: 2x Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|6       |2       |NULL     |NULL  |Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes|
|7       |2       |NULL     |1   |Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes|
|8       |1       |NULL      |NULL  |Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|9       |1       |4     |1,5  |Meatlovers: 2x Bacon, 2x Chicken, Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|10      |1       |NULL      |NULL  |Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|
|10       |1       |2,6      |1,4  |Meatlovers: 2x Bacon, 2x Cheese, Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami|


#### 6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

```mysql
WITH ingredients AS (
	SELECT 
		pt.topping_name, 
        CASE
			      WHEN pr.topping_id IN (c.extras) THEN 2
            WHEN pr.topping_id IN (c.exclusions) THEN 0 
            ELSE 1
        END AS qty
FROM temp_pizza_recipes pr 
JOIN temp_customer_orders_split c ON c.pizza_id = pr.pizza_id 
JOIN pizza_toppings pt ON pt.topping_id = pr.topping_id
)

SELECT 
	topping_name, 
    SUM(qty) AS total_qty
FROM ingredients 
GROUP BY topping_name
ORDER BY total_qty DESC;
```

**Answer:**
|topping_name|total_qty|
|------------|---------|
|Bacon       | 18      |
|Mushrooms   | 16      |
|Chicken     | 15      |
|Cheese      | 15      |
|Salami      | 14      |
|Pepperoni   | 14      |
|Beef        | 14      |
|BBQ Sauce   | 12      |
|Tomato Sauce| 4       |
|Tomatoes    | 4       |
|Peppers     | 4       |
|Onions      | 4       |


### D. Pricing and Ratings
#### 1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?

```mysql
SELECT 
	pn.pizza_name, 
    COUNT(*) AS qty,
    SUM(CASE
		WHEN pn.pizza_id = 1 THEN 12
        ELSE 10
    END) AS total
FROM temp_customer_orders c 
JOIN pizza_names pn ON pn.pizza_id = c.pizza_id
JOIN temp_runner_orders r ON r.order_id = c.order_id
WHERE r.cancellation IS NULL
GROUP BY pn.pizza_name WITH ROLLUP;
```

**Answer:** 
|pizza_name|qty|total|
|----------|---|-----|
|Meatlovers|9  |108  |
|Vegetarian|3  |30   |
|NULL      |12 |138  |


#### 2. What if there was an additional $1 charge for any pizza extras?
- Add cheese is $1 extra

```mysql
WITH pizza_base AS (
SELECT 
	c.order_id,
    CASE
		WHEN c.pizza_id = 1 THEN 12
        ELSE 10
    END AS pizza_cost, 
    GROUP_CONCAT(DISTINCT c.extras) AS extras
FROM temp_customer_orders_split c 
JOIN temp_runner_orders r ON r.order_id = c.order_id
WHERE r.cancellation IS NULL
GROUP BY c.row_num
)

SELECT 
	SUM(CASE
		WHEN extras IS NULL THEN pizza_cost
        WHEN CHAR_LENGTH(extras) = 1 THEN pizza_cost + 1
        ELSE pizza_cost + 2
    END) AS total_revenue 
FROM pizza_base;
```

**Answer:** 
|total_revenue|
|-------------|
|142          |


#### 3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.

```mysql
DROP TABLE IF EXISTS rating; 
CREATE TABLE rating (
	order_id INT NOT NULL, 
    ratings INT NOT NULL,
    reviews VARCHAR(255) NULL
); 

INSERT INTO rating(order_id, ratings, reviews)
VALUES 
	(1, 4, 'the pizza tastes good, but service is okay'),
    (2, 5, NULL), 
    (10, 5, 'the pizza were awesome and service is super fast'), 
    (4, 1, 'it takes forever for the pizza to get delivered to me and pizza arrives cold'), 
    (3, 3, 'it is alright, nothing special'), 
    (5, 5, 'craving satisfy without leaving the house'), 
    (8, 5, 'super fast and pizza tastes good. would recommend'), 
    (7, 2, NULL);
```

**Answer:**
|order_id|ratings|reviews                  |
|--------|-------|-------------------------|
|1       |4      |the pizza tastes good, but service is okay|
|2       |5      |NULL                      |
|10      |5      |the pizza were awesome and service is super fast|
|4       |1      |it takes forever for the pizza to get delivered to me and pizza arrives cold|
|3       |3      |it is alright, nothing special|
|5       |5      |craving satisfy without leaving the house|
|8       |5      |super fast and pizza tastes good. would recommend|
|7       |2      |NULL|


#### 4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
- customer_id
- order_id
- runner_id
- rating
- order_time
- pickup_time
- Time between order and pickup
- Delivery duration
- Average speed
- Total number of pizzas

```mysql
SELECT 
	c.customer_id, 
    c.order_id, 
    tr.runner_id, 
    r.ratings,
    c.order_time, 
    tr.pickup_time, 
    MINUTE(TIMEDIFF(tr.pickup_time, c.order_time)) AS 'time_between_order_and_pickup',
    tr.duration_in_minute AS delivery_duration, 
    ROUND((tr.distance_in_km)/(tr.duration_in_minute/60), 2) AS average_speed, 
    COUNT(pizza_id) AS total_pizza
FROM temp_customer_orders c 
JOIN temp_runner_orders tr ON tr.order_id = c.order_id 
JOIN rating r ON r.order_id = c.order_id 
WHERE tr.cancellation IS NULL 
GROUP BY c.order_id; 
```

**Answer:**
|customer_id|order_id|runner_id|ratings|order_time|pickup_time |time_between_order_and_pickup|delivery_duration|avg_speed|total_pizza|
|-----------|--------|---------|-------|----------|------------|-----------------------------|-----------------|---------|-----------|
|101|1|1|4|2020-01-01 18:05:02|2020-01-01 18:15:34|10|32|37.5|1|
|101|2|1|5|2020-01-01 19:00:52|2020-01-01 19:10:54|10|27|44.44|1|
|102|3|1|3|2020-01-02 23:51:23|2020-01-03 00:12:37|21|20|40.2|2|
|103|4|2|1|2020-01-04 13:23:46|2020-01-04 13:53:03|29|40|35.1|3|
|104|5|3|5|2020-01-08 21:00:29|2020-01-08 21:10:57|10|15|40|1|
|105|7|2|2|2020-01-08 21:20:29|2020-01-08 21:30:45|10|25|60|1|
|102|8|2|5|2020-01-09 23:54:33|2020-01-10 00:15:02|20|15|93.6|1|
|104|10|1|5|2020-01-11 18:34:49|2020-01-11 18:50:20|15|10|60|2|


#### 5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

```mysql
WITH accounting AS (
	SELECT 
		SUM(IF(c.pizza_id = 1, 12, 10)) AS pizza_revenue, 
        distance_in_km,
        ROUND((r.distance_in_km * 0.30),2) AS runner_fees
	FROM temp_customer_orders c 
    JOIN temp_runner_orders r ON r.order_id = c.order_id 
    WHERE r.cancellation IS NULL
)

SELECT 
	(SUM(pizza_revenue) - SUM(runner_fees)) AS total_profit 
FROM accounting;
```

**Answer:**
|total_profit|
|------------|
|132.00      |


### E. Bonus Questions
#### If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?

```mysql
INSERT INTO pizza_names (pizza_id, pizza_name)
VALUES (3, 'Supreme'); 

INSERT INTO pizza_recipes (pizza_id, toppings) 
VALUES (3, (SELECT GROUP_CONCAT(topping_id) FROM pizza_toppings));
```

**Answer:**
|pizza_id|pizza_name|
|--------|----------|
|1|Meatlovers|
|2|Vegetarian|
|3|Supreme|

|pizza_id|toppings|
|--------|--------|
|1|1,2,3,4,6,8,10|
|2|4,6,7,9,11,12|
|3|1,2,3,4,5,6,7,8,9,10,11,12|











