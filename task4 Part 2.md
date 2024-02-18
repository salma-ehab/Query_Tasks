## Part2

## Original Query
```
SELECT
"Purchase_Transaction_ID",
"Purchase_DATEID",
"Ship Mode",
"Segment",
"Region",
"Category",
"Sub-Category",
"Type",
"Quantity",
"Discount",
"Profit"
FROM fct_SampleSuperstore_Purchase
WHERE
"Type" IN ('Second Class - Consumer', 'Standard Class - Consumer', 'First Class - Consumer', 'Same Day - Consumer')
and Purchase_DATEID IN 
('2022-12-01',
'2022-11-01',
'2022-01-01',
'2022-10-01',
'2022-06-01',
'2022-04-01',
'2022-09-01',
'2022-02-01',
'2022-07-01',
'2022-05-01',
'2022-08-01',
'2022-03-01')
```

## Improved Query 1

```
with 

purchase_CTE as

(
    select 

    Purchase_Transaction_ID,
    Purchase_DATEID,
    extract (year from Purchase_DATEID) as Purchase_Year,
    extract (month from Purchase_DATEID) as Purchase_Month,
    extract (day from Purchase_DATEID) as Purchase_Day,
    Ship_Mode,
    Region,
    Category,
    SubCategory,
    Sales,
    Quantity,
    Discount,
    Profit,
    split_part(Type, '- ', 2) as Type

    from fct_SampleSuperstore_Purchase

)

select *

from purchase_CTE

where Type = 'Consumer' and Purchase_Year = 2022 and Purchase_Day = 1;
```

## Improved Query 2
```

    alter table fct_SampleSuperstore_Purchase

    add 
    
    column Purchase_Year int,
    column Purchase_Month int,
    column Purchase_Day int;
```

```
    update fct_SampleSuperstore_Purchase

    set 

    Purchase_Year = extract(year from Purchase_DATEID),
    Purchase_Month = extract(month from Purchase_DATEID),
    Purchase_Day = extract(day from Purchase_DATEID),
    Type = split_part(Type, '- ', 2); 
```

```
    select *

    from fct_SampleSuperstore_Purchase

    where Type = 'Consumer' and Purchase_Year = 2022 and Purchase_Day = 1;
```

## Justification
- The type had to be split into two parts: "shipment type" and "customer type". 
  Since "shipment type" already exists as shipping mode, the type was adjusted from encompassing both shipment 
  and customer types to solely representing customer type. This adjustment allows for more flexible filtering in future queries.

- Similarly, regarding date handling, it's beneficial to split it from the beginning into day, month, and year parts. 

- These alterations were implemented in two ways. 
  - One method involved using a Common Table Expression (CTE). If executed in a tool like dbt, 
    this CTE would typically be stored in a staging folder and used to create another table or view for querying. 
    
  - Second method involved adjusting the table directly by adding additional columns and updating the values.

