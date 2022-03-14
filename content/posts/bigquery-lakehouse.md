---
author: "Or Elimelech"
date: 2022-03-14
title: Build a datalake on top of BigQuery
---

Google BigQuery is a very powerfull serverless data-warehosue.
It lets you ingest any amount of data and pay per-use (storage + querying).
The strength of datawarehouses is first and foremost being able to query and analyze imense amounts of **structured** data.

Modern warehouses support new unstructured data types such as JSON, Avro etc. which makes them a great contender for datalakes.
BigQuery recently added native JSON column type which we can leverage for our semi-structured lakehouse.
With JSON support we can skip the traditional datalakes that are just blob storage with files, from classic Hadoop + hive through GCS, S3, Athena etc.
Maintaining such infrastructure and understanding the low level parts like metastore, ORC, Parquet is an expensive process and expertise for most startups.

## Datalakes

In order to understand better what's the benefits of using BigQuery for such workload let's go back a bit to standard lakes.
Datalakes are built for unstructured data using blob storage systems using efficient compression and columnar data files such as Parquet.
The files are partitioned using a special directory structure for the time range to make it easy to scan specific time ranges.
Now we need to make it even faster to query lots of data for a specific `event_name` we need to use another component called metastore.

Metastore is a meta-data storage system that can index certain properties of the data which makes it easier to pin-point the correct files containing the relevant data.
Now we have two separate large scale systems that needs to be managed by data engineerns.
Filesystems are cumbersome, harder to query and maintain.

It's worth noting that blob storage have some great advantages:
1. Cheaper when it comes to storing data.
1. Have storage tiers like archival, cold and warm.
1. Can hold any type or size of data (audio, images, executables).
   - Real unstructured data like audio can be used for ML piplines, text-to-speech algorithms etc.
1. Easy to replicate across regions and providers.

## What's a data lakehouse?

If a warehouse is structured data and lake is unstructured then lakehouse is the best of both worlds.
Exploiting the semi-structured JSON column of BigQuery helps us ingest flexible data structure while the native columns will act as our metastore.

## Example

| id                                   | timestamp                   | event_name         | source    | content_type     | data                          | data_bytes |
| ---                                  | ---                         | ---                | ---       | ---              | ---                           | ---        |
| 6EF89A2E-8E11-4848-91B3-11682020559F | Sun Mar 6 15:48:24 IST 2022 | something_happened | service-a | application/json | `{"hello": "from lakehouse"}` | NULL       |

The example above has a few columns that we use for partioning (timestamp) and clustering (event_name, source) and act as metadata to our semi-structured data.
This way it's easy to navigate and explore data via SQL and narrow down the subset of data you need, then use the new native JSON notation

```sql
SELECT id, timestamp, event_name, data.hello FROM datalake
```

Next, we use materialized-views or scheduled queries (via BigQuery/Airflow etc.) to build our final facts and dimensions tables.

## Easy ETLs with materialized-views

IMHO SQL is a great language to describe data.
We can see SQL coming back big-time, even Elastic and MongoDB has SQL support today. The rise of new-SQL databases like CockroachDB, modern warehouses, logging systems etc. are great evidence that SQL is here to stay.

We will use a cheap form of ETL using BigQuery's materialized-view feature.
The default refresh rate for materialized-view is 30m, you can of course tweak that but it's sufficient enough for most workloads.

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

- BigQuery can help you start small and grow big.
- It fits all sizes of organizations, if you don't store anything you won't pay for it.
- It helps teams build a great data platform with just the power of SQL.
