![Logo](PizzaRunnerLogo.png)

# Case Study 2 - Pizza Runner
*This case study is part of the 8 weeks SQL challenge which you can find details [here](https://8weeksqlchallenge.com/)

## Introduction
Danny was scrolling through his Instagram feed when something really caught his eye - “80s Retro Styling and Pizza Is The Future!”

Danny was sold on the idea, but he knew that pizza alone was not going to help him get seed funding to expand his new Pizza Empire - so he had one more genius idea to combine with it - he was going to Uberize it - and so Pizza Runner was launched!

Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

## Problem Statement and Available Data
Danny is  very aware that data collection was going to be critical for his business’ growth. He has prepared for us an entity relationship diagram (see below) of his database design but requires further assistance to clean his data and apply some basic calculations so he can better direct his runners and optimise Pizza Runner’s operations.

![DataERD](PizzaRunnerERD.png)

## Danny's questions and my SQL solutions:

### Pizza Metrics

**1. How many pizzas were ordered?**

```sql
SELECT
  COUNT(pizza_id) 
FROM pizza_runner.customer_orders;
```

**Output**

count |
----  |
14    |

**2. How many unique customer orders were made?**

```sql
SELECT
  COUNT(DISTINCT order_id)
FROM pizza_runner.customer_orders;
```
**Output**

count |
----  |
10    |

**3. How many successful orders were delivered by each runner?**

```sql
SELECT
  runner_id,
  COUNT(DISTINCT order_id) 
FROM pizza_runner.runner_orders
WHERE cancellation IS NULL 
  OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
GROUP BY runner_id
ORDER BY 2 DESC;
```

**Output**
runner_id   | count
----------- | ------------
1           | 4
2           | 3
3           | 1

**4. How many of each type of pizza was delivered?**

```sql
SELECT
  names.pizza_name,
  COUNT(delivery.*) AS number_ordered
FROM pizza_runner.runner_orders AS delivery
INNER JOIN pizza_runner.customer_orders AS orders
  ON delivery.order_id = orders.order_id
INNER JOIN pizza_runner.pizza_names AS names
  ON orders.pizza_id = names.pizza_id
WHERE delivery.cancellation IS NULL
  OR delivery.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
GROUP BY names.pizza_name
ORDER BY number_ordered DESC;
```
**Output**

pizza_name          | number_ordered
---------         | ------------
Meatloves           | 9
Vegetarian          | 3

**5. How many Vegetarian and Meatlovers were ordered by each customer?**

```sql
SELECT
  orders.customer_id,
  SUM(CASE WHEN names.pizza_name = 'Vegetarian' THEN 1 ELSE 0 END) AS vegetarian,
  SUM(CASE WHEN names.pizza_name = 'Meatlovers' THEN 1 ELSE 0 END) AS meatlovers
FROM pizza_runner.customer_orders AS orders
INNER JOIN pizza_runner.pizza_names AS names
  ON orders.pizza_id = names.pizza_id
GROUP BY
  orders.customer_id
ORDER BY customer_id;
```

**Output**

customer_id   |vegetarian    | meatlovers
---------     | ----------   | ------------
101  |  2     | 1
102  |  2     | 1
103  |  3     | 1
104  | 3      | 0
105  | 0     | 1

**6. What was the maximum number of pizzas delivered in a single order?**

```sql
SELECT
  runner.order_id,
  COUNT(orders.pizza_id) AS pizza_delivered
FROM pizza_runner.runner_orders AS runner
INNER JOIN pizza_runner.customer_orders AS orders
  ON runner.order_id = orders.order_id
WHERE runner.cancellation IS NULL
  OR runner.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
GROUP BY runner.order_id
ORDER BY pizza_delivered DESC
LIMIT 1;
```
**Output**

order_id  |   pizza_delivered
---- | ---
4   | 3

**7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**

```sql
WITH orders AS(
SELECT
  customer_id,
  order_id,
  CASE WHEN exclusions IN ('null', '') THEN NULL ELSE exclusions END,
  CASE WHEN extras IN ('null', '') THEN NULL ELSE extras END
FROM pizza_runner.customer_orders
)

SELECT
  t1.customer_id,
  SUM(CASE WHEN t1.exclusions IS NULL OR t1.extras IS NULL THEN 1 ELSE 0 END) AS pizza_no_change,
  SUM(CASE WHEN t1.exclusions is NOT NULL OR t1.extras IS NOT NULL THEN 1 ELSE 0 END) AS pizza_with_change
FROM
  orders AS t1
INNER JOIN pizza_runner.runner_orders AS t2
  ON t1.order_id = t2.order_id
  WHERE (t2.cancellation IS NULL OR t2.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation'))
GROUP BY customer_id
ORDER BY customer_id;
```

**Output**

customer_id    |  pizza_no_change   | pizza_with_change
--- | ---- | ----
101  |  2  |  0
102  |  3  |  0
103  |  3  |  3
104  | 2   |  2  
105  | 1  | 1


