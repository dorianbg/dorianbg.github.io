---
title: "Advanced data analysis for cBioPortal"
date: "2017-09-24"
categories: 
  - "data-science"
---

As part of my application for the cBioPortal Google Summer of Code position I made this Jupyter notebook in which I demonstrated how different studies (RDBMS equivalent of a table) could be clustered together into types of studies based on the attributes measured (column names and contents of columns).

The goal of the project was to develop a clustering algorithm that would recognize and cluster all of the studies into different study types. We can also state it as: developing a Fuzzy Matching algorithm in order to further normalize the database.

In the Jupyter notebook I did the following:

1. First I parsed data from a REST API providing data in JSON format into pandas dataframes.
2. Then I showed graphically the results of clustering of studies based on names measured attributes (names of columns)
3. Lastly, I implemented similarity scoring functions (Fuzzy Matching) to score the similarity of studies based on values of attributes (columns in a table) and graphically plotted those results in a heatmap using Seaborn package.

### You can take a look at result of the process here:

https://github.com/dorianb96/cBioPortal-Advanced-Data-Analysis/blob/master/cBioPortal_Data_Analysis.ipynb
