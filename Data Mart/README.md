
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
    when right(segment,1) = 'C' then 'Couples'
    when right(segment,1) = 'F' then 'Families'
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
MONDAY    |





