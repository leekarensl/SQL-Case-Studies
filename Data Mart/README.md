
## The business questions and my SQL solutions:

### Part A Data Cleansing Steps
**1. Generate a new table and call it clean_weekly_sales, convert the week_date to a date format, add a week_number, a month_number, a calendar_year column and a new column called age_band whwere segment 1 is Young Adults, segment 2 is Middle Aged and 3/4 are Retirees. If the segment contains C is its Couples and F is Families for a new demographic column. Also to create a new column called avg_transaction, rounded to 2 decimal places**

```sql
drop table if exists data_mart.clean_weekly_sales;
create table data_mart.clean_weekly_sales as
select
  TO_DATE(week_date, 'DD/MM/YY') as week_date,
  DATE_PART('week', TO_DATE(week_date, 'DD/MM/YY')) as week_number,
  DATE_PART('month', TO_DATE(week_date, 'DD/MM/YY')) as month_number,
  DATE_PART('year', TO_DATE(week_date, 'DD/MM/YY')) as calendar_year,
  region,
  platform,
  case
    when segment = 'null' then 'Unknown'
  else segment end as segment,
  case
    when segment like '%1' then 'Young Adults'
    when segment like '%2' then 'Middle Aged'
    when segment like '%3' or segment like '%4' then 'Retirees' 
    else 'Unknown'
  end as age_band,
  case
    when left(segment,1) = 'C' then 'Couples'
    when left(segment,1) = 'F' then 'Families'
  else 'Unknown' end as demographic,
  customer_type,
  transactions,
  sales,
  round(sales/transactions,2) as avg_transaction
from data_mart.weekly_sales
;
```

### Part B Data Exploration
**1. What day of the week is used for each week_date value?**

```sql
select distinct to_char(week_date, 'DAY') as weekday
from data_mart.clean_weekly_sales;
```

**Output**

weekday |
----  |
MONDAY |

**2. What range of week numbers are missing from the dataset?**

```sql
with all_week_numbers as(
select generate_series(1,52) as week_number
)

select
  week_number
from all_week_numbers t1
where not exists(
 select 1
 from data_mart.clean_weekly_sales t2
 where t2.week_number = t1.week_number
 )
 ;
```
**Output**

week_number |
----  |
1 |
2 |
...|
11 |
12|
37|
38|
...|
52|

**3. How many total transactions were there for each year?**

```sql
select
   calendar_year,
   sum(transactions) as total_transactions
from data_mart.clean_weekly_sales
group by 1
order by 1;
```
**Output**

calendar_year | total_transactions
--- | ----
2018 | 346406460
2019 | 365639285
2020 | 375813651

**4. What is the total sales for each region for each month?**

```sql
select
  distinct date_trunc('month', week_date) as calendar_month,
  region,
  sum(sales) as total_sales
from data_mart.clean_weekly_sales
group by 1,2
order by 1,2;
```
**Output**

calendar_month | region | total_sales
---| ---| ----
2018-03-01 | CANADA | 33815571
2018-03-01  | EUROPE | 8402183
-- | --| --
2020-08-01  | USA | 277361606

**5. What is the total count of transactions for each platform**

```sql
select
  platform,
  sum(transactions) as total_transactions
from data_mart.clean_weekly_sales
group by 1
order by 1;
```
**Output**

platform | total_transactions
---| ---
Retail | 1081934227
Shopify | 5925169

**6. What is the percentage of sales for Retail vs Shopify for each month?**

```sql
with monthly_platform_sales as(
select
  date_trunc('month', week_date) as month,
  platform,
  sum(sales) as monthly_sales
from data_mart.clean_weekly_sales
group by 1,2
order by 1,2
)
,

monthly_total_sales as(
select
  month,
  SUM(case when platform = 'Retail' then monthly_sales end) as retail_sales,
  SUM(case when platform = 'Shopify' then monthly_sales end) as shopify_sales,
  SUM(monthly_sales) as total_sales
from monthly_platform_sales
group by 1
)

select
  month,
  round(100*coalesce(retail_sales, 0) / NULLIF(total_sales, 0),2) as retail_percentage,
  round(100*coalesce(shopify_sales, 0) / NULLIF(total_sales, 0),2) as shopify_percentage
from monthly_total_sales
order by 1;
```
**Output**

month | retail_percentage | shopify_percentage
-- | -- | --
2018-03-01 | 97.92 | 2.08
2018-04-01 | 97.93 | 2.07
-- | -- | --
2020-08-01 | 96.5 | 13.49

**7. What is the percentage of sales by demographic for each year in the dataset?**

```sql
with yearly_sales as(
select
  calendar_year as year,
  demographic,
  sum(sales) as sales
from data_mart.clean_weekly_sales
group by 1,2
)

select
  year,
  round(100 * couples_sales /total_sales,2) as pct_couples_sales,
  round(100 * families_sales/ total_sales,2) as pct_families_sales,
  round(100 * unknown_sales/ total_sales,2) as pct_unknown_sales
from(
select
  year,
  SUM(case when demographic = 'Couples' then sales end) as couples_sales,
  SUM(case when demographic = 'Families' then sales end) as families_sales,
  SUM(case when demographic = 'Unknown' then sales end) as unknown_sales,
  SUM(sales) as total_sales
from yearly_sales
group by 1
)
order by 1;
```
**Output**

year | pct_couples_sales | pct_families_sales  | pct_unknown_sales
-- | -- | -- | ---
2018 | 26.38  | 31.99  | 41.63
2019  | 27.28  | 32.47  | 40.25
2020  | 28.72 | 32.73  | 38.55


**8. What age-band and demographic values contribute most to Retail sales?**

```sql
-- age band
SELECT
  age_band,
  SUM(sales) AS total_sales,
  ROUND(100.0 * SUM(sales) / SUM(SUM(sales)) OVER (), 0) AS sales_percentage
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1
ORDER BY 3 DESC;
```
**Output**

age_band  | total_sales  | sales_percentage
--  |   --  | --
Unknown  | 16067285533 | 41
Retirees | 13005266930 | 33
Middle Aged  | 6208251884 | 16
Young Adults | 4373812090 | 11

```sql
-- demographic
SELECT
  demographic,
  SUM(sales) AS total_sales,
  ROUND(100.0 * SUM(sales) / SUM(SUM(sales)) OVER (), 0) AS sales_percentage
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1
ORDER BY 3 DESC;
```

**Output**

demographic  | total_sales  | sales_percentage
--  |   --  | --
Unknown  | 116067285533 | 41
Families | 12759667763 | 32
Couples | 10827663141 | 27













