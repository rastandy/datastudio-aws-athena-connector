# AWS Athena Connector for Data Studio

![](./assets/example.png)

*This is not an official Google product*

This [Data Studio](https://datastudio.google.com) [Community
Connector](https://developers.google.com/datastudio/connector) lets users query data from AWS S3 Buckets directly.

The connector is using [AWS Athena](https://aws.amazon.com/athena/) for underlying queries.

## Performance optimizations (this fork)

This fork adds latency optimizations for Looker Studio dashboards backed by
partitioned Athena tables:

- **Athena result reuse** — identical repeat queries return cached results
  without re-scanning S3 (requires Athena engine v3). Configurable via the new
  **Athena Result Reuse (minutes)** connector setting (default `60`; `0`
  disables).
- **Partition pruning** — the generated `WHERE` clause restricts scans to the
  Hive partitions (`year`/`month`/`day`) covered by the report's date range,
  instead of scanning the whole table.
- **Faster status polling** — exponential backoff (250 ms → 2 s cap) while
  waiting for a query, instead of a flat 3 s sleep, so fast or reused queries
  return almost immediately.
- **Glue schema cache** — the table schema is cached (10 min) to avoid a Glue
  `GetTable` call on every `getData` request.
- **Memoized crypto** — the CryptoJS dependency is loaded once per execution.

## Try the Community Connector in Data Studio

### Notes

This example is running in the `us-west-2` region.

### Create IAM User

Create an [IAM User](https://console.aws.amazon.com/iam/home) with **programmatic access**.

Attach managed policies `AmazonAthenaFullAccess` and `AmazonS3ReadOnlyAccess` to this user.

Remember the user's access key and secret.

### Create Athena Table

Visit the [Athena Console](https://us-west-2.console.aws.amazon.com/athena/home) and create a sample table:

```
CREATE EXTERNAL TABLE IF NOT EXISTS cloudfront_logs (
  LogDate DATE,
  Time STRING,
  Location STRING,
  Bytes INT,
  RequestIP STRING,
  Method STRING,
  Host STRING,
  Uri STRING,
  Status INT,
  Referrer STRING,
  os STRING,
  Browser STRING,
  BrowserVersion STRING
  ) ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
  WITH SERDEPROPERTIES (
  "input.regex" = "^(?!#)([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+([^ ]+)\\s+[^\(]+[\(]([^\;]+).*\%20([^\/]+)[\/](.*)$"
  ) LOCATION 's3://athena-examples-us-west-2/cloudfront/plaintext/';
```

You could then try `SELECT * FROM "default"."cloudfront_logs" limit 10;` to preview the table.

### Setup Connector

In the connector, fill in the values like this:

Key                      | Value
-------------------------| -----
`AWS_ACCESS_KEY_ID`      | {KEY}
`AWS_SECRET_ACCESS_KEY`  | {SECRET}
`AWS Region`             | {AWS_REGION}
`Glue Database Name`     | `default`
`Glue Table Name`        | `cloudfront_logs`
`Query Output Location`  | `s3://aws-athena-query-results-{account_id}-us-west-2/data-studio`
`Date Range Column Name` | `LogDate`

For `Query Output Location`, AWS should have created a S3 bucket to store the query results, you could find the bucket name in S3 console.

If not, you could create a S3 bucket that starts with the name `aws-athena-query-results-` yourself.

### Create Report

Data Studio will automatically crawls the table schema.

You could then try to explore the data. Note that the sample data is ranged from `2014-07-05` to `2014-08-05`.
