---
layout: post
title: "Self documenting SQL query using CTEs"
description: ""
date: 2017-08-30
tags: []
comments: true
---

The few times we’ve moved expensive Ruby code to SQL has been very rewarding. For example, we had a query in a job that used to take ~17 seconds trimmed to ~50 milliseconds. We also had an index page that sporadically timed out because we were initializing ActiveRecord objects and their associations before computing the needed data that is now humming along after we created an SQL view to match the columns that we need to display. SQL views are clearly powerful. However, I always experiment with remodeling the Ruby code first because there’s a level of readability that I can’t apply when I write complex SQL.

I recently worked on a feature where we want to delay the next run time of jobs that have been failing. A setting whose most recent job failed will get an offset of 1 hour on their next expected run time. The offset is equivalent to the nth number in the fibonacci sequence where n is the number of contiguous failures starting from the latest job.
Assume we have the following tables:

```sql
CREATE TABLE settings (
  id              serial,
  job_frequency   integer
);
CREATE TABLE jobs (
  id              serial,
  success         boolean,
  created_at      timestamp,
  setting_id      integer
);
```

And here's how we want the offset to be computed:

1. Tally the statuses of a setting's jobs starting from the latest.

2. Note the number of contiguous failures of the first 7 jobs from the tally. We want the offset to be 13 hours at most so the maximum fibonacci seed is 7. On a series with success equals to `false, true, false, false, false, true, false`, only the last `false` is significant.

3. Run the fibonacci sequence as many times as the number of significant failures.

4. Get the last computed value in the fibonacci sequence and return it as the offset.

After several attempts to shorten or flatten the subqueries, I chanced upon Postgres' Common Table Expressions. Using the steps above as guide, I was able to write a series of named statements that allowed me to write an explicit `SELECT` query at the end:

```sql
WITH RECURSIVE -- since the fibonacci statement is a recursion, the RECURSIVE modifier must be added
  constants AS (
    SELECT 7 AS max_fibonacci_seed
  ),
  fibonacci(a, b) AS (
    VALUES(0, 1)
    UNION ALL
    SELECT greatest(a, b), a + b AS a FROM fibonacci
  ),
  failures AS (
    SELECT
      row_number() OVER (
        PARTITION BY setting_id ORDER BY jobs.created_at DESC
      ) AS row_number,
      bit_and (
        CASE WHEN success = true THEN 1
        ELSE 0 END
      ) OVER (
        PARTITION BY setting_id ORDER BY jobs.created_at DESC
      ) AS failing,
      setting_id,
      created_at
    FROM jobs
  ),
  significant_failures_per_setting AS (
    SELECT
      MAX(row_number) OVER (
        PARTITION BY setting_id
      ) AS significant_failures,
      MAX(created_at) OVER (
        PARTITION BY setting_id
      ) AS latest_job_created_at,
      setting_id
    FROM failures -- previously defined statements are available to latter statements
    WHERE failing = 1
    AND row_number <= (
      SELECT max_fibonacci_seed FROM constants
    )
    ORDER BY created_at DESC
  ),
  offset_in_hours_per_failing_setting AS (
    SELECT
      setting_id,
      latest_job_created_at,
      (
        SELECT a
        FROM fibonacci
        LIMIT 1
        OFFSET significant_failures
      ) AS offset_in_hours
    FROM significant_failures_per_setting
  )

SELECT
  (
    latest_job_created_at +
    CONCAT(offset_in_hours, ' hours')::INTERVAL
  ) AS next_scheduled_run_time
FROM offset_in_hours_per_failing_setting;
```
