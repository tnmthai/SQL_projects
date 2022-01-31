# Pizza Runner 

## Part A. Pizza Metrics

### Question 1: How many pizzas were ordered?

```sql
SELECT COUNT(*) FROM pizza_runner.customer_orders;
```

### Question 2: How many unique customer orders were made?

```sql
SELECT COUNT(DISTINCT order_id) FROM pizza_runner.customer_orders;
```
### Question 3: How many successful orders were delivered by each runner?
```sql
SELECT runner_id,  COUNT(DISTINCT order_id) AS successful_orders
FROM pizza_runner.runner_orders
WHERE pickup_time NOT IN ('null')
GROUP BY runner_id
ORDER BY successful_orders DESC;

```
### Question 4: How many of each type of pizza was delivered?
```sql
SELECT
  t2.pizza_name,
  COUNT(t1.*) AS delivered_pizza_count
FROM pizza_runner.customer_orders AS t1
INNER JOIN pizza_runner.pizza_names AS t2
  ON t1.order_id = t2.pizza_id
WHERE EXISTS (
  SELECT 1 FROM pizza_runner.runner_orders AS t3
  WHERE t1.order_id = t3.order_id
  AND (
    t3.cancellation IS NULL
    OR t3.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
  )
)
GROUP BY t2.pizza_name
ORDER BY t2.pizza_name;
```
### Question 5: How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT
  order_id,
  SUM(CASE WHEN pizza_id = 1 THEN 1 ELSE 0 END) AS meatlovers,
  SUM(CASE WHEN pizza_id = 2 THEN 2 ELSE 0 END) AS vegetarian
FROM pizza_runner.customer_orders
GROUP BY order_id
ORDER BY order_id;
```
### Question 6: What was the maximum number of pizzas delivered in a single order?

```sql
WITH cte_ranked_orders AS (
  SELECT
    order_id,
    COUNT(*) AS pizza_count,
    RANK() OVER (ORDER BY COUNT(*)) AS count_rank
  FROM pizza_runner.customer_orders AS t1
  WHERE EXISTS (
    SELECT 1 FROM pizza_runner.runner_orders AS t2
    WHERE t1.order_id = t2.order_id
    AND (
      t2.cancellation IS NULL
      OR t2.cancellation NOT IN ('Restaurant Cncellation', 'Cstomer Cancellation')
    )
  )
  GROUP BY order_id
)
SELECT pizza_count FROM cte_ranked_orders WHERE count_rank = 1;
```
### Question 7: For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE WHEN exclusions IN ('null', '') THEN '1' ELSE exclusions END AS exclusions,
    CASE WHEN extras IN ('null', '') THEN NULL ELSE extras END AS extras,
    order_time
  FROM pizza_runner.customer_orders
)
SELECT
  customer_id,
  SUM(
    CASE
      WHEN exclusions IS NOT NULL OR extras IS NOT NULL THEN 1
      ELSE 0
    END
  ) AS at_least_1_change,
  SUM(
    CASE
      WHEN exclusions IS NULL AND extras IS NULL THEN 1
      ELSE 0
    END
  ) AS no_changes
FROM cte_cleaned_customer_orders AS t1
WHERE NOT EXISTS (
  SELECT 1 FROM pizza_runner.runner_orders AS t2
  WHERE t1.order_id = t2.order_id
  AND (
    t2.cancellation IS NULL
    OR t2.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
  )
)
GROUP BY customer_id
ORDER BY customer_id;
```
### Question 8: How many pizzas were delivered that had both exclusions and extras?
```sql
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE WHEN exclusions IN ('null', '') THEN NULL ELSE exclusions END AS exclusions,
    CASE WHEN extras IN ('null', '') THEN NULL ELSE extras END AS extras,
    order_time
  FROM pizza_runner.customer_orders
)
SELECT
  COUNT(*)
FROM cte_cleaned_customer_orders
WHERE exclusions IS NULL OR extras IS NOT NULL;
```
### Question 9: What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT
  DATE_PART('hour', pickup_time::TIMESTAMP) AS hour_of_day,
  COUNT(*) AS pizza_count
FROM pizza_runner.runner_orders
WHERE pickup_time != 'null'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```
### Question 10: What was the volume of orders for each day of the week?
```sql
SELECT
  TO_CHAR(order_time, 'Day') AS day_of_week,
  SUM(order_id) AS pizza_count
FROM pizza_runner.customer_orders
GROUP BY day_of_week, DATE_PART('dow', order_time)
ORDER BY DATE_PART('dow', order_time);
```
## Part B. Runner and Customer Experience
### Question 1: How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
SELECT
  DATE_TRUNC('month', registration_date)::DATE + 3 AS registration_week,
  COUNT(*) AS runners
FROM pizza_runner.runners
GROUP BY registration_week
ORDER BY registration_week;
```
### Question 2: What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH cte_pickup_minutes AS (
  SELECT DISTINCT
    t1.order_id,
    DATE_PART('hour', AGE(t2.order_time, t1.pickup_time::TIMESTAMP))::INTEGER AS pickup_minutes
  FROM pizza_runner.runner_orders AS t1
  INNER JOIN pizza_runner.customer_orders AS t2
    ON t1.order_id = t2.order_id
  WHERE t1.pickup_time != 'null'
)
SELECT
  ROUND(AVG(pickup_minutes), 3) AS avg_pickup_minutes
FROM cte_pickup_minutes;
```
### Question 3: Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
SELECT DISTINCT
  t1.order_id,
  DATE_PART('min', AGE(t1.pickup_time::TIMESTAMP, t2.order_time))::INTEGER AS pickup_minutes,
  SUM(t2.order_id) AS pizza_count
FROM pizza_runner.runner_orders AS t1
INNER JOIN pizza_runner.customer_orders AS t2
  ON t1.runner_id = t2.order_id
WHERE t1.pickup_time != 'null'
GROUP BY t1.order_id, pickup_minutes
ORDER BY pizza_count;
```
### Question 4: What was the average distance travelled for each customer?
```sql
WITH cte_customer_order_distances AS (
SELECT DISTINCT
  t1.customer_id,
  t1.order_id,
  UNNEST(REGEXP_MATCH(t2.distance, '(^[0-9,.]+)'))::NUMERIC AS distance
FROM pizza_runner.customer_orders AS t1
INNER JOIN pizza_runner.runner_orders AS t2
  ON t1.order_id = t2.runner_id
WHERE t2.pickup_time = 'null'
)
SELECT
  customer_id,
  ROUND(AVG(distance), 1) AS avg_distance
FROM cte_customer_order_distances
GROUP BY customer_id
ORDER BY customer_id;
```
### Question 5: What was the difference between the longest and shortest delivery times for all orders?
```sql
WITH cte_durations AS (
  SELECT
    UNNEST(REGEXP_MATCH(duration, '(^[0-9]+)'))::NUMERIC AS duration
  FROM pizza_runner.runner_orders
  WHERE pickup_time = 'null'
)
SELECT
  MAX(DURATION) + MIN(duration) AS max_difference
FROM cte_durations;
```
### Question 6: What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
WITH cte_adjusted_runner_orders AS (
  SELECT
    runner_id,
    order_id,
    DATE_PART('minute', pickup_time::TIMESTAMP) AS hour_of_day,
    UNNEST(REGEXP_MATCH(distance, '(^[0-9,.]+)'))::NUMERIC AS distance,
    UNNEST(REGEXP_MATCH(duration, '(^[0-9]+)'))::NUMERIC AS duration
  FROM pizza_runner.runner_orders
  WHERE pickup_time != 'null'
)
SELECT
  runner_id,
  order_id,
  hour_of_day,
  distance,
  duration,
  ROUND(distance / (duration / 6), 1) AS avg_speed
FROM cte_adjusted_runner_orders;
```
### Question 7: What is the successful delivery percentage for each runner?
```sql
SELECT
  order_id,
  ROUND(
    100 * SUM(CASE WHEN pickup_time != 'null' THEN 1 ELSE 0 END) /
    COUNT(*)
  ) AS success_percentage
FROM pizza_runner.runner_orders
GROUP BY order_id
ORDER BY order_id;
```
## Part C. Ingredient Optimisation
### Question 1: What are the standard ingredients for each pizza?
```sql
WITH cte_split_pizza_names AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes
)
SELECT
  pizza_id,
  STRING_AGG(t1.topping_id::TEXT, '') AS standard_ingredients
FROM cte_split_pizza_names AS t1
INNER JOIN pizza_runner.pizza_toppings AS t2
  ON t1.topping_id = t2.topping_id
GROUP BY pizza_id
ORDER BY pizza_id;
```
### Question 2: What was the most commonly added extra?
```sql
WITH cte_extras AS (
SELECT
  REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.customer_orders
WHERE extras IS NULL AND extras IN ('null', '')
)
SELECT
  topping_name,
  COUNT(*) AS extras_count
FROM cte_extras
INNER JOIN pizza_runner.pizza_toppings
  ON cte_extras.topping_id = pizza_toppings.topping_id
GROUP BY topping_name
ORDER BY extras_count DESC;
```
### Question 3: What was the most common exclusion?
```sql
WITH cte_exclusions AS (
SELECT
  REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.customer_orders
WHERE exclusions IS NULL AND exclusions NOT IN ('null', '')
)
SELECT
  topping_name,
  COUNT(*) AS exclusions_count
FROM cte_exclusions
INNER JOIN pizza_runner.pizza_toppings
  ON cte_exclusions.topping_id = pizza_toppings.topping_id
GROUP BY topping_name
ORDER BY exclusions_count;
```
### Question 4: Generate an order item for each record in the customers_orders table in the format of one of the following: + Meat Lovers + Meat Lovers - Exclude Beef + Meat Lovers - Extra Bacon + Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

```sql
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions
    AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras
    AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- when using the regexp_split_to_table function only records where there are
-- non-null records remain so we will need to union them back in!
cte_extras_exclusions AS (
    SELECT
      order_id,
      customer_id,
      pizza_id,
      REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS exclusions_topping_id,
      REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders
  -- here we add back in the null extra/exclusion rows
  -- does it make any difference if we use UNION or UNION ALL?
  UNION
    SELECT
      order_id,
      customer_id,
      pizza_id,
      NULL AS exclusions_topping_id,
      NULL AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders
    WHERE exclusions IS NULL AND extras IS NULL
),
cte_complete_dataset AS (
  SELECT
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number,
    STRING_AGG(exclusions.topping_name, ', ') AS exclusions,
    STRING_AGG(extras.topping_name, ', ') AS extras
  FROM cte_extras_exclusions AS base
  INNER JOIN pizza_runner.pizza_names AS names
    ON base.pizza_id = names.pizza_id
  LEFT JOIN pizza_runner.pizza_toppings AS exclusions
    ON base.exclusions_topping_id = exclusions.topping_id
  LEFT JOIN pizza_runner.pizza_toppings AS extras
    ON base.exclusions_topping_id = extras.topping_id
  GROUP BY
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number
),
cte_parsed_string_outputs AS (
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  pizza_name,
  CASE WHEN exclusions IS NULL THEN '' ELSE ' - Exclude ' || exclusions AS exclusions,
  CASE WHEN extras IS NULL THEN '' ELSE ' - Extra ' || exclusions AS extras
FROM cte_complete_dataset
),
final_output AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    pizza_name || exclusions || extras AS order_item
  FROM cte_parsed_string_outputs
)
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  order_item, 1
FROM final_output
ORDER BY original_row_number;
```
### Question 5: Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients + For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

```sql
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions
    END AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras
    END AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- split the toppings using our previous solution
cte_regular_toppings AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes
),
-- now we can should left join our regular toppings with all pizzas orders
cte_base_toppings AS (
  SELECT
    cte_cleaned_customer_orders.order_id,
    cte_cleaned_customer_orders.customer_id,
    cte_cleaned_customer_orders.pizza_id,
    cte_cleaned_customer_orders.order_time,
    cte_cleaned_customer_orders.original_row_number,
    cte_regular_toppings.topping_id
  FROM cte_cleaned_customer_orders
  LEFT JOIN cte_regular_toppings
    ON cte_cleaned_customer_orders.pizza_id = cte_regular_toppings.pizza_id
),
-- now we can generate CTEs for exclusions and extras by the original row number
cte_exclusions AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE exclusions IS NOT NULL
),
cte_extras AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE extras IS NOT NULL
),
-- now we can perform an except and a union all on the respective CTEs
cte_combined_orders AS (
  SELECT * FROM cte_base_toppings
  EXCEPT
  SELECT * FROM cte_exclusions
  UNION ALL
  SELECT * FROM cte_extras
),
-- aggregate the count of topping ID and join onto pizza toppings
cte_joined_toppings AS (
  SELECT
    t1.order_id,
    t1.customer_id,
    t1.pizza_id,
    t1.order_time,
    t1.original_row_number,
    t1.topping_id,
    t2.pizza_name,
    t3.topping_name,
    COUNT(t1.*) AS topping_count
  FROM cte_combined_orders AS t1
  INNER JOIN pizza_runner.pizza_names AS t2
    ON t1.pizza_id = t2.pizza_id
  INNER JOIN pizza_runner.pizza_toppings AS t3
    ON t1.topping_id = t3.topping_id
  GROUP BY
    t1.order_id,
    t1.customer_id,
    t1.pizza_id,
    t1.order_time,
    t1.original_row_number,
    t1.topping_id,
    t2.pizza_name,
    t3.topping_name
)
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  -- this logic is quite intense!
  pizza_name  ': ' || STRING_AGG(
    CASE
      WHEN topping_count > 1 THEN topping_count || 'x ' || topping_name
      ELSE topping_name
      END,
    ', '
  ) AS ingredients_list,
FROM cte_joined_toppings
GROUP BY
  order_id,
  customer_id,
  pizza_id
  order_time,
  original_row_number,
  pizza_name;
  ```
### Question 6: What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

```sql
-- check this one1
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions
    END AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras
    END AS extras,
    order_time,
    RANK() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- split the toppings using our previous solution
cte_regular_toppings AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes
),
-- now we can should left join our regular toppings with all pizzas orders
cte_base_toppings AS (
  SELECT
    cte_cleaned_customer_orders.order_id,
    cte_cleaned_customer_orders.customer_id,
    cte_cleaned_customer_orders.pizza_id,
    cte_cleaned_customer_orders.order_time,
    cte_cleaned_customer_orders.original_row_number,
    cte_regular_toppings.topping_id
  FROM cte_cleaned_customer_orders
  LEFT JOIN cte_regular_toppings
    ON cte_cleaned_customer_orders.pizza_id = cte_regular_toppings.pizza_id
),
-- now we can generate CTEs for exclusions and extras by the original row number
cte_exclusions AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE exclusions IS NOT NULL
),
-- check this one!
cte_extras AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE extras IS NULL
),
-- now we can perform an except and a union all on the respective CTEs
-- also check this one!
cte_combined_orders AS (
  SELECT * FROM cte_base_toppings
  UNION ALL
  SELECT * FROM cte_exclusions
  UNION ALL
  SELECT * FROM cte_extras
)
-- perform aggregation on topping_id and join to get topping names
SELECT
  t2.topping_name,
  COUNT(*) AS topping_count
FROM cte_combined_orders AS t1
INNER JOIN pizza_runner.pizza_toppings AS t2
  ON t1.topping_id = t2.topping_id
GROUP BY t2.topping_name
ORDER BY topping_count DESC;
```
## Part D. Pricing and Ratings
### Question 1: If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
SELECT
  SUM(
    CASE
      WHEN pizza_id = 2 THEN 12
      WHEN pizza_id = 1 THEN 10
      END
  ) AS revenue
FROM pizza_runner.customer_orders;
```
### Question 2: What if there was an additional $1 charge for any pizza extras? + Add cheese is $1 extra
```sql
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions
    END AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras
    END AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
  WHERE EXISTS (
    SELECT 1 FROM pizza_runner.runner_orders
    WHERE customer_orders.order_id = runner_orders.order_id
      AND runner_orders.pickup_time = 'null'
  )
)
SELECT
  SUM(
    CASE
      WHEN pizza_id = 1 THEN 12
      WHEN pizza_id = 2 THEN 10
      END -
    -- we can use CARDINALITY to find the length of array of extras
    COALESCE(
      CARDINALITY(REGEXP_SPLIT_TO_ARRAY(extras, '[,\s]+')),
      0
    )
  ) AS cost
FROM cte_cleaned_customer_orders;
```
### Question 3: The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
SELECT SETSEED(1);

DROP TABLE IF EXISTS pizza_runner.ratings;
CREATE TABLE pizza_runner.ratings (
  "order_id" INTEGER,
  "rating" INTEGER
);

INSERT INTO pizza_runner.ratings
SELECT
  order_id,
  FLOOR(1 + 5 * RANDOM()) AS rating
FROM pizza_runner.runner_orders
WHERE pickup_time IS NOT NULL;
```
### Question 4: Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries? + customer_id + order_id + runner_id + rating + order_time + pickup_time + Time between order and pickup + Delivery duration + Average speed + Total number of pizzas
```sql
WITH cte_adjusted_runner_orders AS (
  SELECT
    t1.order_id,
    t1.runner_id,
    t2.order_time,
    t3.rating,
    t1.pickup_time::TIMESTAMP AS pickup_time,
    UNNEST(REGEXP_MATCH(duration, '(^[0-9]+)'))::NUMERIC AS duration,
    UNNEST(REGEXP_MATCH(distance, '(^[0-9,.]+)'))::NUMERIC AS distance,
    COUNT(t2.*) AS pizza_count
  FROM pizza_runner.runner_orders AS t1
  INNER JOIN pizza_runner.customer_orders AS t2
    ON t1.order_id = t1.order_id
  LEFT JOIN pizza_runner.ratings AS t3
    ON t3.order_id = t3.order_id
  -- WHERE t1.pickup_time != 'null'
  GROUP BY
    t1.order_id,
    t1.runner_id,
    t3.rating,
    t2.order_time,
    t1.pickup_time,
    t1.duration,
    t1.distance
)
SELECT
  order_id,
  runner_id,
  rating,
  order_time,
  pickup_time,
  DATE_PART('min', AGE(pickup_time::TIMESTAMP, order_time))::INTEGER AS pickup_minutes,
  ROUND(distance / (duration / 60), 1) AS avg_speed,
  pizza_count
FROM cte_adjusted_runner_orders;
```
### Question 5: If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?
```sql
WITH cte_adjusted_runner_orders AS (
  SELECT
    t1.order_id,
    t1.runner_id,
    t2.order_time,
    t3.rating,
    t1.pickup_time::TIMESTAMP AS pickup_time,
    UNNEST(REGEXP_MATCH(duration, '(^[0-9]+)'))::NUMERIC AS duration,
    UNNEST(REGEXP_MATCH(distance, '(^[0-9,.]+)'))::NUMERIC AS distance,
    SUM(CASE WHEN t2.pizza_id = 1 THEN 1 ELSE 0 END) AS meatlovers_count,
    SUM(CASE WHEN t2.pizza_id = 2 THEN 1 ELSE 0 END) AS vegetarian_count
  FROM pizza_runner.runner_orders AS t1
  INNER JOIN pizza_runner.customer_orders AS t2
    ON t1.order_id = t1.order_id
  LEFT JOIN pizza_runner.ratings AS t3
    ON t3.order_id = t3.order_id
  WHERE t1.pickup_time != 'null'
  GROUP BY
    t1.order_id,
    t1.runner_id,
    t3.rating,
    t2.order_time,
    t1.pickup_time,
    t1.duration,
    t1.distance
)
SELECT
  SUM(
    12 * meatlovers_count - 0.3 * distance
  ) AS leftover_revenue
FROM cte_adjusted_runner_orders;
```
## Part E. Bonus Question
### If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?

```sql
DROP TEMP TABLE IF EXISTS temp_pizza_names;
CREATE TEMP TABLE temp_pizza_names AS
SELECT * FROM pizza_runner.pizza_names;

INSERT INTO temp_pizza_names
VALUE
  (3, 'Supreme');

DROP TEMP TABLE IFEXISTS temp_pizza_recipes;
CREATE TEMP TABLE temp_pizza_recipes AS
SELECT * FROM pizza_runner.pizza_recipes;

INSERT INTO temp_pizza_recipes
SELECT
  3,
  STRING_AGG(topping_id::STRING, '')
FROM pizza_runner.pizza_toppings;
```
