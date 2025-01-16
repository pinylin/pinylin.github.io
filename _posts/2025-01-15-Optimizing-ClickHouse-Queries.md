---
layout: post
title: A Step by Step Troubleshooting Journey
subtitle: 
author: pinylin
header-style: text
catalog: true
tags:
  - ClickHouse
  - 全文检索
  - index
---

> Previously, a product line used the ELK stack for their logging system, processing approximately 1.5 TB of data daily. After migrating to ClickHouse, the daily data volume reduced to around 0.5 TB (excluding replications). However, after the migration, we encountered a critical issue where the keyword search functionality has become painfully slow, and are unable to effectively retrieve historical data.


Initially, no one suspected that the issue was the query statements; the consensus was that the large volume of data was the primary culprit for the low query efficiency.

So, the first step was to test the performance of the full-text index.

## Test Case

- full_text
```
   ┌─database─┬─table─────────┬─compressed_size─┬─uncompressed_size─┬─ratio─┐
│ logs     │ hackernews_ft │ 6.24 GiB        │ 11.00 GiB         │  1.76 │
└──────────┴───────────────┴─────────────────┴───────────────────┴───────┘
```
- tokenbf
```
┌─database─┬─table──────────┬─compressed_size─┬─uncompressed_size─┬─ratio─┐
│ logs     │ hackernews_tbf │ 6.24 GiB        │ 11.00 GiB         │  1.76 │ └──────────┴────────────────┴─────────────────┴───────────────────┴───────┘
```

## count(id)

|           | tokenbf_v1 | full_text | t&f    | cnt  |     |
| --------- | ---------- | --------- | ------ | ---- | --- |
| kilobyte  | 7.218      | 5.327     | 7.515  | 459  |     |
| megabyte  | 8.835      | 8.994     | 7.283  | 1741 |     |
| gigabyte  | 11.473     | 6.708     | 9.463  | 2648 |     |
| terabyte  | 10.705     | 6.856     | 11.727 | 1646 |     |
| petabyte  | 8.535      | 8.197     | 7.348  | 929  |     |
| exabyte   | 3.066      | 2.029     | 2.138  | 191  |     |
| zettabyte | 0.478      | 0.406     | 1.283  | 53   |     |
| yottabyte | 0.673      | 0.502     | 1.019  | 68   |     |
|           |            |           |        |      |     |
| assessmen | 0.609      | 0.163     | 0.271  | 15   |     |
| gatewa    | 0.553      | 0.180     | 0.311  | 13   |     |
| clickhous | 0.286      | 0.063     | 0.059  | 2    |     |

With ==index_granularity = 8192; GRANULARITY=1;==  Assuming each log entry has 150 tokens, let's do the calculation. There's absolutely no problem.
[Bloom filter calculator](https://hur.st/bloomfilter/?n=&p=1.0E-2&m=30720&k=2)

So what the issue?  suddenly I recalled that, in some query statements, there were several sort items after the "order by".
## select limit

  count(gigabyte) = 2648,   count(this)=7198701  these two both taking around 12s,  roughly on the same level.  The duration is also roughly equivalent to the time it takes to scan the entire table.

- With Table Struct: ORDER BY (type, author)

```sql
SELECT * FROM hackernews_ft WHERE hasToken(lower(comment), 'gigabyte') ORDER BY type limit 50;
-- 3.153 sec

SELECT * FROM hackernews_tbf WHERE hasToken(lower(comment), 'gigabyte') ORDER BY type limit 50;
-- 2.599 sec
```

- Wrong ORDER BY CASE: it essentially haven't using the index.
```sql

SELECT * FROM hackernews_ft WHERE hasToken(lower(comment), 'gigabyte') ORDER BY type, url desc limit 50;
-- 10.285 sec
SELECT * FROM hackernews_tbf WHERE hasToken(lower(comment), 'gigabyte') ORDER BY type, url desc limit 50;
-- 8.699 sec
```

It's evident that in scenarios with a large result set, full-text search offers no advantage.

Having ruled out the index selection, I began to be convinced that the query statement's performance is too low, essentially not utilizing the index at all.

Let's create a new table to verify this.

## Test Case(Order)


```sql
CREATE TABLE hackernews_ord (    id UInt64,    deleted UInt8,    type String,    author String,    timestamp DateTime,    comment String,    dead UInt8,    parent UInt64,    poll UInt64,    children Array(UInt32),    url String,    score UInt32,    title String,    parts Array(UInt32),    descendants UInt32)ENGINE = MergeTree 
ORDER BY (toYYYYMMDD(timestamp), type, author, poll);
```

Then add index: `tokenbf_v1`, start  to `select`


**==obvious and clear at a glance==**

```sql
SELECT * FROM hackernews_ord WHERE hasToken(lower(comment), 'gigabyte') ORDER BY type limit 50;
-- 12.349 sec  not use the table struct order

SELECT * FROM hackernews_ord WHERE hasToken(lower(comment), 'gigabyte') ORDER BY timestamp limit 50;
-- 13.433 sec

SELECT * FROM hackernews_ord WHERE hasToken(lower(comment), 'gigabyte') ORDER BY toYYYYMMDD(timestamp) limit 50;
-- 2.223 sec

SELECT * FROM hackernews_ord WHERE hasToken(lower(comment), 'gigabyte') ORDER BY toYYYYMMDD(timestamp),poll limit 50;
-- 6.785 sec
```


## Conclusion

Yay, the conclusion is that the query statement didn't even make use of the primary index, while others were doubting the performance of the secondary index.