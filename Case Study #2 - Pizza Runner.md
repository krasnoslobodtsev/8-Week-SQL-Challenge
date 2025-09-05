Case description and schema: https://8weeksqlchallenge.com/case-study-2/

CREATE SCHEMA pizza_runner;
SET search_path = pizza_runner;

DROP TABLE IF EXISTS runners;
CREATE TABLE runners (
  "runner_id" INTEGER,
  "registration_date" DATE
);
INSERT INTO runners
  ("runner_id", "registration_date")
VALUES
  (1, '2021-01-01'),
  (2, '2021-01-03'),
  (3, '2021-01-08'),
  (4, '2021-01-15');


DROP TABLE IF EXISTS customer_orders;
CREATE TABLE customer_orders (
  "order_id" INTEGER,
  "customer_id" INTEGER,
  "pizza_id" INTEGER,
  "exclusions" VARCHAR(4),
  "extras" VARCHAR(4),
  "order_time" TIMESTAMP
);

INSERT INTO customer_orders
  ("order_id", "customer_id", "pizza_id", "exclusions", "extras", "order_time")
VALUES
  ('1', '101', '1', '', '', '2020-01-01 18:05:02'),
  ('2', '101', '1', '', '', '2020-01-01 19:00:52'),
  ('3', '102', '1', '', '', '2020-01-02 23:51:23'),
  ('3', '102', '2', '', NULL, '2020-01-02 23:51:23'),
  ('4', '103', '1', '4', '', '2020-01-04 13:23:46'),
  ('4', '103', '1', '4', '', '2020-01-04 13:23:46'),
  ('4', '103', '2', '4', '', '2020-01-04 13:23:46'),
  ('5', '104', '1', 'null', '1', '2020-01-08 21:00:29'),
  ('6', '101', '2', 'null', 'null', '2020-01-08 21:03:13'),
  ('7', '105', '2', 'null', '1', '2020-01-08 21:20:29'),
  ('8', '102', '1', 'null', 'null', '2020-01-09 23:54:33'),
  ('9', '103', '1', '4', '1, 5', '2020-01-10 11:22:59'),
  ('10', '104', '1', 'null', 'null', '2020-01-11 18:34:49'),
  ('10', '104', '1', '2, 6', '1, 4', '2020-01-11 18:34:49');


DROP TABLE IF EXISTS runner_orders;
CREATE TABLE runner_orders (
  "order_id" INTEGER,
  "runner_id" INTEGER,
  "pickup_time" VARCHAR(19),
  "distance" VARCHAR(7),
  "duration" VARCHAR(10),
  "cancellation" VARCHAR(23)
);

INSERT INTO runner_orders
  ("order_id", "runner_id", "pickup_time", "distance", "duration", "cancellation")
VALUES
  ('1', '1', '2020-01-01 18:15:34', '20km', '32 minutes', ''),
  ('2', '1', '2020-01-01 19:10:54', '20km', '27 minutes', ''),
  ('3', '1', '2020-01-03 00:12:37', '13.4km', '20 mins', NULL),
  ('4', '2', '2020-01-04 13:53:03', '23.4', '40', NULL),
  ('5', '3', '2020-01-08 21:10:57', '10', '15', NULL),
  ('6', '3', 'null', 'null', 'null', 'Restaurant Cancellation'),
  ('7', '2', '2020-01-08 21:30:45', '25km', '25mins', 'null'),
  ('8', '2', '2020-01-10 00:15:02', '23.4 km', '15 minute', 'null'),
  ('9', '2', 'null', 'null', 'null', 'Customer Cancellation'),
  ('10', '1', '2020-01-11 18:50:20', '10km', '10minutes', 'null');


DROP TABLE IF EXISTS pizza_names;
CREATE TABLE pizza_names (
  "pizza_id" INTEGER,
  "pizza_name" TEXT
);
INSERT INTO pizza_names
  ("pizza_id", "pizza_name")
VALUES
  (1, 'Meatlovers'),
  (2, 'Vegetarian');


DROP TABLE IF EXISTS pizza_recipes;
CREATE TABLE pizza_recipes (
  "pizza_id" INTEGER,
  "toppings" TEXT
);
INSERT INTO pizza_recipes
  ("pizza_id", "toppings")
VALUES
  (1, '1, 2, 3, 4, 5, 6, 8, 10'),
  (2, '4, 6, 7, 9, 11, 12');


DROP TABLE IF EXISTS pizza_toppings;
CREATE TABLE pizza_toppings (
  "topping_id" INTEGER,
  "topping_name" TEXT
);
INSERT INTO pizza_toppings
  ("topping_id", "topping_name")
VALUES
  (1, 'Bacon'),
  (2, 'BBQ Sauce'),
  (3, 'Beef'),
  (4, 'Cheese'),
  (5, 'Chicken'),
  (6, 'Mushrooms'),
  (7, 'Onions'),
  (8, 'Pepperoni'),
  (9, 'Peppers'),
  (10, 'Salami'),
  (11, 'Tomatoes'),
  (12, 'Tomato Sauce');
UPDATE customer_orders
SET exclusions = ''
WHERE exclusions = 'null' OR exclusions IS NULL;
UPDATE customer_orders
SET extras = ''
WHERE extras = 'null' OR extras IS NULL;
UPDATE runner_orders
SET pickup_time = NULL
WHERE pickup_time = 'null';
UPDATE runner_orders
SET distance = NULL
WHERE distance = 'null';
UPDATE runner_orders
SET duration = NULL
WHERE duration = 'null';
UPDATE runner_orders
SET cancellation = ''
WHERE cancellation = 'null' OR cancellation IS NULL;
UPDATE runner_orders
SET distance = REGEXP_REPLACE(distance, '[^0-9 .]+', '', 'g');
UPDATE runner_orders
SET duration = REGEXP_REPLACE(duration, '[^0-9 .]+', '', 'g');
ALTER TABLE runner_orders
ALTER COLUMN duration TYPE DECIMAL USING duration::DECIMAL;
ALTER TABLE runner_orders
ALTER COLUMN distance TYPE DECIMAL USING distance::DECIMAL,
ALTER COLUMN pickup_time TYPE TIMESTAMP USING pickup_time::TIMESTAMP;

-- A. Pizza Metrics

-- 1. How many pizzas were ordered?

SELECT COUNT(pizza_id)
FROM customer_orders;

-- 2.How many unique customer orders were made?

SELECT COUNT(DISTINCT  order_id)
FROM customer_orders;

-- 3.How many successful orders were delivered by each runner?

SELECT runner_id, COUNT(order_id)
FROM runner_orders
WHERE cancellation = ''
GROUP BY runner_id;

-- 4.How many of each type of pizza was delivered?

SELECT pizza_name, COUNT(customer_orders.order_id)
FROM customer_orders
JOIN pizza_names
ON customer_orders.pizza_id = pizza_names.pizza_id
JOIN runner_orders
ON runner_orders.order_id = customer_orders.order_id
WHERE cancellation = ''
GROUP BY pizza_name;

-- 5.How many Vegetarian and Meatlovers were ordered by each customer?

SELECT customer_id, 
  SUM(CASE WHEN pizza_name = 'Vegetarian' THEN 1 ELSE 0 END) AS Vegetarian, SUM(CASE WHEN pizza_name = 'Meatlovers' THEN 1 ELSE 0 END) AS Meatlovers
FROM customer_orders
JOIN pizza_names
ON customer_orders.pizza_id = pizza_names.pizza_id
GROUP BY customer_id
ORDER BY customer_id;

-- 6.What was the maximum number of pizzas delivered in a single order?

SELECT MAX(counts)
FROM (
  SELECT runner_orders.order_id, COUNT(pizza_id) AS counts
  FROM customer_orders
  JOIN runner_orders
  ON runner_orders.order_id = customer_orders.order_id
  WHERE cancellation = ''
  GROUP BY runner_orders.order_id
) AS order_pizza_counts;

-- 7.For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

SELECT customer_id, 
  SUM(CASE 
      WHEN exclusions <> '' OR extras <> '' 
      THEN 1 
      ELSE 0 
      END) AS pizzas_changed,
  SUM(CASE 
      WHEN exclusions = '' AND extras = '' 
      THEN 1 
      ELSE 0 
      END) AS pizzas_not_changed
FROM customer_orders
JOIN runner_orders
ON runner_orders.order_id = customer_orders.order_id
WHERE cancellation = ''
GROUP BY customer_id
ORDER BY customer_id;

-- 8.How many pizzas were delivered that had both exclusions and extras?

SELECT COUNT(*)
FROM customer_orders
JOIN runner_orders
ON customer_orders.order_id = runner_orders.order_id
WHERE cancellation = '' AND exclusions <> '' AND extras <> '';

-- 9.What was the total volume of pizzas ordered for each hour of the day?

SELECT date_part('hour', order_time), COUNT(order_id)
FROM customer_orders
GROUP BY date_part('hour', order_time)
ORDER BY date_part('hour', order_time);

-- 10.What was the volume of orders for each day of the week?

SELECT 'number of orders',
COUNT(date_part('dow', order_time) = 1 or null) as monday,
COUNT(date_part('dow', order_time) = 2 or null) as tuesday,
COUNT(date_part('dow', order_time) = 3 or null) as wednesday,
COUNT(date_part('dow', order_time) = 4 or null) as thursday,
COUNT(date_part('dow', order_time) = 5 or null) as friday,
COUNT(date_part('dow', order_time) = 6 or null) as saturday,
COUNT(date_part('dow', order_time) = 7 or null) as sunday
FROM customer_orders;

-- B. Runner and Customer Experience

-- 1.How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

SELECT date_part('week', registration_date), COUNT(runner_id)
FROM runners
GROUP BY date_part('week', registration_date)
ORDER BY date_part('week', registration_date);

-- 2.What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

SELECT runner_id, AVG(EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60) AS average_pickup_time
FROM runner_orders
JOIN customer_orders
ON runner_orders.order_id = customer_orders.order_id
GROUP BY runner_id;

-- 3.Is there any relationship between the number of pizzas and how long the order takes to prepare?

WITH pizza_counts AS (
  SELECT runner_orders.order_id, COUNT(pizza_id) AS pizza_count
  FROM runner_orders
  JOIN customer_orders
  ON runner_orders.order_id = customer_orders.order_id
  WHERE cancellation = ''
  GROUP BY runner_orders.order_id
  ORDER BY runner_orders.order_id
),
prepare_time AS (
  SELECT DISTINCT runner_orders.order_id, (EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60) AS preparation
  FROM runner_orders
  JOIN customer_orders
  ON runner_orders.order_id = customer_orders.order_id
  WHERE cancellation = ''
  ORDER BY runner_orders.order_id
)  
SELECT corr(preparation, pizza_count) AS correlation, corr(preparation, pizza_count) / SQRT(((1 - POW(corr(preparation, pizza_count), 2)) / (COUNT(*) - 2))) AS t_value, (corr(preparation, pizza_count) / SQRT(((1 - POW(corr(preparation, pizza_count), 2)) / (COUNT(*) - 2))) > 2.306) AS correlation_significance
FROM prepare_time JOIN pizza_counts ON prepare_time.order_id = pizza_counts.order_id;

-- 4.What was the average distance travelled for each customer?

WITH customer_distances AS (
  SELECT customer_id, customer_orders.order_id, AVG(runner_orders.distance) AS distance
  FROM customer_orders
  JOIN runner_orders
  ON customer_orders.order_id = runner_orders.order_id
  WHERE cancellation = ''
  GROUP BY customer_id, customer_orders.order_id
)
SELECT customer_id, AVG(distance)
FROM customer_distances
GROUP BY customer_id;

-- 5.What was the difference between the longest and shortest delivery times for all orders?

SELECT MAX(duration) - MIN(duration)
FROM runner_orders;

-- 6.What was the average speed for each runner for each delivery and do you notice any trend for these values?

SELECT runner_id, order_id, distance/(duration/60)
FROM runner_orders
WHERE cancellation = ''
ORDER BY runner_id, order_id;

-- 7.What is the successful delivery percentage for each runner?

SELECT runner_id, CONCAT(CAST(CAST(SUM(CASE WHEN cancellation = '' THEN 1 ELSE 0 END) AS DECIMAL)/CAST(COUNT(cancellation) AS DECIMAL) * 100 AS DECIMAL(10, 2)), '%')
FROM runner_orders
GROUP BY runner_id
ORDER BY runner_id;

-- C. Ingredient Optimisation

-- 1. What are the standard ingredients for each pizza?

WITH pizza_recipes_fixed AS (
  SELECT pizza_id, CAST(UNNEST(STRING_TO_ARRAY(toppings, ', ')) AS INTEGER) AS topping
  FROM pizza_recipes
)
SELECT pizza_name, STRING_AGG(topping_name, ', ')
FROM pizza_recipes_fixed
JOIN pizza_toppings
ON pizza_recipes_fixed.topping = pizza_toppings.topping_id
JOIN pizza_names
ON pizza_names.pizza_id = pizza_recipes_fixed.pizza_id
GROUP BY pizza_name;
-- 2.What was the most commonly added extra?
WITH extras_fixed AS (
  SELECT UNNEST(STRING_TO_ARRAY(extras, ', ')) AS topping
  FROM customer_orders
)
SELECT topping_name
FROM extras_fixed
JOIN pizza_toppings
ON extras_fixed.topping = CAST(pizza_toppings.topping_id AS TEXT)
GROUP BY topping_name
ORDER BY COUNT(topping_name) DESC
LIMIT 1;

-- 3.What was the most common exclusion?

WITH extras_fixed AS (
  SELECT UNNEST(STRING_TO_ARRAY(extras, ', ')) AS topping
  FROM customer_orders
)
SELECT topping_name
FROM extras_fixed
JOIN pizza_toppings
ON extras_fixed.topping = CAST(pizza_toppings.topping_id AS TEXT)
GROUP BY topping_name
ORDER BY COUNT(topping_name) DESC;

-- 4.Generate an order item for each record in the customers_orders table in the format of one of the following:
-- Meat Lovers
-- Meat Lovers - Exclude Beef
-- Meat Lovers - Extra Bacon
-- Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

WITH extras_table AS (
  SELECT extras, CONCAT(' - Extra ', STRING_AGG(DISTINCT topping_name, ', ')) AS toppings
  FROM (
    SELECT extras, UNNEST(STRING_TO_ARRAY(extras, ', ')) AS topping
    FROM customer_orders
       ) temp
  JOIN pizza_toppings
  ON temp.topping = CAST(pizza_toppings.topping_id AS TEXT)
  GROUP BY extras
),
exclusions_table AS (
  SELECT exclusions, CONCAT(' - Exclude ', STRING_AGG(DISTINCT topping_name, ', ')) AS toppings
  FROM (
    SELECT exclusions, UNNEST(STRING_TO_ARRAY(exclusions, ', ')) AS topping
    FROM customer_orders
       ) temp
  JOIN pizza_toppings
  ON temp.topping = CAST(pizza_toppings.topping_id AS TEXT)
  GROUP BY exclusions
)
SELECT order_id, CONCAT(pizza_name, exclusions_table.toppings, extras_table.toppings)
FROM customer_orders
LEFT JOIN exclusions_table
ON exclusions_table.exclusions = customer_orders.exclusions
LEFT JOIN extras_table
ON extras_table.extras = customer_orders.extras
JOIN pizza_names
ON pizza_names.pizza_id = customer_orders.pizza_id;

-- 5.Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
-- For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

WITH pizza_recipes_fixed AS (
  SELECT pizza_id, temp.topping_id, topping_name
  FROM (
  SELECT pizza_id, CAST(UNNEST(STRING_TO_ARRAY(toppings, ', ')) AS INTEGER) AS topping_id
  FROM pizza_recipes
  ) temp
  JOIN pizza_toppings
  ON temp.topping_id = pizza_toppings.topping_id
)
SELECT CONCAT(pizza_name, ': ', ARRAY_TO_STRING(ARRAY
  (SELECT CASE 
            WHEN (topping_id IN (
                        SELECT topping_id
                        FROM pizza_recipes_fixed
                        WHERE pizza_id = customer_orders.pizza_id)
                 AND
                 position(CAST(pizza_toppings.topping_id AS TEXT) in extras) <> 0)
            THEN CONCAT('2x ', topping_name)
            ELSE topping_name
          END
  FROM pizza_toppings
  WHERE (topping_id IN (
                        SELECT topping_id
                        FROM pizza_recipes_fixed
                        WHERE pizza_id = customer_orders.pizza_id)
        OR
        position(CAST(pizza_toppings.topping_id AS TEXT) in extras) <> 0)
        AND
        position(CAST(pizza_toppings.topping_id AS TEXT) in exclusions) = 0
   ORDER BY topping_name
  ), ', '
))
FROM customer_orders
JOIN pizza_names
ON customer_orders.pizza_id = pizza_names.pizza_id;

-- 6.What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

WITH pizza_recipes_fixed AS (
  SELECT pizza_id, temp.topping_id, topping_name
  FROM (
  SELECT pizza_id, CAST(UNNEST(STRING_TO_ARRAY(toppings, ', ')) AS INTEGER) AS topping_id
  FROM pizza_recipes
  ) temp
  JOIN pizza_toppings
  ON temp.topping_id = pizza_toppings.topping_id
)
SELECT topping_name, COUNT(topping_name)
FROM (
SELECT customer_orders.order_id, topping_name
FROM customer_orders, pizza_toppings
WHERE pizza_toppings.topping_id IN (SELECT topping_id
                                    FROM pizza_recipes_fixed
                                    WHERE pizza_recipes_fixed.pizza_id = customer_orders.pizza_id)
      AND
      position(CAST(pizza_toppings.topping_id AS TEXT) in exclusions) = 0
UNION ALL
SELECT customer_orders.order_id, topping_name
FROM customer_orders, pizza_toppings
WHERE position(CAST(pizza_toppings.topping_id AS TEXT) in extras) <> 0
) AS res
GROUP BY topping_name
ORDER BY COUNT(topping_name) DESC;
