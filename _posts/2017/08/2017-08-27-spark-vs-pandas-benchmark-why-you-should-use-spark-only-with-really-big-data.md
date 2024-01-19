---
title: "Spark vs Pandas benchmark: Why you should use Spark 2.1 only for really big data"
date: "2017-08-27"
categories: 
  - "data-engineering"
---

## Intro

Apache Spark is quickly becoming one of the best open source data analysis platforms.

With the continuous improvement of [Apache Spark](http://spark.apache.org/), especially the SQL engine and emergence of related projects such as Zeppelin notebooks we are starting to get the data analysis functionality we had on single machine setups using RDBMS and data analysis libraries like Pandas.

[Pandas](http://pandas.pydata.org/), a data analysis tools for the [Python](http://www.python.org/) programming language, is currently the most popular and mature open souce data analysis tool. The library is highly optimized for performance, with critical code paths written in [Cython](https://en.wikipedia.org/wiki/Cython "Cython") or [C](https://en.wikipedia.org/wiki/C_(programming_language) "C (programming language)").

This benchmark will compare the performance of those frameworks on common data analysis tasks:

1. Complex where clauses
2. Sorting a dataset
3. Joining a dataset
4. Self joins
5. Grouping the data
    

The tests will be performed on Dutch OpenData license plate dataset (found [here](https://opendata.rdw.nl/Voertuigen/Open-Data-RDW-Gekentekende_voertuigen/m9d7-ebf2)) which has ~13.5 million records, but I used command line tools to split it into files with 100k, 1m, 5m, 10m and ~13.5 lines. The according file sizes are: 49MB, 480MB, 2.4GB, 4.7GB and 6.5GB accordingly. To perform joins testing I made this small [dataset](https://github.com/dorianb96/apache-spark-vs-pandas-benchmark/blob/master/join_data.csv) with ~100 rows.

The underlying software is Python 3.5.3/pandas 0.19.2 and Scala 2.10/Spark 1.6.2 on a machine with 32GB RAM and 8 CPUs. The Spark local mode was done on the same machine with 32GB of RAM and 8 CPUs while the distributed mode was done in yarn-client mode using zeppelin noteboooks on a setup over 3 machines with the same specs. In the Spark local mode I utilized the normal file system and in the distributed mode I utilized HDFS.

You can find the Scala code for Spark [here](https://github.com/dorianb96/apache-spark-vs-pandas-benchmark/blob/master/spark_scala_code). (* the code is not a proper scala file, but it can be just copied into Zeppelin notebook cells.) Code for Pandas (in python) is [here](https://github.com/dorianb96/apache-spark-vs-pandas-benchmark/blob/master/Pandas_benchmark.ipynb).

## Test 1: Complex where clauses

Expectations:

This is a case where Spark should shine as it can concurrently read a dataset from multiple machines.

Query:

```sql
select  * 
from df 
where  Voertuigsoort like ‘%Personenauto%’ 
     and (Merk like ‘MERCEDES-BENZ’ or Merk like ‘BMW’ ) 
     and Inrichting like ‘%hatchback%’
```
Results:
```
+-------+--------+-------------+------------+
|       | pandas | spark local | spark yarn |
+-------+--------+-------------+------------+
| 100k  | 0.002  | 1.71        | 10.278     |
| 1m    | 0.006  | 4.91        | 16.219     |
| 5m    | 0.191  | 21.94       | 69.947     |
| 10m   | 0.12   | 43.845      | 137.986    |
| 13.5m | fail   | 64.819      | 188.339    |
+-------+--------+-------------+------------+
```
Comment:

Pandas was abviously blazing fast for small datasets where Spark struggled because of the underlying complexity.

##  Test 2: Sorting the dataset

Expectations:

This is usually a very demanding task and there is a reason why ETL developers recommend not to do sorting in memory (eg. SSIS) but only in the database engine.

Query:
```sql
select * 
from df 
order by 'Datum tenaamstelling'
```
Results:
```
+-------+---------+-------------+------------+
|       | pandas  | spark local | spark yarn |
+-------+---------+-------------+------------+
| 100k  | 0.278   | 12.66       | 13.048     |
| 1m    | 4.502   | fail        | fail       |
| 5m    | 49.631  | fail        | fail       |
| 10m   | 115.372 | fail        | fail       |
| 13.5m | fail    | fail        | fail       |
+-------+---------+-------------+------------+
```
Comment:

I was quite disappointed with Spark in both modes.

## Test 3: Joining the dataset

Expectations:

This is another taxing task which is not as brute force as sorting but can be done efficiently using a good query optimizer. Also the fact that the join is on a string column will certainly degrade performance.

Query:
```sql
select * 
from df 
join join_df  on df.Inrichting = join_df.type
```
Results:
```
+-------+--------+-------------+------------+
|       | pandas | spark local | spark yarn |
+-------+--------+-------------+------------+
| 100k  | 0.148  | 8.04        | 20.797     |
| 1m    | 1.507  | fail        | fail       |
| 5m    | 7.973  | fail        | fail       |
| 10m   | 14.72  | fail        | fail       |
| 13.5m | fail   | fail        | fail       |
+-------+--------+-------------+------------+
```
Comment:

I saw other blogpost [detailing](http://ihorbobak.com/index.php/2015/06/03/spark-sql-bad-performance/) similar problem.  The performance is disappointing but I hope Spark manages to fix this issue in the future.  Spark >= 2.0 supposedly shows great [improvements](https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html) in this aspect.

## Test 4: Self joins

Expectations:

This will be even more demanding as we are not joining to a table with 100 rows.

Query:
```sql
select * 
from  df 
join df on df.Kenteken = df.Kenteken
```
Results:

+-------+--------+-------------+------------+
|       | pandas | spark local | spark yarn |
+-------+--------+-------------+------------+
| 100k  | 0.32   | fail        | fail       |
| 1m    | 3.652  | fail        | fail       |
| 5m    | 19.928 | fail        | fail       |
| 10m   | 38.207 | fail        | fail       |
| 13.5m | fail   | fail        | fail       |
+-------+--------+-------------+------------+

Comment:

Same as for Test 3.

## Test 5: Grouping the data

Expectations:

This is a task Apache Spark should perform well in as it can be efficiently ran as Map-Reduce tesk.

Query:

```sql
select 
      count(*) 
from 
      df 
group by 
      Merk
```
Results:
```
+-------+--------+-------------+------------+
|       | pandas | spark local | spark yarn |
+-------+--------+-------------+------------+
| 100k  | 0.014  | 4.767       | 6.925      |
| 1m    | 0.118  | 5.35        | 15.998     |
| 5m    | 0.651  | 26.212      | 64.879     |
| 10m   | 1.243  | 47.805      | 130.908    |
| 13.5m | fail   | 73.143      | 171.963    |
+-------+--------+-------------+------------+
```
Comment:

Everything went as expected.

## Conclusion

These benchmarks show that although Apache Spark has great capabilities and potential in the big data area, it is still no match for Pandas on the datasets we examined.

The most worrying aspect is the bad performance of joins as well as sorting but these tasks are naturally very demanding compared to the simpler where and group-by statements which can be efficiently distributed with the Map-Reduce model. 

We could probably conclude that any dataset with less than 10 million rows (<5 GB file) shouldn’t be analyzed with Spark. For datasets larger than 5GB, rather than using a Spark cluster I propose to use Pandas on a single server with 128/160/192GB RAM. This will be more effective for intermediate size datasets (<200–500GB) than Spark (especially if you use a library like [Dask](http://dask.pydata.org/en/latest/)). For datasets above 500GB Spark combined with Hadoop Distributed File System is definitely the best solution as it allows quicker data reads and parralel workloads. Also at this data size it is quite hard to utilize SMP data processing which is much more efficient than MPP / distributed processing.

** In Spark 2.2 we got a [completely new query optimizer](https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html) so I might have to re-run the test when I get access to proper infrastructure again.
