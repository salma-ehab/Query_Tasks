Here you will find some use cases and SQL code. 
- Analyze the use cases and find more efficient code to enhance the logic.
- Include the improved SQL code for your choice, along with an explanation of why it is better.


# Example 1: Identify the salesperson with the highest sales for each quarter of the year for the past two years.

## Original Query
```
SELECT
    sp.salesperson_id,
    q.quarter_name,
    SUM(t.order_amount) AS total_sales
FROM sales_transactions AS t
JOIN salespeople AS sp ON t.salesperson_id = sp.salesperson_id
JOIN quarters AS q ON t.transaction_date BETWEEN q.start_date AND q.end_date
GROUP BY
    sp.salesperson_id,
    q.quarter_name
```

## Improved Query
```
with 

ranked_sales as
(
  select 

  extract (year from t.transaction_date) as year_value,
  q.quarter_name,
  t.salesperson_id,
  sum(t.order_amount) as total_sales,
  rank() over (partition by q.quarter_name, year_value order by sum(t.order_amount) desc) as sales_rank

  from sales_transactions as t
  join quarters as q on t.transaction_date between q.start_date and q.end_date

  where year_value >= year(getdate()) - 2 and year_value < year(getdate())

  group by year_value, q.quarter_name, t.salesperson_id
)

  select 

  year_value,
  quarter_name,
  salesperson_id,
  total_sales
  
  from ranked_sales
  
  where sales_rank = 1

  order by year_value, quarter_name;
```

## Justification
- The improved query eliminates an unnecessary join with the "salespeople" table by recognizing 
  that only the salesperson's ID is required, and this information is already available in the "sales_transactions" table. 

- Furthermore, the improved query effectively identifies the salesperson with the highest sales for 
  each quarter throughout the past two years (2022 and 2023), considering the present year has recently begun.
  
- In cases where multiple salespersons attain the same maximum sales for a quarter, all of them will be included in the results due to the utilization of ranking.


# Example 2: Identifying the top 5 salespeople based on total sales for the year:

## Original Query
```
SELECT
    sp.salesperson_id,
    SUM(t.order_amount) AS total_sales
FROM sales_transactions AS t
JOIN salespeople AS sp ON t.salesperson_id = sp.salesperson_id
GROUP BY
    sp.salesperson_id
ORDER BY total_sales DESC
LIMIT 5
```

## Improved Query
```
select

t.salesperson_id,
sum(t.order_amount) as total_sales

from sales_transactions as t

where extract(year from t.transaction_date) = extract(year from current_date) - 1

group by t.salesperson_id
order by total_sales desc
limit 5;
```

## Justification
- The improved query eliminates an unnecessary join with the "salespeople" table by recognizing 
  that only the salesperson's ID is required, and this information is already available in the "sales_transactions" table. 

- Moreover, the improved query effectively identifies the top 5 salespeople based on total sales for the year,
  considering that the current year has not concluded; the reference year in this query pertains to the preceding year, which is 2023.


# Example 3: Return a list of all the employees in those departments.

## Original Query
```
SELECT 
  first_name,
  last_name
FROM employee e1
WHERE department_id IN (
   SELECT department_id
   FROM department
   WHERE manager_name=‘John Smith’)
```

## Improved Query
```
select 

e1.first_name,
e1.last_name

from employee e1
join department d1 on e1.department_id = d1.department_id

where d1.manager_name='John Smith';
```

## Justification
- The improved query replaces the original subquery with a JOIN operation, aiming to improve efficiency.
  By directly connecting the employee and department tables based on the department_id, this approach 
  eliminates the need for a separate subquery, potentially making the query faster.


# Example 4: Find Duplicate Rows

## Original Query
```
SELECT 
  employee_id,
  last_name,
  first_name,
  dept_id,
  manager_id,
  salary
FROM employee
GROUP BY   
  employee_id,
  last_name,
  first_name,
  dept_id,
  manager_id,
  salary
HAVING COUNT(*) > 1
```

## Improved Query
```
select 

e1.employee_id,
e1.last_name,
e1.first_name,
e1.dept_id,
d1.manager_id,
e1.salary

from employee e1
join department d1 on e1.dept_id = d1.dept_id

group by e1.employee_id, e1.last_name, e1.first_name, e1.dept_id, d1.manager_id, e1.salary
having count(*) > 1;
```

## Justification
- This is the most effective method for detecting duplicate rows, unless the objective is 
  to specifically identify duplicate employees. In such cases, we will solely focus on selecting and grouping by employee_id.

- Assuming the manager ID is present in the department table, as the query before implied that manager data 
  is stored alongside department information. Consequently, a straightforward join was sufficient to obtain the manager ID.
  

# Example 5: Compute a Year-Over-Year Difference
-  The main query should return the year, year_amount, revenue_previous_year, Year-Over-Year_diff_value, and Year-Over-Year_diff_perc(%) from the year_metrics

## Original Query
```
SELECT
    extract(year from day) as year,
    SUM(daily_amount) as year_amount,
  FROM sales
  GROUP BY year
```

## Improved Query
```
with

years_revenue as
(
  select

  extract(year from day) as year_value,
  sum(daily_amount) as year_amount

  from sales
  group by year_value
)

select 

yr.year_value,
yr.year_amount,
pyr.year_value as prev_year,
pyr.year_amount as prev_year_amount,

case when pyr.year_amount is null
then null
else yr.year_amount - pyr.year_amount 
end as year_over_year_diff_value,

case when pyr.year_amount is null
then null
else cast(round(((yr.year_amount - pyr.year_amount) /pyr.year_amount) * 100,3) as varchar) || '%'
end as year_over_year_diff_perc

from years_revenue yr
left join years_revenue pyr on
yr.year = pyr.year + 1

order by year_value desc;
```

## Justification
- The improved query performs a self-join on the "Years_Revenue" table using aliases, 
  creating a link between the current year (yr) and the previous year (pyr). 
  It then computes the Year-Over-Year difference in both value and percentage. 
  However, if the revenue amount for the previous year is null, the Year-Over-Year difference values 
  and percentages will also be null.


# Example 6: Return the MAX sales and 2nd max sales 

## Original Query
```
SELECT
(SELECT MAX(sales) FROM Sales_orders) max_sales,
(SELECT MAX(sales) FROM Sales_orders
WHERE sales NOT IN (SELECT MAX(sales) FROM Sales_orders )) as 2ND_max_sales;
```

## Improved Query 1
```
with

maximum_2 as
( 
  select distinct sales
  from Sales_orders
  order by sales desc
  limit 2
)

select 
    
max_1.sales as max_sales, 
max_2.sales as second_max_sales

from maximum_2 max_1
join maximum_2 max_2
on max_1.sales > max_2.sales;
```

## Justification
- The improved query likely performs better because it eliminates the need for nested subqueries. 
  Subqueries can sometimes be less efficient than joins, and by using a small join on a precomputed 
  result set, the query may execute more quickly.

## Improved Query 2
```
with 

ranked_sales as
(
  select

  sales,
  dense_rank() over (order by sales desc) as sales_rank

  from Sales_orders
)

select distinct

rs1.sales as max_sales,
rs2.sales as second_max_sales

from ranked_sales rs1
cross join ranked_sales rs2

where rs1.sales_rank = 1 and rs2.sales_rank = 2;
```

## Justification
- The improved query is designed to enhance performance, especially in situations demanding more than two highest sales. 
  It utilizes the DENSE_RANK() function, assigning a distinct rank to each row in the result set. 
  When multiple rows have identical sales values, they receive the same rank, and the subsequent rank is incremented.


# Example 7: 
- This query will select the Essns of all employees who work the same
(project, hours) combination on some project that employee ‘John
Smith’ (whose Ssn =‘123456789’) works on. I

## Original Query
```
SELECT DISTINCT Pnumber
FROM PROJECT
WHERE Pnumber IN  
( SELECT Pnumber
FROM PROJECT, DEPARTMENT, EMPLOYEE
WHERE Dnum=Dnumber AND
Mgr_ssn=Ssn AND Lname=‘Smith’ )
OR
Pnumber IN
( SELECT Pno
FROM WORKS_ON, EMPLOYEE
WHERE Essn=Ssn AND Lname=‘Smith’ )
```

## Query 1
- This query will select the Essns of all employees who work the same
  (project, hours) combination on some project that employee ‘John
  Smith’ (whose Ssn =‘123456789’) works on.

## Improved Query 1
```
with

john_pno_hours as
(
  select pno, hours
  from works_on
  where essn = '123456789'
)

select distinct works_on.essn

from works_on join john_pno_hours on
works_on.pno =  john_pno_hours.pno 
and  works_on.hours =  john_pno_hours.hours

where works_on.essn <> '123456789';
```

## Justification
- The objective of this query was to retrieve the Social Security Numbers (SSNs) of all employees 
  involved in a project with the same (project, hours) combination as the employee 'John Smith' (SSN = '123456789').
  Given the uniqueness of each SSN, there was no necessity to filter by name. Additionally, as the SSN is already present in the 
  'works_on' table, there was no need to link with the 'employee' table. 
  When joining the data, John's SSN was filtered out, and the focus was on identifying other employees.

- In cases where John worked on multiple projects with the same hours, and there were other employees matching these project 
  and hour combinations, the use of DISTINCT was implemented. This ensured that duplicate entries were eliminated.

- To optimize the query, inner joins were employed instead of including all the join conditions in the WHERE clause. 

## Query 2
- This query will select the Essns of all employees who work the same
(project) as employee ‘John Smith’.

## Improved Query 2
```
with

john_pno_hours as
(
  select pnumber,ssn
  from project
  join department on dnum=dnumber
  join employee on mgr_ssn=ssn
  where fname='John' and lname='Smith' 

  union

  select pno,ssn
  from works_on
  join employee on essn=ssn
  where fname='John' and lname='Smith' 
)

select distinct essn

from works_on join john_pno_hours on 
works_on.pno =  john_pno_hours.pnumber

where works_on.essn <> john_pno_hours.ssn;
```

## Justification
- The query aimed to retrieve the Social Security Numbers (SSNs) of employees who are either working on the same project as 
  'John Smith' or are under the management of 'John Smith'.

-  When joining the data, John's SSN was filtered out, and the focus was on identifying other employees.

- To optimize the query, inner joins were employed instead of including all the join conditions in the WHERE clause. 

## Justification behind the implementation of two different queries 
- The initial query is designed to obtain the desired result of all employees sharing the same (project, hours) 
  combination on a project where employee 'John Smith' (with SSN '123456789') is involved. On the other hand, 
  the second query is an optimization of the original query provided. The original query sought to gather SSNs of 
  employees either working on the same project as 'John Smith' or being managed by 'John Smith'.
  Due to the distinct use cases, two separate queries were formulated to address the specific requirements appropriately.