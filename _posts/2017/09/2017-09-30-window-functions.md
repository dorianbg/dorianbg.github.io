---
title: "T-SQL Window functions syntax"
date: "2017-09-30"
categories: 
  - "data-engineering"
  - "sql-server"
  - "t-sql"
---

Window functions are an advanced and powerful feature of the T-SQL language. I will give a few tips on how to use and examples on the AdventureWorks2014 OLTP database.

Here I will give some notes on how to use them:

## 1. They can only be specified in the SELECT and ORDER BY clause of a SQL statement

This has important implications since we know that the logical processing of a query is in the following order:

FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY

Thus the input of the WINDOW FUNCTION is the result set after applying FROM, WHERE, GROUP BY and HAVING transformation.

This means that if there was the use of the WHERE clause of GROUP BY we will get different input for the window function that what is in the source table.

####  Example:

[code lang="sql"]

SELECT TOP 5 AdressLine1, ROW_NUMBER() OVER (ORDER BY AdressLine1) as [row_number] FROM [AdventureWorks2014].[Person].[Address] [/code]

While a similar query with where clause gives completely different result.

[code lang="sql"] SELECT TOP 5 AdressLine1, ROW_NUMBER() OVER (ORDER BY AdressLine1) as [row_number] FROM [AdventureWorks2014].[Person].[Address] WHERE city_name LIKE 'New York' [/code]

## 2. They are outlined by 2 main parts

<window function> OVER ( <window definition>) AS alias

1. Window function
    - There are 2 types of window functions
        1. Ranking
            - ROW_NUMBER()
            - RANK()
            - DENSE_RANK()
            - NTILE()
        2. Offset
            - LEAD(<col>)
            - LAG(<col>)
            - FIRST_VALUE(<col>)
            - LAST_VALUE(<col>)
        3. Aggregate window functions
            - Sum(col)
            - Avg(col)
            - Max(col)
            - ...
2. Window on which the function operates
    - specified in the OVER clause
    - defined in more detail below

## 3. The OVER clause has 3 parts

Before reading anything below, you must understand that the window function operates on the Result Set of the underlying query.

1. Partitioning
    - PARTITION BY clause
    - restricts the window on which the window function operates
    - acts like a where clause
    - defined on a column of the underlying result set
2. Ordering
    - ORDER BY clause
    - defines ordering in the window
    - defined on a column of the underlying result set
3. Framing
    - ROWS BETWEEN <above delimiter> AND <below delimiter> clause
    - defines how many rows from the CURRENT ROW will be in the window
    - example functions:
        - UNBOUNDED PRECEDING
        - CURRENT ROW
        - UNBOUNDED FOLLOWING

An example of a window function utilizing most of the options is:

####  Example:

This code will calculate for each item (salesOrderDetailId) in a large order (orderId) the item which was bought before with the maximum price.

Eg. I bought 3 items in one order

- 1 Fridge for 1000$
- 2 TV for 2000$
- 3 Phone for 500$

The result would be

- 1 1000$
- 2 2000$
- 3 2000$

Code:

[code lang="sql"] SELECT&amp;nbsp; salesOrderId, max(linetotal) OVER ( PARTITION BY salesOrderId&amp;nbsp; ORDER BY salesOrderDetailId ROWS BETWEEN unbounded preceding AND current row) AS average_line_total; FROM [AdventureWorks2014].[Sales].[SalesOrderDetail] [/code]
