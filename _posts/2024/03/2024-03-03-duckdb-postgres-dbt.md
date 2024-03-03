---
title: 'How I halved the runtime of my PostgreSQL dbt model using DuckDB'
date: 2024-03-03 12:00:00
categories: ["databases", "data-engineering", "sql"]
tags: ["duckdb", "dbt", "postgresql"]
toc: true
---

# How I halved the runtime of my PostgreSQL dbt model using DuckDB

With dbt, you can build fairly cross compatible SQL models with only a bit of effort.

DuckDB is a new database which is similar to SQlite in how it's an in-process but it's architecture integrates columnar storage, vectorized query execution, and advanced indexing techniques all of which enable efficient data processing

DuckDB is mostly compatible with PostgreSQL dialect of SQL (so far I've only found difference with series).  
In dbt you can use DuckDB as a plugin with the well supported [DBT adapter](https://github.com/duckdb/dbt-duckdb).  
Furthermore, DuckDB has recently made great improvements in their [PostgreSQL extension](https://duckdb.org/docs/extensions/postgres.html).

## Motivation

I have a [personal project](https://github.com/dorianbg/realestate-analytics-dbt) which stores data in PostgreSQL and has a dbt model documented [here](https://dorianbg.github.io/realestate-analytics-dbt/#!/overview).

As a fan of DuckDB I had an idea pop up after reading their recent article on [Multi-Database Support in DuckDB](https://duckdb.org/2024/01/26/multi-database-support-in-duckdb.html) alongside the optimisations that DuckDB did on the [PostgreSQL scanner](https://duckdb.org/2022/09/30/postgres-scanner.html).

Those two articles got me thinking of changing my dbt model to run with DuckDB querying my PostgreSQL database. In a sense, it's a perfect match - PostgreSQL is really robust and optimised for row oriented data storage while DuckDB is great at running column oriented analytical queries.

#### dbt targets

I use the below dbt target config for my PostgreSQL database:

##### PostgreSQL
```
property_cro_postgres:
  target: dev_postgres
  outputs:
    dev_postgres:
      type: postgres
      host: localhost
      port: 5432
      user: "{{ env_var('PGUSER') }}"
      password: "{{ env_var('PGPASSWORD') }}"
      dbname: property_cro
      schema: analytics
      threads: 4
      keepalives_idle: 0 # default 0, indicating the system default. See below
      connect_timeout: 10 # default 10 seconds
      
```

To get my dbt model working for both DuckDB and PostgreSQL, I only had to ensure that the path/alias matched the name of the actual PostgreSQL database. You can find the DuckDB target config I used below:  

##### DuckDB:
```
property_cro_duckdb:
  target: dev_duckdb
  outputs:
    dev_duckdb:
      type: duckdb
      path: prop_cro.duckdb
      extensions:
        - postgres
      attach:
        - path: "dbname=property_cro host=127.0.0.1" # presumes we setup: PGUSER, PGPASSWORD
          type: postgres
          alias: property_cro
      settings:
        pg_experimental_filter_pushdown: true
      threads: 4
```


## Results

Commands I used:

PostgreSQL: 
```
poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_postgres --target dev_postgres --threads 4 --exclude ad_price_history_agg
```

DuckDB:
```
poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_duckdb  --target dev_duckdb --threads 4 --exclude ad_price_history_agg
```

Below is a table comparing the execution times. I think using 4 threads (parallelism) is the more realistic workflow but included a run with just 1 thread for comparison.

Note execution time is how long it took the model to run but total work time includes how long did each thread spend working (running).

| Task                                   | Time (seconds) |
|----------------------------------------|---------------|
| PostgreSQL 1 thread (execution time)   | 72.46        |
| PostgreSQL 4 threads (execution time)  | 35.69         |
| PostgreSQL 4 threads (total work time) | 83.73     |
| DuckDB 1 thread (execution time)       | 26.08 |
| DuckDB 4 threads (execution time)      | 12.32     |
| DuckDB 4 threads (total work time)     | 34.13     |

Please find a more detailed breakdown in the Appendix A. 

## Conclusion

The analysis and run times might be slightly flawed due to DuckDB accessing a local instance of PostgreSQL instead of a remove instance through the network.
In public clouds the network speed would barely be noticeable on datasets of this size (<1GB of raw data). 
I do want to note that it wouldn't work as well if pulling a lot of unfiltered data to a DuckDB running on your laptop outside of the cloud (especially if through a normal router/VPN...).   

Overall, I think DuckDB should be considered more often for running analytics queries on top of PostgreSQL.
It could greatly help reduce the load on PostgreSQL and execute analytical queries more efficient with the few caveats above.

Also, the idea of column storage getting implemented into core PostgreSQL is starting to seem more and more unlikely as PostgreSQL is unlikely to ever have a column oriented push-based execution engine (though I can only hope that I am wrong). Read more on that [here](https://justinjaffray.com/query-engines-push-vs.-pull/) and even some discussion on PostgreSQL specifically [here](https://www.postgresql.org/message-id/168321488817216%40web10g.yandex.ru)

## Appendix A: comparison of dbt runtime for each object:

Note this is with 4 threads.

| Task                                                             | PostgreSQL (seconds) | DuckDB (seconds) |
|------------------------------------------------------------------|----------------------|------------------|
| analytics_rentals_flats.vw_last_ad_time                         | 0.18                 | 0.18             |
| analytics_sales_flats.vw_last_ad_time                           | 0.17                 | 0.18             |
| analytics_sales_houses.vw_last_ad_time                          | 0.17                 | 0.17             |
| analytics_common.ad_price_history                               | 10.84                | 5.78             |
| analytics_common.mv_date_to_quarter_mapping                     | 0.15                 | 0.10             |
| analytics_common.zagreb_locations                               | 0.10                 | 0.06             |
| analytics_rentals_flats.mv_price_history                        | 3.53                 | 3.38             |
| analytics_sales_houses.mv_price_history                         | 0.80                 | 0.99             |
| analytics_sales_houses.vw_avg_ask_px_per_loc                    | 0.03                 | 0.02             |
| analytics_sales_houses.vw_avg_ask_px_per_loc_size               | 0.08                 | 0.03             |
| analytics_sales_houses.vw_avg_ask_px_per_loc_year_size          | 0.03                 | 0.03             |
| analytics_sales_houses.mv_enriched_ad                           | 1.30                 | 0.38             |
| analytics_sales_houses_analysis.mv_inventory_general_location_sales_houses | 0.74                 | 0.27             |
| analytics_sales_houses_analysis.mv_inventory_location_sales_houses | 1.27                 | 0.38             |
| analytics_sales_houses_analysis.mv_inventory_sales_houses       | 0.32                 | 0.22             |
| analytics_sales_houses_alerts.vw_new_ads                       | 0.03                 | 0.03             |
| analytics_sales_houses_alerts.vw_price_drops                   | 0.03                 | 0.03             |
| analytics_rentals_flats.vw_avg_ask_px_per_loc                  | 0.04                 | 0.07             |
| analytics_rentals_flats.vw_avg_ask_px_per_loc_size             | 0.04                 | 0.13             |
| analytics_rentals_flats.vw_avg_ask_px_per_loc_year_size        | 0.03                 | 0.04             |
| analytics_rentals_flats.mv_enriched_ad                         | 3.59                 | 2.06             |
| analytics_rentals_flats_analysis.mv_inventory_general_location_rentals_flats | 2.66                 | 1.12             |
| analytics_rentals_flats_analysis.mv_inventory_location_rentals_flats | 3.98                 | 1.06             |
| analytics_rentals_flats_analysis.mv_inventory_rentals_flats     | 1.25                 | 1.13             |
| analytics_sales_flats.mv_price_history                          | 13.30                | 7.71             |
| analytics_sales_flats.vw_avg_ask_px_per_loc                    | 0.06                 | 0.17             |
| analytics_sales_flats.vw_avg_ask_px_per_loc_size               | 0.07                 | 0.17             |
| analytics_sales_flats.vw_avg_ask_px_per_loc_year_size          | 0.07                 | 0.18             |
| analytics_sales_flats.mv_enriched_ad                           | 5.59                 | 1.42             |
| analytics_sales_flats_analysis.mv_inventory_general_location_sales_flats | 11.05                | 2.14             |
| analytics_sales_flats_analysis.mv_inventory_location_sales_flats | 16.13                | 2.09             |
| analytics_sales_flats_analysis.mv_inventory_sales_flats        | 5.59                 | 2.08             |
| analytics_common.ad_price_changes_per_week                      | 0.30                 | 0.12             |
| analytics_sales_flats_alerts.vw_best_new_or_reduced_flats      | 0.03                 | 0.04             |
| analytics_sales_flats_alerts.vw_best_streets_zg                | 0.07                 | 0.03             |
| analytics_sales_flats_alerts.vw_large_price_drops              | 0.03                 | 0.03             |
| analytics_sales_flats_alerts.vw_modern_central_flats           | 0.03                 | 0.05             |
| analytics_sales_flats_alerts.vw_new_ads                        | 0.03                 | 0.03             |
| analytics_sales_flats_alerts.vw_price_drops                    | 0.02                 | 0.03             |
| **Total**                                                                                       | **83.73**            | **34.13**        |


## Appendix B: full execution logs

Below are full logs:

### PostgreSQL with 4 threads 
```
poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_postgres --target dev_postgres --threads 4 --exclude ad_price_history_agg

Configuration file exists at /Users/dbg/Library/Preferences/pypoetry, reusing this directory.

Consider moving TOML configuration files to /Users/dbg/Library/Application Support/pypoetry, as support for the legacy directory will be removed in an upcoming release.
11:10:08  Running with dbt=1.7.8
11:10:08  Registered adapter: postgres=1.7.8
11:10:08  Unable to do partial parsing because config vars, config profile, or config target have changed
11:10:08  Unable to do partial parsing because env vars used in profiles.yml have changed
11:10:08  Unable to do partial parsing because a project dependency has been added
11:10:09  Found 40 models, 7 sources, 0 exposures, 0 metrics, 526 macros, 0 groups, 0 semantic models
11:10:09
11:10:09  Concurrency: 4 threads (target='dev_postgres')
11:10:09
11:10:09  1 of 39 START sql view model analytics_rentals_flats.vw_last_ad_time ........... [RUN]
11:10:09  2 of 39 START sql view model analytics_sales_flats.vw_last_ad_time ............. [RUN]
11:10:09  3 of 39 START sql view model analytics_sales_houses.vw_last_ad_time ............ [RUN]
11:10:09  4 of 39 START sql table model analytics_common.ad_price_history ................ [RUN]
11:10:09  1 of 39 OK created sql view model analytics_rentals_flats.vw_last_ad_time ...... [CREATE VIEW in 0.18s]
11:10:09  3 of 39 OK created sql view model analytics_sales_houses.vw_last_ad_time ....... [CREATE VIEW in 0.17s]
11:10:09  2 of 39 OK created sql view model analytics_sales_flats.vw_last_ad_time ........ [CREATE VIEW in 0.17s]
11:10:09  5 of 39 START sql table model analytics_common.mv_date_to_quarter_mapping ...... [RUN]
11:10:09  6 of 39 START sql table model analytics_common.zagreb_locations ................ [RUN]
11:10:09  7 of 39 START sql table model analytics_rentals_flats.mv_price_history ......... [RUN]
11:10:09  6 of 39 OK created sql table model analytics_common.zagreb_locations ........... [SELECT 72 in 0.10s]
11:10:09  8 of 39 START sql table model analytics_sales_houses.mv_price_history .......... [RUN]
11:10:09  5 of 39 OK created sql table model analytics_common.mv_date_to_quarter_mapping . [SELECT 4550 in 0.15s]
11:10:09  9 of 39 START sql table model analytics_sales_flats.mv_price_history ........... [RUN]
11:10:10  8 of 39 OK created sql table model analytics_sales_houses.mv_price_history ..... [SELECT 14454 in 0.80s]
11:10:10  10 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc ..... [RUN]
11:10:10  10 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc  [CREATE VIEW in 0.03s]
11:10:10  11 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_size  [RUN]
11:10:10  11 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.08s]
11:10:10  12 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_year_size  [RUN]
11:10:10  12 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.03s]
11:10:10  13 of 39 START sql table model analytics_sales_houses.mv_enriched_ad ........... [RUN]
11:10:12  13 of 39 OK created sql table model analytics_sales_houses.mv_enriched_ad ...... [SELECT 14113 in 1.30s]
11:10:12  14 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_general_location_sales_houses  [RUN]
11:10:12  14 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_general_location_sales_houses  [SELECT 439 in 0.74s]
11:10:12  15 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_location_sales_houses  [RUN]
11:10:13  7 of 39 OK created sql table model analytics_rentals_flats.mv_price_history .... [SELECT 63213 in 3.53s]
11:10:13  16 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_sales_houses  [RUN]
11:10:13  16 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_sales_houses  [SELECT 23 in 0.32s]
11:10:13  17 of 39 START sql view model analytics_sales_houses_alerts.vw_new_ads ......... [RUN]
11:10:13  17 of 39 OK created sql view model analytics_sales_houses_alerts.vw_new_ads .... [CREATE VIEW in 0.03s]
11:10:13  18 of 39 START sql view model analytics_sales_houses_alerts.vw_price_drops ..... [RUN]
11:10:13  18 of 39 OK created sql view model analytics_sales_houses_alerts.vw_price_drops  [CREATE VIEW in 0.03s]
11:10:13  19 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc .... [RUN]
11:10:13  19 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc  [CREATE VIEW in 0.04s]
11:10:13  20 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_size  [RUN]
11:10:13  20 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.04s]
11:10:13  21 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
11:10:13  21 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.03s]
11:10:13  22 of 39 START sql table model analytics_rentals_flats.mv_enriched_ad .......... [RUN]
11:10:14  15 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_location_sales_houses  [SELECT 2580 in 1.27s]
11:10:17  22 of 39 OK created sql table model analytics_rentals_flats.mv_enriched_ad ..... [SELECT 62981 in 3.59s]
11:10:17  23 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [RUN]
11:10:17  24 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_location_rentals_flats  [RUN]
11:10:19  23 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [SELECT 230 in 2.66s]
11:10:19  25 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_rentals_flats  [RUN]
11:10:20  4 of 39 OK created sql table model analytics_common.ad_price_history ........... [SELECT 463818 in 10.84s]
11:10:21  25 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_rentals_flats  [SELECT 41 in 1.25s]
11:10:21  24 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_location_rentals_flats  [SELECT 2150 in 3.98s]
11:10:23  9 of 39 OK created sql table model analytics_sales_flats.mv_price_history ...... [SELECT 132262 in 13.30s]
11:10:23  26 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc ...... [RUN]
11:10:23  27 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_size . [RUN]
11:10:23  28 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
11:10:23  26 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc . [CREATE VIEW in 0.06s]
11:10:23  27 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.07s]
11:10:23  28 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.07s]
11:10:23  29 of 39 START sql table model analytics_sales_flats.mv_enriched_ad ............ [RUN]
11:10:28  29 of 39 OK created sql table model analytics_sales_flats.mv_enriched_ad ....... [SELECT 131858 in 5.59s]
11:10:28  30 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_general_location_sales_flats  [RUN]
11:10:28  31 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_location_sales_flats  [RUN]
11:10:28  32 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_sales_flats  [RUN]
11:10:28  33 of 39 START sql table model analytics_common.ad_price_changes_per_week ...... [RUN]
11:10:29  33 of 39 OK created sql table model analytics_common.ad_price_changes_per_week . [SELECT 900 in 0.30s]
11:10:29  34 of 39 START sql view model analytics_sales_flats_alerts.vw_best_new_or_reduced_flats  [RUN]
11:10:29  34 of 39 OK created sql view model analytics_sales_flats_alerts.vw_best_new_or_reduced_flats  [CREATE VIEW in 0.03s]
11:10:29  35 of 39 START sql view model analytics_sales_flats_alerts.vw_best_streets_zg .. [RUN]
11:10:29  35 of 39 OK created sql view model analytics_sales_flats_alerts.vw_best_streets_zg  [CREATE VIEW in 0.07s]
11:10:29  36 of 39 START sql view model analytics_sales_flats_alerts.vw_large_price_drops  [RUN]
11:10:29  36 of 39 OK created sql view model analytics_sales_flats_alerts.vw_large_price_drops  [CREATE VIEW in 0.03s]
11:10:29  37 of 39 START sql view model analytics_sales_flats_alerts.vw_modern_central_flats  [RUN]
11:10:29  37 of 39 OK created sql view model analytics_sales_flats_alerts.vw_modern_central_flats  [CREATE VIEW in 0.03s]
11:10:29  38 of 39 START sql view model analytics_sales_flats_alerts.vw_new_ads .......... [RUN]
11:10:29  38 of 39 OK created sql view model analytics_sales_flats_alerts.vw_new_ads ..... [CREATE VIEW in 0.03s]
11:10:29  39 of 39 START sql view model analytics_sales_flats_alerts.vw_price_drops ...... [RUN]
11:10:29  39 of 39 OK created sql view model analytics_sales_flats_alerts.vw_price_drops . [CREATE VIEW in 0.02s]
11:10:34  32 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_sales_flats  [SELECT 41 in 5.59s]
11:10:39  30 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_general_location_sales_flats  [SELECT 401 in 11.05s]
11:10:44  31 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_location_sales_flats  [SELECT 2630 in 16.13s]
11:10:44
11:10:44  Finished running 20 view models, 19 table models in 0 hours 0 minutes and 35.69 seconds (35.69s).
11:10:44
11:10:44  Completed successfully
11:10:44
11:10:44  Done. PASS=39 WARN=0 ERROR=0 SKIP=0 TOTAL=39
```


### DuckDB with 4 threads

```
poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_duckdb  --target dev_duckdb --threads 4 --exclude ad_price_history_agg

Configuration file exists at /Users/dbg/Library/Preferences/pypoetry, reusing this directory.

Consider moving TOML configuration files to /Users/dbg/Library/Application Support/pypoetry, as support for the legacy directory will be removed in an upcoming release.
11:17:46  Running with dbt=1.7.8
11:17:47  Registered adapter: duckdb=1.7.1
11:17:47  Unable to do partial parsing because config vars, config profile, or config target have changed
11:17:47  Unable to do partial parsing because env vars used in profiles.yml have changed
11:17:47  Unable to do partial parsing because a project dependency has been added
11:17:48  Found 40 models, 7 sources, 0 exposures, 0 metrics, 516 macros, 0 groups, 0 semantic models
11:17:48
11:17:48  Concurrency: 4 threads (target='dev_duckdb')
11:17:48
11:17:48  1 of 39 START sql view model main_rentals_flats.vw_last_ad_time ................ [RUN]
11:17:48  2 of 39 START sql view model main_sales_flats.vw_last_ad_time .................. [RUN]
11:17:48  3 of 39 START sql view model main_sales_houses.vw_last_ad_time ................. [RUN]
11:17:48  4 of 39 START sql table model main_common.ad_price_history ..................... [RUN]
11:17:48  1 of 39 OK created sql view model main_rentals_flats.vw_last_ad_time ........... [OK in 0.18s]
11:17:48  5 of 39 START sql table model main_common.mv_date_to_quarter_mapping ........... [RUN]
11:17:48  3 of 39 OK created sql view model main_sales_houses.vw_last_ad_time ............ [OK in 0.17s]
11:17:48  2 of 39 OK created sql view model main_sales_flats.vw_last_ad_time ............. [OK in 0.18s]
11:17:48  6 of 39 START sql table model main_common.zagreb_locations ..................... [RUN]
11:17:48  7 of 39 START sql table model main_rentals_flats.mv_price_history .............. [RUN]
11:17:49  6 of 39 OK created sql table model main_common.zagreb_locations ................ [OK in 0.06s]
11:17:49  8 of 39 START sql table model main_sales_houses.mv_price_history ............... [RUN]
11:17:49  5 of 39 OK created sql table model main_common.mv_date_to_quarter_mapping ...... [OK in 0.10s]
11:17:49  9 of 39 START sql table model main_sales_flats.mv_price_history ................ [RUN]
11:17:50  8 of 39 OK created sql table model main_sales_houses.mv_price_history .......... [OK in 0.99s]
11:17:50  10 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc .......... [RUN]
11:17:50  10 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc ..... [OK in 0.02s]
11:17:50  11 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc_size ..... [RUN]
11:17:50  11 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc_size  [OK in 0.03s]
11:17:50  12 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc_year_size  [RUN]
11:17:50  12 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc_year_size  [OK in 0.03s]
11:17:50  13 of 39 START sql table model main_sales_houses.mv_enriched_ad ................ [RUN]
11:17:50  13 of 39 OK created sql table model main_sales_houses.mv_enriched_ad ........... [OK in 0.38s]
11:17:50  14 of 39 START sql table model main_sales_houses_analysis.mv_inventory_general_location_sales_houses  [RUN]
11:17:50  14 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_general_location_sales_houses  [OK in 0.27s]
11:17:50  15 of 39 START sql table model main_sales_houses_analysis.mv_inventory_location_sales_houses  [RUN]
11:17:51  15 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_location_sales_houses  [OK in 0.38s]
11:17:51  16 of 39 START sql table model main_sales_houses_analysis.mv_inventory_sales_houses  [RUN]
11:17:51  16 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_sales_houses  [OK in 0.22s]
11:17:51  17 of 39 START sql view model main_sales_houses_alerts.vw_new_ads .............. [RUN]
11:17:51  17 of 39 OK created sql view model main_sales_houses_alerts.vw_new_ads ......... [OK in 0.03s]
11:17:51  18 of 39 START sql view model main_sales_houses_alerts.vw_price_drops .......... [RUN]
11:17:51  18 of 39 OK created sql view model main_sales_houses_alerts.vw_price_drops ..... [OK in 0.03s]
11:17:52  7 of 39 OK created sql table model main_rentals_flats.mv_price_history ......... [OK in 3.38s]
11:17:52  19 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc ......... [RUN]
11:17:52  20 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc_size .... [RUN]
11:17:52  19 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc .... [OK in 0.07s]
11:17:52  21 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
11:17:52  21 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc_year_size  [OK in 0.04s]
11:17:52  20 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc_size  [OK in 0.13s]
11:17:52  22 of 39 START sql table model main_rentals_flats.mv_enriched_ad ............... [RUN]
11:17:54  4 of 39 OK created sql table model main_common.ad_price_history ................ [OK in 5.78s]
11:17:54  22 of 39 OK created sql table model main_rentals_flats.mv_enriched_ad .......... [OK in 2.06s]
11:17:54  23 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [RUN]
11:17:54  24 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_location_rentals_flats  [RUN]
11:17:54  25 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_rentals_flats  [RUN]
11:17:55  24 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_location_rentals_flats  [OK in 1.06s]
11:17:55  23 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [OK in 1.12s]
11:17:55  25 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_rentals_flats  [OK in 1.13s]
11:17:56  9 of 39 OK created sql table model main_sales_flats.mv_price_history ........... [OK in 7.71s]
11:17:56  26 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc ........... [RUN]
11:17:56  27 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc_size ...... [RUN]
11:17:56  28 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc_year_size . [RUN]
11:17:56  27 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc_size . [OK in 0.17s]
11:17:56  26 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc ...... [OK in 0.17s]
11:17:56  28 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc_year_size  [OK in 0.18s]
11:17:56  29 of 39 START sql table model main_sales_flats.mv_enriched_ad ................. [RUN]
11:17:58  29 of 39 OK created sql table model main_sales_flats.mv_enriched_ad ............ [OK in 1.42s]
11:17:58  30 of 39 START sql table model main_sales_flats_analysis.mv_inventory_general_location_sales_flats  [RUN]
11:17:58  31 of 39 START sql table model main_sales_flats_analysis.mv_inventory_location_sales_flats  [RUN]
11:17:58  32 of 39 START sql table model main_sales_flats_analysis.mv_inventory_sales_flats  [RUN]
11:17:58  33 of 39 START sql table model main_common.ad_price_changes_per_week ........... [RUN]
11:17:58  33 of 39 OK created sql table model main_common.ad_price_changes_per_week ...... [OK in 0.12s]
11:17:58  34 of 39 START sql view model main_sales_flats_alerts.vw_best_new_or_reduced_flats  [RUN]
11:17:58  34 of 39 OK created sql view model main_sales_flats_alerts.vw_best_new_or_reduced_flats  [OK in 0.04s]
11:17:58  35 of 39 START sql view model main_sales_flats_alerts.vw_best_streets_zg ....... [RUN]
11:17:58  35 of 39 OK created sql view model main_sales_flats_alerts.vw_best_streets_zg .. [OK in 0.03s]
11:17:58  36 of 39 START sql view model main_sales_flats_alerts.vw_large_price_drops ..... [RUN]
11:17:58  36 of 39 OK created sql view model main_sales_flats_alerts.vw_large_price_drops  [OK in 0.03s]
11:17:58  37 of 39 START sql view model main_sales_flats_alerts.vw_modern_central_flats .. [RUN]
11:17:58  37 of 39 OK created sql view model main_sales_flats_alerts.vw_modern_central_flats  [OK in 0.05s]
11:17:58  38 of 39 START sql view model main_sales_flats_alerts.vw_new_ads ............... [RUN]
11:17:58  38 of 39 OK created sql view model main_sales_flats_alerts.vw_new_ads .......... [OK in 0.03s]
11:17:58  39 of 39 START sql view model main_sales_flats_alerts.vw_price_drops ........... [RUN]
11:17:58  39 of 39 OK created sql view model main_sales_flats_alerts.vw_price_drops ...... [OK in 0.03s]
11:18:00  32 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_sales_flats  [OK in 2.08s]
11:18:00  31 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_location_sales_flats  [OK in 2.09s]
11:18:00  30 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_general_location_sales_flats  [OK in 2.14s]
11:18:00
11:18:00  Finished running 20 view models, 19 table models in 0 hours 0 minutes and 12.32 seconds (12.32s).
11:18:00
11:18:00  Completed successfully
11:18:00
11:18:00  Done. PASS=39 WARN=0 ERROR=0 SKIP=0 TOTAL=39
```

### PostgreSQL with 1 thread
```
(prop-analytics-dbt-py3.10) ➜  ~/code/IdeaProjects/prop_analytics_dbt git:(main) ✗ poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_postgres --target dev_postgres --threads 1 --exclude ad_price_history_agg

Configuration file exists at /Users/dbg/Library/Preferences/pypoetry, reusing this directory.

Consider moving TOML configuration files to /Users/dbg/Library/Application Support/pypoetry, as support for the legacy directory will be removed in an upcoming release.
12:39:31  Running with dbt=1.7.8
12:39:31  Registered adapter: postgres=1.7.8
12:39:31  Unable to do partial parsing because config vars, config profile, or config target have changed
12:39:31  Unable to do partial parsing because env vars used in profiles.yml have changed
12:39:31  Unable to do partial parsing because a project dependency has been added
12:39:32  Found 40 models, 7 sources, 0 exposures, 0 metrics, 526 macros, 0 groups, 0 semantic models
12:39:32  
12:39:33  Concurrency: 1 threads (target='dev_postgres')
12:39:33  
12:39:33  1 of 39 START sql view model analytics_rentals_flats.vw_last_ad_time ........... [RUN]
12:39:33  1 of 39 OK created sql view model analytics_rentals_flats.vw_last_ad_time ...... [CREATE VIEW in 0.11s]
12:39:33  2 of 39 START sql view model analytics_sales_flats.vw_last_ad_time ............. [RUN]
12:39:33  2 of 39 OK created sql view model analytics_sales_flats.vw_last_ad_time ........ [CREATE VIEW in 0.03s]
12:39:33  3 of 39 START sql view model analytics_sales_houses.vw_last_ad_time ............ [RUN]
12:39:33  3 of 39 OK created sql view model analytics_sales_houses.vw_last_ad_time ....... [CREATE VIEW in 0.04s]
12:39:33  4 of 39 START sql table model analytics_common.ad_price_history ................ [RUN]
12:39:44  4 of 39 OK created sql table model analytics_common.ad_price_history ........... [SELECT 463818 in 10.98s]
12:39:44  5 of 39 START sql table model analytics_common.mv_date_to_quarter_mapping ...... [RUN]
12:39:44  5 of 39 OK created sql table model analytics_common.mv_date_to_quarter_mapping . [SELECT 4550 in 0.08s]
12:39:44  6 of 39 START sql table model analytics_common.zagreb_locations ................ [RUN]
12:39:44  6 of 39 OK created sql table model analytics_common.zagreb_locations ........... [SELECT 72 in 0.03s]
12:39:44  7 of 39 START sql table model analytics_rentals_flats.mv_price_history ......... [RUN]
12:39:47  7 of 39 OK created sql table model analytics_rentals_flats.mv_price_history .... [SELECT 63213 in 2.49s]
12:39:47  8 of 39 START sql table model analytics_sales_flats.mv_price_history ........... [RUN]
12:40:00  8 of 39 OK created sql table model analytics_sales_flats.mv_price_history ...... [SELECT 132262 in 13.18s]
12:40:00  9 of 39 START sql table model analytics_sales_houses.mv_price_history .......... [RUN]
12:40:00  9 of 39 OK created sql table model analytics_sales_houses.mv_price_history ..... [SELECT 14454 in 0.66s]
12:40:00  10 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc .... [RUN]
12:40:00  10 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc  [CREATE VIEW in 0.03s]
12:40:00  11 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_size  [RUN]
12:40:00  11 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.03s]
12:40:00  12 of 39 START sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
12:40:01  12 of 39 OK created sql view model analytics_rentals_flats.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.03s]
12:40:01  13 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc ...... [RUN]
12:40:01  13 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc . [CREATE VIEW in 0.03s]
12:40:01  14 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_size . [RUN]
12:40:01  14 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.02s]
12:40:01  15 of 39 START sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
12:40:01  15 of 39 OK created sql view model analytics_sales_flats.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.02s]
12:40:01  16 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc ..... [RUN]
12:40:01  16 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc  [CREATE VIEW in 0.02s]
12:40:01  17 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_size  [RUN]
12:40:01  17 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_size  [CREATE VIEW in 0.03s]
12:40:01  18 of 39 START sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_year_size  [RUN]
12:40:01  18 of 39 OK created sql view model analytics_sales_houses.vw_avg_ask_px_per_loc_year_size  [CREATE VIEW in 0.03s]
12:40:01  19 of 39 START sql table model analytics_rentals_flats.mv_enriched_ad .......... [RUN]
12:40:04  19 of 39 OK created sql table model analytics_rentals_flats.mv_enriched_ad ..... [SELECT 62981 in 2.87s]
12:40:04  20 of 39 START sql table model analytics_sales_flats.mv_enriched_ad ............ [RUN]
12:40:10  20 of 39 OK created sql table model analytics_sales_flats.mv_enriched_ad ....... [SELECT 131858 in 6.22s]
12:40:10  21 of 39 START sql table model analytics_sales_houses.mv_enriched_ad ........... [RUN]
12:40:11  21 of 39 OK created sql table model analytics_sales_houses.mv_enriched_ad ...... [SELECT 14113 in 1.09s]
12:40:11  22 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [RUN]
12:40:13  22 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [SELECT 230 in 2.35s]
12:40:13  23 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_location_rentals_flats  [RUN]
12:40:17  23 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_location_rentals_flats  [SELECT 2150 in 3.63s]
12:40:17  24 of 39 START sql table model analytics_rentals_flats_analysis.mv_inventory_rentals_flats  [RUN]
12:40:18  24 of 39 OK created sql table model analytics_rentals_flats_analysis.mv_inventory_rentals_flats  [SELECT 41 in 1.17s]
12:40:18  25 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_general_location_sales_flats  [RUN]
12:40:28  25 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_general_location_sales_flats  [SELECT 401 in 10.42s]
12:40:28  26 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_location_sales_flats  [RUN]
12:40:38  26 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_location_sales_flats  [SELECT 2630 in 9.05s]
12:40:38  27 of 39 START sql table model analytics_sales_flats_analysis.mv_inventory_sales_flats  [RUN]
12:40:42  27 of 39 OK created sql table model analytics_sales_flats_analysis.mv_inventory_sales_flats  [SELECT 41 in 4.59s]
12:40:42  28 of 39 START sql view model analytics_sales_flats_alerts.vw_best_new_or_reduced_flats  [RUN]
12:40:42  28 of 39 OK created sql view model analytics_sales_flats_alerts.vw_best_new_or_reduced_flats  [CREATE VIEW in 0.03s]
12:40:42  29 of 39 START sql view model analytics_sales_flats_alerts.vw_best_streets_zg .. [RUN]
12:40:42  29 of 39 OK created sql view model analytics_sales_flats_alerts.vw_best_streets_zg  [CREATE VIEW in 0.03s]
12:40:42  30 of 39 START sql view model analytics_sales_flats_alerts.vw_large_price_drops  [RUN]
12:40:42  30 of 39 OK created sql view model analytics_sales_flats_alerts.vw_large_price_drops  [CREATE VIEW in 0.03s]
12:40:42  31 of 39 START sql view model analytics_sales_flats_alerts.vw_modern_central_flats  [RUN]
12:40:42  31 of 39 OK created sql view model analytics_sales_flats_alerts.vw_modern_central_flats  [CREATE VIEW in 0.03s]
12:40:42  32 of 39 START sql view model analytics_sales_flats_alerts.vw_new_ads .......... [RUN]
12:40:42  32 of 39 OK created sql view model analytics_sales_flats_alerts.vw_new_ads ..... [CREATE VIEW in 0.03s]
12:40:42  33 of 39 START sql view model analytics_sales_flats_alerts.vw_price_drops ...... [RUN]
12:40:42  33 of 39 OK created sql view model analytics_sales_flats_alerts.vw_price_drops . [CREATE VIEW in 0.03s]
12:40:42  34 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_general_location_sales_houses  [RUN]
12:40:43  34 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_general_location_sales_houses  [SELECT 439 in 0.65s]
12:40:43  35 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_location_sales_houses  [RUN]
12:40:44  35 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_location_sales_houses  [SELECT 2580 in 1.21s]
12:40:44  36 of 39 START sql table model analytics_sales_houses_analysis.mv_inventory_sales_houses  [RUN]
12:40:44  36 of 39 OK created sql table model analytics_sales_houses_analysis.mv_inventory_sales_houses  [SELECT 23 in 0.28s]
12:40:44  37 of 39 START sql table model analytics_common.ad_price_changes_per_week ...... [RUN]
12:40:45  37 of 39 OK created sql table model analytics_common.ad_price_changes_per_week . [SELECT 900 in 0.34s]
12:40:45  38 of 39 START sql view model analytics_sales_houses_alerts.vw_new_ads ......... [RUN]
12:40:45  38 of 39 OK created sql view model analytics_sales_houses_alerts.vw_new_ads .... [CREATE VIEW in 0.03s]
12:40:45  39 of 39 START sql view model analytics_sales_houses_alerts.vw_price_drops ..... [RUN]
12:40:45  39 of 39 OK created sql view model analytics_sales_houses_alerts.vw_price_drops  [CREATE VIEW in 0.03s]
12:40:45  
12:40:45  Finished running 20 view models, 19 table models in 0 hours 1 minutes and 12.46 seconds (72.46s).
12:40:45  
12:40:45  Completed successfully
12:40:45  
12:40:45  Done. PASS=39 WARN=0 ERROR=0 SKIP=0 TOTAL=39
```

### DuckDB with 1 thread
```
(prop-analytics-dbt-py3.10) ➜  ~/code/IdeaProjects/prop_analytics_dbt git:(main) ✗ poetry run dbt run --profiles-dir property_cro --project-dir property_cro --profile property_cro_duckdb  --target dev_duckdb --threads 1 --exclude ad_price_history_agg 

Configuration file exists at /Users/dbg/Library/Preferences/pypoetry, reusing this directory.

Consider moving TOML configuration files to /Users/dbg/Library/Application Support/pypoetry, as support for the legacy directory will be removed in an upcoming release.
12:38:31  Running with dbt=1.7.8
12:38:32  Registered adapter: duckdb=1.7.1
12:38:32  Found 40 models, 7 sources, 0 exposures, 0 metrics, 516 macros, 0 groups, 0 semantic models
12:38:32  
12:38:34  Concurrency: 1 threads (target='dev_duckdb')
12:38:34  
12:38:34  1 of 39 START sql view model main_rentals_flats.vw_last_ad_time ................ [RUN]
12:38:34  1 of 39 OK created sql view model main_rentals_flats.vw_last_ad_time ........... [OK in 0.12s]
12:38:34  2 of 39 START sql view model main_sales_flats.vw_last_ad_time .................. [RUN]
12:38:34  2 of 39 OK created sql view model main_sales_flats.vw_last_ad_time ............. [OK in 0.08s]
12:38:34  3 of 39 START sql view model main_sales_houses.vw_last_ad_time ................. [RUN]
12:38:34  3 of 39 OK created sql view model main_sales_houses.vw_last_ad_time ............ [OK in 0.08s]
12:38:34  4 of 39 START sql table model main_common.ad_price_history ..................... [RUN]
12:38:39  4 of 39 OK created sql table model main_common.ad_price_history ................ [OK in 4.73s]
12:38:39  5 of 39 START sql table model main_common.mv_date_to_quarter_mapping ........... [RUN]
12:38:39  5 of 39 OK created sql table model main_common.mv_date_to_quarter_mapping ...... [OK in 0.09s]
12:38:39  6 of 39 START sql table model main_common.zagreb_locations ..................... [RUN]
12:38:39  6 of 39 OK created sql table model main_common.zagreb_locations ................ [OK in 0.06s]
12:38:39  7 of 39 START sql table model main_rentals_flats.mv_price_history .............. [RUN]
12:38:40  7 of 39 OK created sql table model main_rentals_flats.mv_price_history ......... [OK in 0.94s]
12:38:40  8 of 39 START sql table model main_sales_flats.mv_price_history ................ [RUN]
12:38:46  8 of 39 OK created sql table model main_sales_flats.mv_price_history ........... [OK in 6.40s]
12:38:46  9 of 39 START sql table model main_sales_houses.mv_price_history ............... [RUN]
12:38:47  9 of 39 OK created sql table model main_sales_houses.mv_price_history .......... [OK in 0.35s]
12:38:47  10 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc ......... [RUN]
12:38:47  10 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc .... [OK in 0.06s]
12:38:47  11 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc_size .... [RUN]
12:38:47  11 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc_size  [OK in 0.06s]
12:38:47  12 of 39 START sql view model main_rentals_flats.vw_avg_ask_px_per_loc_year_size  [RUN]
12:38:47  12 of 39 OK created sql view model main_rentals_flats.vw_avg_ask_px_per_loc_year_size  [OK in 0.08s]
12:38:47  13 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc ........... [RUN]
12:38:47  13 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc ...... [OK in 0.06s]
12:38:47  14 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc_size ...... [RUN]
12:38:47  14 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc_size . [OK in 0.06s]
12:38:47  15 of 39 START sql view model main_sales_flats.vw_avg_ask_px_per_loc_year_size . [RUN]
12:38:47  15 of 39 OK created sql view model main_sales_flats.vw_avg_ask_px_per_loc_year_size  [OK in 0.08s]
12:38:47  16 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc .......... [RUN]
12:38:47  16 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc ..... [OK in 0.06s]
12:38:47  17 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc_size ..... [RUN]
12:38:47  17 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc_size  [OK in 0.06s]
12:38:47  18 of 39 START sql view model main_sales_houses.vw_avg_ask_px_per_loc_year_size  [RUN]
12:38:47  18 of 39 OK created sql view model main_sales_houses.vw_avg_ask_px_per_loc_year_size  [OK in 0.08s]
12:38:47  19 of 39 START sql table model main_rentals_flats.mv_enriched_ad ............... [RUN]
12:38:48  19 of 39 OK created sql table model main_rentals_flats.mv_enriched_ad .......... [OK in 0.87s]
12:38:48  20 of 39 START sql table model main_sales_flats.mv_enriched_ad ................. [RUN]
12:38:50  20 of 39 OK created sql table model main_sales_flats.mv_enriched_ad ............ [OK in 1.56s]
12:38:50  21 of 39 START sql table model main_sales_houses.mv_enriched_ad ................ [RUN]
12:38:50  21 of 39 OK created sql table model main_sales_houses.mv_enriched_ad ........... [OK in 0.46s]
12:38:50  22 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [RUN]
12:38:50  22 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_general_location_rentals_flats  [OK in 0.39s]
12:38:50  23 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_location_rentals_flats  [RUN]
12:38:51  23 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_location_rentals_flats  [OK in 0.28s]
12:38:51  24 of 39 START sql table model main_rentals_flats_analysis.mv_inventory_rentals_flats  [RUN]
12:38:51  24 of 39 OK created sql table model main_rentals_flats_analysis.mv_inventory_rentals_flats  [OK in 0.25s]
12:38:51  25 of 39 START sql table model main_sales_flats_analysis.mv_inventory_general_location_sales_flats  [RUN]
12:38:53  25 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_general_location_sales_flats  [OK in 1.85s]
12:38:53  26 of 39 START sql table model main_sales_flats_analysis.mv_inventory_location_sales_flats  [RUN]
12:38:55  26 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_location_sales_flats  [OK in 2.07s]
12:38:55  27 of 39 START sql table model main_sales_flats_analysis.mv_inventory_sales_flats  [RUN]
12:38:57  27 of 39 OK created sql table model main_sales_flats_analysis.mv_inventory_sales_flats  [OK in 1.90s]
12:38:57  28 of 39 START sql view model main_sales_flats_alerts.vw_best_new_or_reduced_flats  [RUN]
12:38:57  28 of 39 OK created sql view model main_sales_flats_alerts.vw_best_new_or_reduced_flats  [OK in 0.06s]
12:38:57  29 of 39 START sql view model main_sales_flats_alerts.vw_best_streets_zg ....... [RUN]
12:38:57  29 of 39 OK created sql view model main_sales_flats_alerts.vw_best_streets_zg .. [OK in 0.06s]
12:38:57  30 of 39 START sql view model main_sales_flats_alerts.vw_large_price_drops ..... [RUN]
12:38:57  30 of 39 OK created sql view model main_sales_flats_alerts.vw_large_price_drops  [OK in 0.06s]
12:38:57  31 of 39 START sql view model main_sales_flats_alerts.vw_modern_central_flats .. [RUN]
12:38:57  31 of 39 OK created sql view model main_sales_flats_alerts.vw_modern_central_flats  [OK in 0.06s]
12:38:57  32 of 39 START sql view model main_sales_flats_alerts.vw_new_ads ............... [RUN]
12:38:57  32 of 39 OK created sql view model main_sales_flats_alerts.vw_new_ads .......... [OK in 0.06s]
12:38:57  33 of 39 START sql view model main_sales_flats_alerts.vw_price_drops ........... [RUN]
12:38:57  33 of 39 OK created sql view model main_sales_flats_alerts.vw_price_drops ...... [OK in 0.06s]
12:38:57  34 of 39 START sql table model main_sales_houses_analysis.mv_inventory_general_location_sales_houses  [RUN]
12:38:57  34 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_general_location_sales_houses  [OK in 0.26s]
12:38:57  35 of 39 START sql table model main_sales_houses_analysis.mv_inventory_location_sales_houses  [RUN]
12:38:58  35 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_location_sales_houses  [OK in 0.12s]
12:38:58  36 of 39 START sql table model main_sales_houses_analysis.mv_inventory_sales_houses  [RUN]
12:38:58  36 of 39 OK created sql table model main_sales_houses_analysis.mv_inventory_sales_houses  [OK in 0.12s]
12:38:58  37 of 39 START sql table model main_common.ad_price_changes_per_week ........... [RUN]
12:38:58  37 of 39 OK created sql table model main_common.ad_price_changes_per_week ...... [OK in 0.09s]
12:38:58  38 of 39 START sql view model main_sales_houses_alerts.vw_new_ads .............. [RUN]
12:38:58  38 of 39 OK created sql view model main_sales_houses_alerts.vw_new_ads ......... [OK in 0.06s]
12:38:58  39 of 39 START sql view model main_sales_houses_alerts.vw_price_drops .......... [RUN]
12:38:58  39 of 39 OK created sql view model main_sales_houses_alerts.vw_price_drops ..... [OK in 0.06s]
12:38:58  
12:38:58  Finished running 20 view models, 19 table models in 0 hours 0 minutes and 26.08 seconds (26.08s).
12:38:58  
12:38:58  Completed successfully
12:38:58  
12:38:58  Done. PASS=39 WARN=0 ERROR=0 SKIP=0 TOTAL=39

```
