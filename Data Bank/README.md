![Logo](data-bank-logo.png)
*This case study is part of the 8 weeks SQL challenge which you can find details [here](https://8weeksqlchallenge.com/)

## Introduction
There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!

The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

## Available Data
The Data Bank team have prepared a data model for this case study as well as a few example rows from the complete dataset below to get you familiar with their tables.
![DataERD](data-bank-ERD.png)

<details>
<summary> Table 1: Regions</summary>
  
Just like popular cryptocurrency platforms - Data Bank is also run off a network of nodes where both money and data is stored across the globe. In a traditional banking sense - you can think of these nodes as bank branches or stores that exist around the world.

This regions table contains the region_id and their respective region_name values: <br>
![Region](region.png)
</details>

<details>
<summary> Table 2: Customer Nodes</summary>
  
Customers are randomly distributed across the nodes according to their region - this also specifies exactly which node contains both their cash and data.

This random distribution changes frequently to reduce the risk of hackers getting into Data Bank’s system and stealing customer’s money and data!

Below is a sample of the top 10 rows of the data_bank.customer_nodes: <br>
![CustomerNodes](customer-nodes.png)
</details>

<details>
<summary> Table 3: Customer Transactions</summary>
  
This table stores all customer deposits, withdrawals and purchases made using their Data Bank debit card:<br>
![CustomerTransactions](customer-transactions.png)
</details>

<br>

## The business questions and my SQL solutions:<br>


### Customer Nodes Exploration

**1. How many unique nodes are there on the Data Bank system?**

```sql
WITH unique_nodes AS(
SELECT
  region_id,
  COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes
GROUP BY region_id
)

SELECT
  SUM(unique_nodes) AS total_unique_nodes
FROM unique_nodes;
```

**Output**

total_unique_nodes
--|
25  |

<br>

**2. What is the number of nodes per region?**

```sql
SELECT
  r.region_name,
  COUNT(DISTINCT c.node_id)
FROM data_bank.customer_nodes AS c
INNER JOIN data_bank.regions AS r
  ON c.region_id = r.region_id
GROUP BY r.region_name;
```

**Output**

region_name | output
--  | --
Africa  | 5
America | 5
Asia  | 5
Australia | 5
Europe  | 5

<br>

**3. How many customers are allocated to each region?**

```sql
SELECT
  r.region_name,
  COUNT(DISTINCT c.customer_id) AS customer_number
FROM data_bank.customer_nodes AS c
INNER JOIN data_bank.regions AS r
ON c.region_id = r.region_id
GROUP BY r.region_name
ORDER BY customer_number DESC;
```

**Output**

region_name | customer_number
--  | --
Australia | 110
America | 105
Africa  | 102
Asia  | 95
Europe  | 88

<br>

**4. How many days on average are customers reallocated to a different node?**

The following code was my original solution which gave me the **output of 14 days** as average duration:

```sql

WITH node_change AS(
SELECT
  customer_id,
  node_id,
  region_id,
  DATE_PART('DAY', AGE(end_date, start_date))::INTEGER AS days_difference,
  ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY start_date) AS row_number
FROM data_bank.customer_nodes
)

SELECT
  ROUND(AVG(days_difference)) AS avg_duration
FROM node_change;
```
<br>

However, I have just been thrown a curve ball. Apparently, there are times where the same node is allocated to the same customer_id. Therefore to deal with this, we need to use recursive cte. Start the recursion by setting an initial 1 value for run_id for each customer record, using the start_date. If the current row does not match the previous row, then increment the run_id by 1:

```sql
DROP TABLE IF EXISTS ranked_customer_nodes;
CREATE TEMP TABLE ranked_customer_nodes AS
SELECT
  customer_id,
  node_id,
  region_id,
  DATE_PART('day', AGE(end_date, start_date))::INTEGER AS duration,
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) AS row_number
FROM data_bank.customer_nodes;


WITH RECURSIVE output_table AS (
SELECT
  customer_id,
  node_id,
  duration,
  row_number,
  1 as run_id
FROM ranked_customer_nodes
WHERE row_number = 1

UNION ALL

SELECT
  t1.customer_id,
  t2.node_id,
  t2.duration,
  t2.row_number,
  CASE WHEN t1.node_id <> t2.node_id THEN t1.run_id + 1 ELSE t1.run_id END AS run_id
FROM output_table AS t1
INNER JOIN ranked_customer_nodes AS t2
  ON t1.row_number + 1 = t2.row_number
  AND t1.customer_id = t2.customer_id
  AND t2.row_number > 1
),

cte AS(
  SELECT
    customer_id,
    run_id,
    SUM(duration) AS node_duration
  FROM output_table
  GROUP BY
    customer_id,
    run_id
)

SELECT
  ROUND(AVG(node_duration)) AS avg_duration
FROM cte;
```

**Output**
avg_duration
--  |
17  |

<br>

**5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**

```sql
DROP TABLE IF EXISTS ranked_customer_nodes;
CREATE TEMP TABLE ranked_customer_nodes AS
SELECT
  customer_id,
  node_id,
  region_id,
  DATE_PART('day', AGE(end_date, start_date))::INTEGER AS duration,
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) AS row_number
FROM data_bank.customer_nodes;


WITH RECURSIVE output_table AS (
SELECT
  customer_id,
  node_id,
  region_id,
  duration,
  row_number,
  1 as run_id
FROM ranked_customer_nodes
WHERE row_number = 1

UNION ALL

SELECT
  t1.customer_id,
  t2.node_id,
  t2.region_id,
  t2.duration,
  t2.row_number,
  CASE WHEN t1.node_id <> t2.node_id THEN t1.run_id + 1 ELSE t1.run_id END AS run_id
FROM output_table AS t1
INNER JOIN ranked_customer_nodes AS t2
  ON t1.row_number + 1 = t2.row_number
  AND t1.customer_id = t2.customer_id
  AND t2.row_number > 1
),

cte AS(
  SELECT
    customer_id,
    run_id,
    region_id,
    SUM(duration) AS node_duration
  FROM output_table
  GROUP BY
    customer_id,
    run_id,
    region_id
)

SELECT
  r.region_name,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY c.node_duration)) AS median_duration,
  ROUND(PERCENTILE_CONT(0.8) WITHIN GROUP (ORDER BY c.node_duration)) AS percentile_80_duration,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY c.node_duration)) AS percentile_95_duration
FROM cte AS c
INNER JOIN data_bank.regions AS r
  ON c.region_id = r.region_id
GROUP BY region_name;

```

**Output**

region_name | median_duration | percentile_80_duration  | percentile_95_duration
--  | --  | --  | --
Africa  | 17  | 27  | 40
America | 17  | 26  | 35
Asia  | 17  | 26  | 40
Australia | 17  | 25  | 38
Europe  | 17  | 27  | 37

<br>


---

### Customer Transactions

**1. What is the unique count and total amount for each transaction type?**

```sql
SELECT
  DISTINCT txn_type,
  COUNT(*) AS transaction_count,
  SUM(txn_amount) AS total_amount
FROM data_bank.customer_transactions
GROUP BY
  txn_type
ORDER BY total_amount DESC;
```

**Output**

txn_type  | transaction_count | total_amount
--  | --  | --
deposit | 2671  | 1359168
purchase  | 1617  | 806537
withdrawal  | 1580  | 793003

<br>

**2. What is the average total historical deposit counts and amounts for all customers?**

```sql
WITH cte AS(
SELECT
  customer_id,
  COUNT(*) AS deposit_counts,
  SUM(txn_amount) AS total_deposit_amt
FROM data_bank.customer_transactions
WHERE txn_type = 'deposit'
GROUP BY customer_id
)

SELECT
  ROUND(AVG(deposit_counts)) AS avg_deposit_counts,
  -- note you can't do a simple AVG for the total_deposit_amt as it will be an average of customer numbers rather than deposit counts
  ROUND(SUM(total_deposit_amt)/ SUM(deposit_counts)) AS avg_total_deposits
FROM cte;
```

**Output**

avg_deposit_counts  |  avg_total_deposits
--  | --
5 | 509

<br>

**3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**

```sql
WITH transaction_count AS(
SELECT
  customer_id,
  DATE_TRUNC('MONTH', txn_date) AS month,
  SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count,
  SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchase_count,
  SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawal_count
FROM data_bank.customer_transactions
GROUP BY customer_id, month
)

SELECT
  month,
  COUNT(DISTINCT customer_id) AS customer_count
FROM transaction_count
WHERE deposit_count > 1 AND (
  purchase_count >= 1 OR withdrawal_count >= 1
)
GROUP BY month
ORDER BY month;

```

**Output**

month | customer_count
--  | --
2020-01-01  | 168
2020-02-01  | 181
2020-03-01  | 192
2020-04-01  | 70

<br>

**4. What is the closing balance for each customer at the end of the month?**

```sql
WITH balance AS(
SELECT
  customer_id,
  DATE_TRUNC('MONTH', txn_date) AS month,
  SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS balance
FROM data_bank.customer_transactions
GROUP BY customer_id, month
ORDER BY customer_id, month
),

months AS(
SELECT
  DISTINCT(customer_id),
  ('2020-01-01':: DATE + GENERATE_SERIES (0, 3) * INTERVAL '1 MONTH'):: DATE AS month
FROM balance

)

SELECT
  b.customer_id,
  m.month,
  COALESCE(b.balance, 0) AS balance_contribution,
  SUM(b.balance) OVER (PARTITION BY b.customer_id ORDER BY m.month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS ending_balance
FROM months AS m
INNER JOIN balance AS b
  ON m.customer_id = b.customer_id
  AND m.month = b.month;

```

**Output**

Output is 1720 rows! You can download an exported csv version [here](Q4_closing_balance_output.csv) <br>

**5. Comparing the closing balance of a customer’s first month and the closing balance from their second month, what percentage of customers:** <br>
**Have a negative first month balance?** <br>
**Have a positive first month balance?** <br>
**Increase their opening month’s positive closing balance by more than 5% in the following month?** <br>
**Reduce their opening month’s positive closing balance by more than 5% in the following month?** <br>
**Move from a positive balance in the first month to a negative balance in the second month?** 

```sql
WITH balance AS(
SELECT
  customer_id,
  DATE_TRUNC('MONTH', txn_date) AS month,
  SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS balance
FROM data_bank.customer_transactions
GROUP BY customer_id, month
ORDER BY customer_id, month
),

months AS(
SELECT
  DISTINCT(customer_id),
  ('2020-01-01':: DATE + GENERATE_SERIES (0, 3) * INTERVAL '1 MONTH'):: DATE AS month,
  GENERATE_SERIES (0, 3) AS month_number
FROM data_bank.customer_transactions
),

monthly_transactions AS(
SELECT
  m.customer_id,
  m.month,
  m.month_number,
  COALESCE(b.balance, 0) AS transaction_amount
FROM months AS m
INNER JOIN balance AS b
  ON m.customer_id = b.customer_id
  AND m.month = b.month
),

monthly_aggregates AS(
SELECT
  customer_id,
  month_number,
  LAG(transaction_amount) OVER (PARTITION BY customer_id ORDER BY month) AS previous_month_transaction
FROM monthly_transactions
),

calculations AS(
SELECT
  COUNT(DISTINCT m.customer_id) AS customer_count,
  SUM(CASE WHEN a.previous_month_transaction > 0 THEN 1 ELSE 0 END) AS positive_first_month,
  SUM(CASE WHEN a.previous_month_transaction < 0 THEN 1 ELSE 0 END) AS negative_first_month,
  SUM(CASE 
      WHEN a.previous_month_transaction > 0
      AND m.transaction_amount > 0
      AND m.transaction_amount > 1.05 * a.previous_month_transaction
      THEN 1 ELSE 0 END) AS increase_count,
  SUM(CASE 
        WHEN m.transaction_amount > 0
        AND a.previous_month_transaction > 0
        AND m.transaction_amount < 1.05 * a.previous_month_transaction THEN 1 ELSE 0 END) AS decrease_count,
  SUM(CASE WHEN m.transaction_amount < 0 AND a.previous_month_transaction > 0 THEN 1 ELSE 0 END) AS negative_bal
FROM monthly_aggregates AS a
INNER JOIN monthly_transactions AS m
ON a.customer_id = m.customer_id
AND a.month_number = m.month_number
WHERE a.previous_month_transaction IS NOT NULL
AND a.month_number IN (0,1)
)

SELECT
  ROUND(100 * positive_first_month/ customer_count, 2) AS positive_1st_month_percent,
  ROUND(100 * negative_first_month/ customer_count, 2) AS negative_1st_month_percent,
  ROUND(100 * increase_count / customer_count, 2) AS increase_percent,
  ROUND(100 * decrease_count / customer_count, 2) AS decrease_percent,
  ROUND(100 * negative_bal/ customer_count, 2) AS negative_bal_percent
FROM calculations;
```

**Output**

positive_1st_month_percent  | negative_1st_month_percent  | increase_percent  | decrease_percent  | negative_bal_percent
--  | --  | --  | --  | --  
66.00  | 33.00  | 13.00 | 14.00 | 38.00





