## Part1

## Dataset:

Table: sales
Columns: product_id, customer_id, sale_date, amount, quantity


## 1. Identify the most popular products in terms of sales quantity during a specific timeframe.
### Finding top N products sold within a specific date:

## Original Query
```
SELECT *
FROM sales
WHERE sale_date BETWEEN '2023-01-01' AND '2023-02-07'
ORDER BY quantity DESC
LIMIT 10;
```

## Improved Query
```
with 

ranked_products as
(
  select 

  product_id, 
  sum(quantity) as total_quantity,
  dense_rank() over (order by total_quantity desc) as product_rank

  from sales

  where sale_date between '2023-01-01' and '2023-02-07'

  group by product_id
) 

select 

product_id,
total_quantity,
product_rank

from ranked_products

where product_rank < 11;
```

## Justification
- Identifying the top N products sold within a specified date range was accomplished using dense ranking.

- Dense ranking was selected to ensure that products with the same total quantities aren't missed. 
  It fills gaps and gives a clear list of the top N products, eliminating any risk of missing data in determining popularity.

## 2. Calculating cumulative sales by product category:

## Original Query
```
SELECT product_category, total_sales
FROM sales
ORDER BY product_category, sale_date
```

## Improved Query
```
with 

category_amount_per_day as
(
  select

  product_category,
  sale_date,
  sum(amount) as amount_per_day

  from sales

  group by product_category, sale_date

  order by product_category, sale_date
)

select 

product_category, 
sale_date,
sum(amount_per_day) over (partition by product_category order by sale_date) as total_sales

from category_amount_per_day;
```

## Justification
- Since the date is at the daily granularity, the initial step was to determine the total amount spent per category per day. 
  Following this, cumulative sales were calculated for each category.

## 3. Identifying customers with the highest average order value within a region:

## Original Query
```
SELECT customer_id, region, amount
FROM sales
```

## Improved Query
```
with 

customer_avg_amount_per_region as
(
  select 
  
  customer_id,
  region,
  round(avg(amount),3) as avg_amount,
  dense_rank() over (partition by region order by avg_amount desc) as customer_rank

  from sales

  group by customer_id, region
  
  order by region, customer_id
)

select 

customer_id,
region,
avg_amount

from customer_avg_amount_per_region

where customer_rank = 1;
```

## Justification
- Identifying customers with the highest average order value per region was achieved through the utilization of dense ranking.

- Dense ranking was chosen to ensure that no customers with identical highest average order values per region would be overlooked, 
  thus mitigating any potential data gaps or risks of omission.

## 4. Finding products with the biggest year-over-year sales growth:

## Original Query
```
SELECT product_id, amount, sale_date
FROM sales;
```
- Identifying products with the most significant year-over-year sales growth can be interpreted as finding products 
  with substantial increases in sales either annually or cumulatively across all years. 
  
- To achieve both, we can employ two methods: performing a self-join on the "product_years_revenue" table using aliases to 
  establish a connection between the current year (yr) and the previous year (pyr), or utilizing the lag function. 
  This will result in four enhanced queries.

## Improved Query 1 (Finding products with the biggest year-over-year sales growth annually using self-join)
```
with

product_years_revenue as
(
  select

  product_id,
  extract(year from sale_date) as year_value,
  sum(amount) as year_amount

  from sales

  group by product_id, year_value

  order by year_value, product_id
),

ranked_products as
(
  select 

  yr.product_id,
  yr.year_value,
  yr.year_amount,
  pyr.year_value as prev_year,
  pyr.year_amount as prev_year_amount,

  case when pyr.year_amount is null
  then null
  else yr.year_amount - pyr.year_amount 
  end as product_year_over_year_diff_value,

  case when pyr.year_amount is null
  then null
  else cast(round(((yr.year_amount - pyr.year_amount) /pyr.year_amount) * 100,3) as varchar) || '%'
  end as product_year_over_year_diff_perc,
 
  case when pyr.year_amount is null
  then null
  else dense_rank() over (partition by yr.year_value 
  order by cast(replace(product_year_over_year_diff_perc,'%','')as decimal(10,2)) desc nulls last) 
  end as proudct_growth_rate_rank

  from product_years_revenue yr
  left join product_years_revenue pyr on
  yr.year_value = pyr.year_value + 1
  and yr.product_id = pyr.product_id

  order by year_value desc, product_id
)

select 

product_id,
year_value,
year_amount,
prev_year,
prev_year_amount,
product_year_over_year_diff_value,
product_year_over_year_diff_perc

from ranked_products

where proudct_growth_rate_rank = 1

order by year_value desc;
```

## Justification
- This query is designed to identify products with the biggest year-over-year sales growth on an annual basis, achieved through a self-join 
  operation on the "product_years_revenue" table using aliases. 
  
- This establishes a connection between the current year (yr) and the previous year (pyr). Subsequently, it calculates both the 
  difference and percentage difference in year-over-year revenue. 
  
- However, in cases where the revenue amount for the previous year is null, the resulting year-over-year difference values and percentages 
  will also be null. 
  
- To rank these products, dense ranking is employed partitioned by "year_value" to ensure that products exhibiting identical highest growth rates are not overlooked. 
  This approach eliminates any potential data omission risks. If the growth rate is null, the corresponding ranking will also be null, 
  and the dense ranking methodology positions null values at the end.


## Improved Query 2 (Finding top 10 products with the biggest year-over-year sales growth across all years using self-join)
```
with

product_years_revenue as
(
  select

  product_id,
  extract(year from sale_date) as year_value,
  sum(amount) as year_amount

  from sales

  group by product_id, year_value

  order by year_value, product_id
),

ranked_products as
(
  select 

  yr.product_id,
  yr.year_value,
  yr.year_amount,
  pyr.year_value as prev_year,
  pyr.year_amount as prev_year_amount,

  case when pyr.year_amount is null
  then null
  else yr.year_amount - pyr.year_amount 
  end as product_year_over_year_diff_value,

  case when pyr.year_amount is null
  then null
  else cast(round(((yr.year_amount - pyr.year_amount) /pyr.year_amount) * 100,3) as varchar) || '%'
  end as product_year_over_year_diff_perc,
 
  case when pyr.year_amount is null
  then null
  else dense_rank() over (order by cast(replace(product_year_over_year_diff_perc,'%','')as decimal(10,2)) desc nulls last) 
  end as proudct_growth_rate_rank

  from product_years_revenue yr
  left join product_years_revenue pyr on
  yr.year_value = pyr.year_value + 1
  and yr.product_id = pyr.product_id

  order by year_value desc, product_id
)

select 

product_id,
year_value,
year_amount,
prev_year,
prev_year_amount,
product_year_over_year_diff_value,
product_year_over_year_diff_perc,
proudct_growth_rate_rank

from ranked_products

where proudct_growth_rate_rank < 11

order by proudct_growth_rate_rank;
```

## Justification
- This query is designed to identify products with the biggest year-over-year sales growth across all years, achieved through a self-join 
  operation on the "product_years_revenue" table using aliases. 
  
- This establishes a connection between the current year (yr) and the previous year (pyr). Subsequently, it calculates both the 
  difference and percentage difference in year-over-year revenue. 
  
- However, in cases where the revenue amount for the previous year is null, the resulting year-over-year difference values and percentages 
  will also be null. 
  
- To rank these products, dense ranking is employed to ensure that products exhibiting identical highest growth rates are not overlooked. 
  This approach eliminates any potential data omission risks. If the growth rate is null, the corresponding ranking will also be null, 
  and the dense ranking methodology positions null values at the end.


## Improved Query 3 (Finding products with the biggest year-over-year sales growth annually using lag window function)
```
with

product_years_revenue as
(
  select

  product_id,
  extract(year from sale_date) as year_value,
  sum(amount) as year_amount

  from sales

  group by product_id, year_value

  order by year_value, product_id
),

products_growth_rates as
(
  select 

  product_id,
  year_value,
  year_amount,
  
  case when lag(year_value, 1) over (partition by product_id order by year_value) != year_value - 1
  then null
  else lag(year_value, 1) over (partition by product_id order by year_value)
  end as prev_year,
  
  case when prev_year is null
  then null
  else
  lag(year_amount, 1) over (partition by product_id order by year_value) 
  end as prev_year_amount,


  case when prev_year_amount is null
  then null
  else year_amount - prev_year_amount
  end as product_year_over_year_diff_value,

  case when prev_year_amount is null
  then null
  else cast(round(((year_amount - prev_year_amount) /prev_year_amount) * 100,3) as varchar) || '%'
  end as product_year_over_year_diff_perc

  from product_years_revenue 

  order by year_value desc, product_id
),

products_ranked as
(
    select 

    product_id,
    year_value,
    year_amount,
    prev_year,
    prev_year_amount,
    product_year_over_year_diff_value,
    product_year_over_year_diff_perc,

    case when prev_year_amount is null
    then null
    else dense_rank() over (partition by year_value 
    order by cast(replace(product_year_over_year_diff_perc,'%','')as decimal(10,2)) desc nulls last) 
    end as proudct_growth_rate_rank

    from products_growth_rates

    order by year_value desc, product_id
)

select 

product_id,
year_value,
year_amount,
prev_year,
prev_year_amount,
product_year_over_year_diff_value,
product_year_over_year_diff_perc

from products_ranked

where proudct_growth_rate_rank = 1

order by year_value desc;
```

## Justification
- This query is designed to identify products with the biggest year-over-year sales growth on an annual basis, achieved through using
  lag window function.

- When the previous year for a specific product does not correspond to the current year minus one, 
  the previous year will be marked as null, and consequently, the revenue amount for that year along with
  the resulting year-over-year difference values and percentages will also be null.
  
- To rank these products, dense ranking is employed partitioned by "year_value" to ensure that products exhibiting identical highest growth rates are not overlooked. 
  This approach eliminates any potential data omission risks. If the growth rate is null, the corresponding ranking will also be null, 
  and the dense ranking methodology positions null values at the end.

## Improved Query 4 (Finding top 10 products with the biggest year-over-year sales growth across all years using lag window function)
```
with

product_years_revenue as
(
  select

  product_id,
  extract(year from sale_date) as year_value,
  sum(amount) as year_amount

  from sales

  group by product_id, year_value

  order by year_value, product_id
),

products_growth_rates as
(
  select 

  product_id,
  year_value,
  year_amount,
  
  case when lag(year_value, 1) over (partition by product_id order by year_value) != year_value - 1
  then null
  else lag(year_value, 1) over (partition by product_id order by year_value)
  end as prev_year,
  
  case when prev_year is null
  then null
  else
  lag(year_amount, 1) over (partition by product_id order by year_value) 
  end as prev_year_amount,


  case when prev_year_amount is null
  then null
  else year_amount - prev_year_amount
  end as product_year_over_year_diff_value,

  case when prev_year_amount is null
  then null
  else cast(round(((year_amount - prev_year_amount) /prev_year_amount) * 100,3) as varchar) || '%'
  end as product_year_over_year_diff_perc

  from product_years_revenue 

  order by year_value desc, product_id
),

products_ranked as
(
    select 

    product_id,
    year_value,
    year_amount,
    prev_year,
    prev_year_amount,
    product_year_over_year_diff_value,
    product_year_over_year_diff_perc,

    case when prev_year_amount is null
    then null
    else dense_rank() over (order by cast(replace(product_year_over_year_diff_perc,'%','')as decimal(10,2)) desc nulls last) 
    end as proudct_growth_rate_rank

    from products_growth_rates

    order by year_value desc, product_id
)

select 

product_id,
year_value,
year_amount,
prev_year,
prev_year_amount,
product_year_over_year_diff_value,
product_year_over_year_diff_perc,
proudct_growth_rate_rank

from products_ranked

where proudct_growth_rate_rank < 11

order by proudct_growth_rate_rank;
```

## Justification
- This query is designed to identify products with the biggest year-over-year sales growth across all years, achieved through using
  lag window function.

- When the previous year for a specific product does not correspond to the current year minus one, 
  the previous year will be marked as null, and consequently, the revenue amount for that year along with
  the resulting year-over-year difference values and percentages will also be null.
  
- To rank these products, dense ranking is employed to ensure that products exhibiting identical highest growth rates are not overlooked. 
  This approach eliminates any potential data omission risks. If the growth rate is null, the corresponding ranking will also be null, 
  and the dense ranking methodology positions null values at the end.
--------------------------------------------------------------------------------------------------------------------------------------
--------------------------------

## Part 2: Those queries have errors. Please identify the errors and write the correct queries.
### Here are some examples of queries with errors related to window functions:

## 1.
## Original Query
```
SELECT *, RANK() OVER (PARTITION BY customer_id) AS rank
FROM sales
ORDER BY rank DESC;
```

## Improved Query
```
select

*,
dense_rank() over(partition by customer_id order by amount desc) as rank

from sales

order by customer_id, rank;
```

## Justification
- The correction includes adjusting the RANK() window function to include an ORDER BY clause, 
  specifying the amount column to rank sales within each partition of "customer_id". 
  
- Moreover, the RANK() function has been replaced with DENSE_RANK() to ensure contiguous ranking without any gaps.

## 2.
## Original Query
```
SELECT * , SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM sales;
```

## Improved Query
```
with 

customer_sales_per_day as
(
  select

  customer_id,
  sale_date,
  sum(amount) as amount_per_day

  from sales

  group by customer_id, sale_date

  order by customer_id, sale_date
)

select 

customer_id, 
sale_date, 

sum(amount_per_day) over (partition by customer_id order by sale_date) as  customer_running_total

from customer_sales_per_day;
```

## Justification
- The correction involves modifying the SUM() window function to incorporate a PARTITION BY clause. 
  Initially, the aim was to compute the cumulative amount spent by each customer. 
  
- Due to the granularity of the date being at the daily level, the total amount spent per customer per day was computed first. 

- However, the inability to join this data with the entire table stems from the absence of a unique identifier per order. 
  Had there been an order_id, we could have included a running total along with all the sales transactions. 
  
## 3.
## Original Query
```
SELECT AVG(amount) OVER (PARTITION BY customer_id) AS avg_amunt,
       COUNT(*) FROM sales;
```

## Improved Query
```
select distinct

customer_id,
avg(amount) over (partition by customer_id) as avg_customer_amunt,
count(*) over (partition by customer_id) as customer_transactions_count

from sales

order by customer_id;
```

## Justification
- The correction entails adjusting the window function COUNT() to include the OVER PARTITION BY clause. 

- Given that the goal of this query was to compute the average amount spent by each customer, it was deemed necessary to tally 
  the number of transactions per customer while also including the customer ID.

## 4.
## Original Query
```
SELECT *, ROW_NUMBER() OVER (ORDER BY sale_date) AS row_num,
          ROW_NUMBER() OVER (ORDER BY amount) AS another_row_num
FROM sales;
```

## Improved Query
```
select 

*, 
dense_rank() over (order by sale_date) as row_num,
dense_rank() over (order by amount) as another_row_num

from sales;
```

## Justification
- The correction entails substituting the window function ROW_NUMBER() with DENSE_RANK() to 
  ensure that rows with identical values are assigned the same rank, without any gaps in the ranking sequence.







