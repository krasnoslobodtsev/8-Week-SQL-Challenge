[Case description and schema](https://8weeksqlchallenge.com/case-study-3/)
```
CREATE VIEW subscriptions_plans AS 
  SELECT plans.plan_id, customer_id, plan_name, price, start_date
    FROM subscriptions
    JOIN plans
    ON plans.plan_id = subscriptions.plan_id
;
```
B. Data Analysis Questions

1. How many customers has Foodie-Fi ever had?
```
SELECT COUNT(DISTINCT customer_id)
FROM subscriptions;
```
2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
```
SELECT to_char(start_date, 'Month'), COUNT(*)
FROM subscriptions
WHERE plan_id = 0
GROUP BY to_char(start_date, 'Month');
```
3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
```
SELECT plan_name, COUNT(customer_id)
FROM subscriptions_plans
WHERE start_date > '2020-12-31'
GROUP BY plan_name;
```
4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?
```
SELECT COUNT(customer_id) AS churn_count, CONCAT(ROUND(COUNT(customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions)::numeric(10,1) * 100, 1), '%') AS churn_percentage
FROM subscriptions_plans
WHERE plan_id = 4;
```
5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?
```
SELECT CONCAT(ROUND(COUNT(*)::numeric / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions_plans WHERE plan_id = 0) * 100, 1), '%')
FROM
(SELECT start_date, LEAD(start_date, 1) OVER
                                       (PARTITION BY customer_id ORDER BY start_date) AS end_date
FROM subscriptions_plans
WHERE plan_id = 0 OR plan_id = 4) AS dates
WHERE end_date - start_date = 7;
```
6. What is the number and percentage of customer plans after their initial free trial?
```
WITH plans_after_trial AS (
SELECT *
FROM
(
SELECT plan_id, LEAD(plan_name, 1) OVER (PARTITION BY customer_id ORDER BY start_date) AS next_plan
FROM subscriptions_plans
) AS next_subscriptions
WHERE plan_id = 0
)
SELECT next_plan, ROUND(COUNT(plan_id)::numeric / (SELECT COUNT(*) FROM plans_after_trial) * 100, 1) AS percentage
FROM plans_after_trial
GROUP BY next_plan;
```
7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?
```
WITH up_to_date_plans AS (
SELECT DISTINCT customer_id, FIRST_VALUE(plan_name) OVER (PARTITION BY customer_id ORDER BY start_date DESC) AS plan
FROM subscriptions_plans
WHERE start_date < '2021-01-01'
)
SELECT plan, ROUND(COUNT(plan)::numeric / (SELECT COUNT(*) FROM up_to_date_plans) * 100, 1) AS percentage
FROM up_to_date_plans
GROUP BY plan;
```
8. How many customers have upgraded to an annual plan in 2020?
```
WITH previous_plans AS (
  SELECT plan_id, LAG(plan_id, 1) OVER (PARTITION BY customer_id ORDER BY start_date) AS previous_plan
  FROM subscriptions_plans
  WHERE start_date < '2021-01-01' AND start_date > '2019-12-31'
)
SELECT COUNT(*)
FROM previous_plans
WHERE plan_id = 3 AND previous_plan < 3;
```
9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
```
WITH join_dates AS (
  SELECT DISTINCT customer_id, MIN(start_date) OVER (PARTITION BY customer_id) AS join_date
  FROM subscriptions_plans
)
, switch_dates AS (
  SELECT customer_id, MIN(start_date) OVER (PARTITION BY customer_id) AS switch_date
  FROM subscriptions_plans
  WHERE plan_id = 3
)
SELECT AVG(switch_date - join_date)
FROM switch_dates
JOIN join_dates
ON switch_dates.customer_id = join_dates.customer_id;
```
10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
```
WITH join_dates AS (
  SELECT DISTINCT customer_id, MIN(start_date) OVER (PARTITION BY customer_id) AS join_date
  FROM subscriptions_plans
)
, switch_dates AS (
  SELECT customer_id, MIN(start_date) OVER (PARTITION BY customer_id) AS switch_date
  FROM subscriptions_plans
  WHERE plan_id = 3
), period_counts AS (
SELECT CONCAT(CASE
                                       WHEN (switch_date - join_date) / 30 * 30 + 1 = 1
                                       THEN 0
                                       ELSE (switch_date - join_date) / 30 * 30 + 1
                                       END
                                       , '-', ((switch_date - join_date) / 30 + 1) * 30, ' days') as period
FROM switch_dates
JOIN join_dates
ON switch_dates.customer_id = join_dates.customer_id
)
SELECT period, COUNT(*)
FROM period_counts
GROUP BY period
ORDER BY COUNT(*) DESC;
```
11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
```
WITH previous_plans AS (
  SELECT plan_id, LAG(plan_id, 1) OVER (PARTITION BY customer_id ORDER BY start_date) AS previous_plan
  FROM subscriptions_plans
  WHERE start_date < '2021-01-01' AND start_date > '2019-12-31'
)
SELECT COUNT(*)
FROM previous_plans
WHERE plan_id = 1 AND previous_plan = 2;
```
The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:

- monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
- upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
- upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period once a customer churns they will no longer make payments

```
WITH sb_periods AS (
	SELECT customer_id, plan_id, plan_name, price, start_date, COALESCE(LEAD(start_date, 1) OVER (PARTITION BY customer_id ORDER BY start_date) - 1, '2020-12-31') AS end_date
	FROM subscriptions_plans
	WHERE plan_id <> 0
), payments_undiscounted AS (
	SELECT customer_id, plan_name, plan_id, generate_series(start_date, end_date, interval '1 month') as payment_date, price
	FROM sb_periods
    WHERE plan_id in (1, 2)
    UNION
  	SELECT customer_id, plan_name, plan_id, start_date, price
    FROM sb_periods
    WHERE plan_id = 3
)
SELECT customer_id, 
       plan_id, 
       plan_name, 
       payment_date, 
       CASE 
             WHEN plan_id in (2, 3) 
             AND (LAG(plan_id, 1) OVER (PARTITION BY customer_id ORDER BY payment_date) = 1)
             AND (LAG(payment_date, 1) OVER (PARTITION BY customer_id ORDER BY payment_date) + interval '1 month' > payment_date)
             THEN price - 9.90
             ELSE price
             END as amount,
        RANK() OVER (PARTITION BY customer_id ORDER BY payment_date) AS payment_order
FROM payments_undiscounted;
```
