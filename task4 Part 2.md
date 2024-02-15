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

## Improved Query
```
select *

from fct_SampleSuperstore_Purchase

where type like '%Consumer' and extract (year from Purchase_DATEID) = 2022 and 
extract (day from Purchase_DATEID) = 1;
```

## Justification
The improved query was modified to accomplish the following: 
- Instead of specifying all consumer types for selection, it employed a wildcard search ("%consumer"). 
- Instead of explicitly stating the first day of every month in 2022, it derived the day from the date and exclusively picked records 
  where the day aligned with the first day of each month in that year.

