---
author: "Or Elimelech"
date: 2022-03-14
title: Build a datalake on top of BigQuery
tags: ['bigquery', 'data', 'google-cloud']
---

Google BigQuery is a very powerful, serverless data warehosue that
lets you ingest unlimited data on a pay-per-use basis (storage + querying).
The primary advantage of data warehouses is the ability to quickly query and analyze immense amounts of **structured** data.

Modern data warehouses support new, unstructured data types such as JSON, Avro, and so on, which makes these data warehouses a great contender for data lakes.
BigQuery recently added native JSON column type, which we can leverage for our semi-structured lakehouse.
With JSON support we can skip the traditional datalakes that are just blob storage with files, from classic Hadoop + hive through GCS, S3, Athena etc.
Maintaining such infrastructure and understanding the low-level components, such as metastore, ORC, and Parquet is an expensive process for most startups, financially and with regard to domain expertise.

## Data Lakes

Let's take a step back and look at standard data lakes. This will lay the foundation for clearly understanding the benefits of using BigQuery for these modern, advanced workloads that contain unstructred data.
Data lakes are built for unstructured data using blob storage systems by using efficient compression and columnar data files, such as Parquet.
The files are partitioned using a special directory structure for the given time range, making it easy to scan specific time ranges.
Now, ideally we need to make it even faster to query lots of data for a specific `event_name`. To do this, we need to use another component called metastore.

Metastore is a meta-data storage system that can index certain properties of the data, which makes it easier to pinpoint the correct files containing the relevant data.
We have two separate large-scale systems that data engineers need to manage.
Filesystems are cumbersome and much more difficult to both query and maintain.

It's worth noting that blob storage has several clear advantages:
1. Cheaper to store data.
1. Contains storage tiers, such as archival, cold, and warm.
1. Can hold data of any type or size (audio, images, executables, etc.).
   - Real unstructured data like audio can be used for ML piplines, text-to-speech algorithms, etc.
1. Easy to replicate across regions and providers.

## What's a data lakehouse?

A data warehouse is storage for structured data. A data lake is storage for unstructured data. A **data lakehouse** is the beautiful storage love child of the warehouse and lake.
The lakehouse leverages the semi-structured JSON column of BigQuery and helps us ingest a flexible data structure while the native columns act as our metastore.

## Lakehouse example

In this example, we use several columns for partioning (timestamp) and clustering (event_name, source). They act as metadata to our semi-structured data.
This way it's easy to navigate and explore data via SQL and to narrow down the subset of data we need. We can then use the new native JSON notation in our query:

```sql
SELECT id, timestamp, event_name, data.hello FROM datalake
```

Next, we use materialized-views or scheduled queries (via BigQuery/Airflow etc.) to build our final facts and dimensions tables.

| id                                   | timestamp                   | event_name         | source    | content_type     | data                          | data_bytes |
| ---                                  | ---                         | ---                | ---       | ---              | ---                           | ---        |
| 6EF89A2E-8E11-4848-91B3-11682020559F | Sun Mar 6 15:48:24 IST 2022 | something_happened | service-a | application/json | `{"hello": "from lakehouse"}` | NULL       |


## Easy ETLs with [materialized-views](https://cloud.google.com/bigquery/docs/materialized-views-intro)

IMHO SQL is a great language to use to describe data.
SQL is making a noticeable come back. Even Elastic and MongoDB support SQL. The rise of new-SQL databases like CockroachDB, modern warehouses, logging systems, etc. are clear evidence that SQL is here to stay.

We will use a cheap form of ETL using BigQuery's materialized-view feature.
The default refresh rate for materialized-view is 30m, you can of course tweak that, but it's sufficient for most workloads.

> If you already have such system in place you can use it as well.
> See: Airflow, `dbt` or just a simple Kubernetes job.

```sql
CREATE MATERIALIZED VIEW `<project_id>.<dataset>.canary_tests`
PARTITION BY TIMESTAMP_TRUNC(timestamp, HOUR)
CLUSTER BY service_name AS
SELECT
  timestamp,
  data.service_name AS service_name,
  data.job_name AS job_name,
  data.workflow_id AS workflow_id,
  data.test_name AS test_name,
  data.build_url as build_url,
FROM
  `<project_id>.<dataset>.datalake_v1`
WHERE
  event_name = "something_happened"
```

## Conclusion
BigQuery is an excellent candidate to serve as the basis for your next data lake adventure (good luck!).

- BigQuery can help you start small and easily scale out.
- Organizations of all sizes will benefit (if you don't store anything you won't pay for it).
- Data engineering teams can build an impressive data platform levereging the power of SQL.
