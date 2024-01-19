---
title: "Data pipeline design anti-patterns"
date: "2023-03-18"
categories: 
  - "data-engineering"
---

- Bunch of scripts

- Single run-everything script

- Hacky "homemade" dependency control

- Hard-coded values eg. data sources, queries… 

- ETL jobs that create duplicate data if ran twice or create holes in the data if not ran at all

- No data quality checks or data ingestion monitoring
