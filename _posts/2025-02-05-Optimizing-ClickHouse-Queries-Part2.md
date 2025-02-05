---
layout: post
title: Optimizing ClickHouse Queries
subtitle: Full Text Search By Quickwit
author: pinylin
header-style: text
catalog: true
tags:
  - ClickHouse
  - 全文检索
  - index
  - quickwit
---

> From the previous tests (as mentioned in the last post), we discovered that primary problem is the table's `TableSortKey`,  which failed to use the primary index (🥬🥬🥬)

After optimizing the table structure, the `full_text` search still too slow to meet the requirements.  as can be seen from the test case in the previous post. The full_text index in ClickHouse does not perform significantly better than `tokenbf_v1`, yet it  disk space nearly up to 40% of the table's (data).


So, my choice is:  **quickwit**: [add-full-text-search-to-your-olap-db](https://quickwit.io/docs/guides/add-full-text-search-to-your-olap-db)

## Test Case

Still using the previous HackerNews test data

- hackernews.json
```

```

### Create a Quickwit index

```yaml
version: 0.7

index_id: hackernews

doc_mapping:
  field_mappings:
    - name: Time
      type: datetime
      input_formats:
        - unix_timestamp
      output_format: unix_timestamp_secs
      fast_precision: seconds
      fast: true
    - name: Id
      type: u64
      true: true
    - name: Text
      type: text
      tokenizer: en_stem  # 使用英文分词器（带词干提取）
  tag_fields: [Id]
  timestamp_field: Time

search_settings:
  default_search_fields: [Text]
indexing_settings:
  commit_timeout_secs: 30  # 索引提交超时时间
```

### Index Hackernews

```sh
quickwit index create --index-config hackernews-index.yaml
quickwit index ingest --input-path hackernews.json --index hackernews
```

- test quickwit index 
```sh
curl "http://127.0.0.1:7280/api/v1/hackernews/search/stream?query=tantivy&output_format=csv&fast_field=Id"
```


## Use Quickwit search inside ClickHouse


1. The fields(used by search stream) must be configured as fast_field in the Quickwit index.
```yaml
# eg. Id
    - name: Id
      type: u64
      fast: true
```

2. The fast_field column should be the `TableSortKey` of the  Table; otherwise, it is essentially unavailable.


## Search Performance

The official statement from Quickwit:

> The search stream endpoint is powerful enough to stream 100 million ids to ClickHouse in less than 2 seconds on a multi TB dataset. And you should be comfortable playing with search stream on even bigger datasets.


Through testing, it’s indeed the case. The Hackernews test data, which consists of up to 28 million rows and an 18GB JSON file on disk, shows that most queries complete in under 0.2 seconds.

## Our solution

Although Quickwit's Search Stream performance is quite impressive and can support TB datasets.  But, as mentioned above, the fast_field in search stream URL, must be Table's **TableSortKey**

Our log table already has `ORDER BY (created)`, and our business scenario requires pagination.

So, the final solution is:
1. Forget  the Search Stream API and use the Search API (`start_offset`, `sort_by`) for paginated queries.
2. Change the `created` field from `DateTime` to `UInt64`, and modify the query statements to use corresponding timestamps.
3. Based on the paginated results, use `WHERE created IN ()` and `LIKE` for querying.

This approach will be slightly slower than using Search Stream, but it fully meets our business requirements.