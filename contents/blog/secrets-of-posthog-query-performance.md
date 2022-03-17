---
date: "2022-03-16" # :TODO
title: "Secrets of PostHog query performance"
rootPage: /blog
sidebar: Blog
showTitle: true
hideAnchor: true
featuredImage: ../images/blog/launch-week-teaser.jpeg # :TODO:
featuredImageType: full
categories: ["Engineering", "Product updates", "Launch week"]
author: ["karl-aksel-puulmann"]
---

We want PostHog to become the first choice for product analytics at any scale. To do that, users should have a smooth experience exploring their product data - including not waiting minutes for queries to load.

In this post, I’m going to break down the why’s and how’s of several major performance improvements we've made in the last few months.

## How does querying work within PostHog?

The free-form querying experience in PostHog allows you to ask questions about your Trends, Funnels, Retention, and Cohorts with complicated filtering to top it off.

These queries are processed by [ClickHouse](https://clickhouse.com/), where event, user, and group data is stored in a raw format without any preaggregation. ClickHouse excels at analyzing billions of events in seconds even with complex filtering.

However, as awesome as ClickHouse is, nothing is without sharp edges and trade-offs.

## Speeding up property filtering (up to) 25x

PostHog allows users to send and analyze arbitrary number of event and user properties with their data. We store this data as JSON-encoded strings in our tables.

One of the first issues we saw after moving to ClickHouse was that, for our largest users, filtering by properties was slow.

[Benchmarking these queries using flamegraphs](https://github.com/Slach/clickhouse-flamegraph) showed that the slowness came from two things: reading JSON properties from disk and (to a lesser extent) parsing it during query-time.

We ended up [materializing the most used properties into new columns](/blog/clickhouse-materialized-columns), leveraging the strength of a columnar database without running into column number limits.

Specifically, the new materialized columns are fast to read from disk as they compress really well and ClickHouse can skip parsing JSON entirely during queries.

On our PostHog Cloud setup, we saw this feature improve query performance on average 55% with the p99 improvement being 25x improvement.

## Speeding up JOINs for large users 10x

Over time, for larger PostHog users with over 10 million visitors some simple queries like (count of unique users) started timing out or running into memory errors.

We narrowed this down to one particular JOIN in our system:

```sql
JOIN (
    SELECT distinct_id, argMax(person_id, _timestamp) as person_id
    FROM (
        SELECT distinct_id, person_id, max(_timestamp) as _timestamp
        FROM person_distinct_id
        WHERE project_id = 123
        GROUP BY person_id, distinct_id, team_id
        HAVING max(is_deleted) = 0
    )
    GROUP BY distinct_id
) AS pdi

-- Table schema
CREATE TABLE person_distinct_id
(
    distinct_id VARCHAR,
    person_id UUID,
    project_id Int64,
    _sign Nullable(Int8),
    is_deleted Nullable(Int8),
    _timestamp DateTime
) ENGINE = CollapsingMergeTree(_sign)
```

This JOIN is as complicated as it is due to a restriction from ClickHouse: updating data is expensive. In ClickHouse, updates require rewriting whole parts of the table instead of individual rows.

Instead, the recommended approach is to use a ReplacingMergeTree or CollapsingMergeTree [TODO LINKS] table engine and handle updating logic at query-time.

During data ingestion, when a given `distinct_id` had its `person_id` changed, PostHog emits a row with `is_deleted=1` for the old `person_id` and a new row with `is_deleted=0`. The above query would then resolve the `distinct_id` => `person_id` mapping at query time.

However, in practice, this query took too much memory and was slow due to needing a subquery to aggregate data correctly. It also had subtle issues with using timestamps for versioning, causing issues.

After noticing the problem, we realised we didn't need to actually emit rows with `is_deleted=0` to behave correctly, and could move to an alternative schema, which can be queried as follows:

```sql
JOIN (
    SELECT distinct_id, argMax(person_id, version) as person_id
    FROM person_distinct_id2
    WHERE project_id = 123
    GROUP BY distinct_id
    HAVING argMax(is_deleted, version) = 0
) AS pdi
```

For PostHog users with over 10 million visitors, this sped up queries previously bottlenecked on this JOIN by up to 10x.

### Sorting events for a 23% win

While debugging other performance issues, one question we kept asking is whether our data is laid out optimally for performance.

PostHog uses a [ClickHouse MergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree/) table engine to store event data on disk. MergeTree tables can have an `ORDER BY` clause, which is then used by ClickHouse to store the data in a sorted format on disk and to create a sparse index of the data. The sparse index can then be used to skip reading data during queries.

For PostHog our `ORDER BY` clause originally looked something like follows:

```sql
ORDER BY (project_id, toDate(timestamp), cityHash64(distinct_id), cityHash64(uuid))
```

Let’s consider this simplified query counting the number of users who did pageviews in a time range.

```sql
SELECT count(DISTINCT person_id)
FROM events
WHERE project_id = 2
    AND event = '$pageview'
    AND timestamp >= '2022-03-01'
    AND timestamp < '2022-04-01'
```

When executing this query, ClickHouse is able to leverage data being sorted and the sparse index to skip reading most of data from disk. In this case, events from other projects and organizations and events from months other than March.

However, almost all of our most time-sensitive queries in PostHog also filter by event type. After measuring and confirming this, we updated the `ORDER BY` clause:

```sql
ORDER BY (project_id, toDate(timestamp), event, cityHash64(distinct_id), cityHash64(uuid))
```

This resulted in a roughly 23% query speed up on average. The best trick to performance optimization is to skip doing unnecessary work.

### Migrating data

Changing how data is sorted on disk is not cheap when you have billions on billions of events. For example, on PostHog Cloud this took five separate attempts and over either weeks total to finish. In addition, PostHog has a lot of self-hosted users at various degrees of scale and technical skill who would need to repeat this process.

To tackle this problem, we ended up building a new [async migrations](/docs/self-host/configure/async-migrations/overview) system, which safely runs these long-running operations at scale with the press of a button while handling common edge cases and keeping the platform usable.

## Self-hosted performance

As mentioned, PostHog can be self-hosted by our users. However getting it working smoothly across a wide range of deployments at scale [keeps our infrastructure team hard at work](TODO-link-to-harry-guido-blog-post).

Some features released in PostHog 1.34.0 which affect performance for self-hosted users are:

- Ability to use an [external ClickHouse provider](docs/self-host/configure/using-altinity-cloud). We’ve [partnered with Altinity to help support larger installations](TODO).
- Support for ClickHouse sharding and replication in our helm chart LINK. This allows leveraging more machines for faster queries.
- Upgrading to ClickHouse 21.11: ClickHouse is changing rapidly and each new release is bringing in new performance improvements.

## What’s next?

Performance work is never complete and PostHog has a lot of work ahead of us to make answering questions about your product fast, no matter your scale.

Some projects currently in the pipeline are:

- **Removing JOINs for persons (and groups)** - ClickHouse is not designed for doing large-scale joins. We’re currently in the middle of refactoring our whole schema to remove the need for JOINs, bypassing the biggest bottleneck most queries share. Read more about our plans here [link to meta PR]
- **Smart caching time-series queries** - PostHog dashboards continually refresh data to show up-to-date graphs. However this results in a lot of repeated work, slowing down queries. By changing semantics around user properties and identifying users, we will be able to start smartly re-using past results when re-calculating queries.
- **Better JSON support in ClickHouse** - [This feature](https://github.com/ClickHouse/ClickHouse/issues/23516) is expected to land in the coming months in ClickHouse and will unlock the benefits of materialized columns with much less complexity.

_Interested on chatting about ClickHouse performance or working on similar problems? Send me an email: [karl+perf@posthog.com](mailto:karl+perf@posthog.com)_