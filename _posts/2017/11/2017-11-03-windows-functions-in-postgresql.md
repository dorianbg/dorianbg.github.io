---
title: "Windows functions in PostgresQL"
date: "2017-11-03"
categories: 
  - "data-engineering"
  - "sql-server"
  - "t-sql"
---

## 1. Setting up postgresql on Mac OS

Install postgresql:

```
brew install postgresql 
```

Start postgres:

```
brew services start postgresql
```

Login to postgres shell to create user:

```
/usr/local/bin/psql -d postgres
```

Create the user:

```
CREATE USER user PASSWORD ‘password';
```

If you use a SQL client, I recommend dBeaver, you can now easily connect to the database.

## 2. Preparing our dataset (DDL, DML)

Defining our table:

```none
create table topup_data (
    user_id int not null, 
    date date not null, 
    top_up_value int not null default '0'  
)
```

Populating the table:

```none
insert into topup_data values 
(1,'2017-06-20',15),
(1,'2017-05-22',10),
(1,'2017-04-18',20),
(1,'2017-03-20',20),
(1,'2017-02-20',15),
(2,'2017-06-20',5),
(2,'2017-06-05',10),
(2,'2017-04-22',10),
(2,'2017-03-30',10),
(2,'2017-03-15',15),
(2,'2017-02-10',10)
```

## 3. Basic window function queries

#### Task 1

Using window function row_number() we can find the 5 most recent top-ups per user:

```none
with temp as (
select 
    user_id, 
    date,
    top_up_value, 
    row_number() over (partition by user_id order by date DESC) as row_n
from 
    topup_data
)
select 
    *
from 
    temp
where 
    row_n <= 5
```

#### Task 2

Using window functions rank() and dense_rank() we can also find the 5 largest top ups per user:

```none
with temp as (
    select 
        user_id, 
        date,
        top_up_value, 
        rank() over (partition by user_id order by top_up_value DESC) as row_n
    from 
        topup_data
)
select 
    *
from 
    temp
where 
    row_n <= 5
```

Compared to row_number(), the rank() function assigned the same rank to identical values while possibly skipping rank values after it assigned multiple ranks to just one value. Eg. 1 -> 2 -> 2 -> 4 -> 5.

If we used dense_rank() instead of rank() then we will not skip rank values in the identical rows. Eg. 1 -> 2 -> 2 -> 3 -> 4.

#### Task 3

We can also do inter-row calculations to enrich the original dataset to include extra columns "previous_top-up_date" and "days_since_previous_top_up". We will use the lag() and lead() window functions for this task.

We could easily create a new table schema as shown in task 2, but we can also create a new table from the result of table as shown below:

```none
with temp1 as (
    select 
        user_id, 
        date,
        top_up_value, 
        lead(date,1) over (partition by user_id order by date desc) as previous_top_up_date
    from 
        topup_data
), 
temp2 as (
    select 
        * , 
        date - previous_top_up_date as days_since_previous_top_up
    from 
        temp1
)
select 
    *
into 
    enriched_topup_data
from 
    temp2
```

Please note that we could have also used the lag() window function in the last query.

## Conclusion

Window functions are the bread and butter of analytical queries since they allow complex queries using very simple syntax.
