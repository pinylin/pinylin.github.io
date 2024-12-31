---
layout: post
title: Happy New Year and ClickHouse
subtitle: 
author: pinylin
header-style: text
catalog: true
tags:
  - ClickHouse
  - ARM
  - 全文检索
---

> 新年快乐，今年的跨年我选择在酒店构建ClickHouse。顺便总结一下最近用到的ClickHouse full_text index

![](img/in-post/post-build-clickhouse.png)
## 构建arm binary

有些信创项目用的arm服务器，由于没有SSE4_2 指令，所以需要自己构建ClickHouse

As far as I know, ARM v8.0 builds are only provided for the `master` branch.

It is really simple to compile the ClickHouse code by yourself. [Here](https://clickhouse.com/docs/en/development/developer-instruction) is a tutorial. Just make sure to check out v24.8 (the last LTS version) and pass `-DNO_ARMV81_OR_HIGHER=1` to CMake.


### build

> git submodule status
> git submodule update --init --recursive


```sh
mkdir build
cd build
```

```sh
export CC=clang-18 CXX=clang++-18
cmake -DNO_ARMV81_OR_HIGHER=1 ..
```

```sh
ninja clickhouse-server clickhouse-client
```



## full_text index

- **full_text**
```sql
ALTER TABLE hackernews ADD INDEX comment_lowercase(lower(comment)) TYPE full_text;
ALTER TABLE hackernews MATERIALIZE INDEX comment_lowercase;
```

- **tokenbf index**
```sql
ALTER TABLE hackernews ADD INDEX token_lowercase(lower(comment)) TYPE tokenbf_v1(30720, 2, 0);
ALTER TABLE hackernews MATERIALIZE INDEX token_lowercase;
```

### 测试数据

测试环境是2C4G的轻量云，所以以下性能请相对参考

- no index
```
┌─database─┬─table──────┬─compressed_size─┬─uncompressed_size─┬─disk_size─┐
│ logs     │ hackernews │ 6.24 GiB        │ 11.00 GiB         │ 6.24 GiB  │ └──────────┴────────────┴─────────────────┴───────────────────┴───────────┘
```
- tokenbf
```
┌─database─┬─table──────┬─compressed_size─┬─uncompressed_size─┬─disk_size─┐
│ logs     │ hackernews │ 6.24 GiB        │ 11.00 GiB         │ 6.34 GiB  │ └──────────┴────────────┴─────────────────┴───────────────────┴───────────┘
```
- full-text
查表跟没有索引一致， 可能跟实验特性有关，
```
┌─database─┬─table──────┬─compressed_size─┬─uncompressed_size─┬─disk_size─┐
│ logs     │ hackernews │ 6.24 GiB        │ 11.00 GiB         │ 6.24 GiB  │ └──────────┴────────────┴─────────────────┴───────────────────┴───────────┘
```


```sql
EXPLAIN indexes=1
SELECT count()FROM hackernews WHERE hasToken(lower(comment), 'clickhouse');
```

case 1: 
```sql
SELECT count()FROM hackernews WHERE hasToken(lower(comment), 'clickhouse');
-- 无索引
-- 1 row in set. Elapsed: 28.060 sec. Processed 28.74 million rows, 9.75 GB (1.02 million rows/s., 347.60 MB/s.)

-- full_text
-- 1 row in set. Elapsed: 2.214 sec. Processed 4.39 million rows, 1.73 GB (1.98 million rows/s., 780.76 MB/s.)

-- tokenbf 
-- 1 row in set. Elapsed: 2.462 sec. Processed 4.46 million rows, 1.76 GB (1.81 million rows/s., 715.81 MB/s.)
```

case2:

```sql
SELECT count()FROM hackernews WHERE hasToken(lower(comment), 'clickhous');
-- 无索引 29s

-- full_text
-- 1 row in set. Elapsed: 0.063 sec. Processed 16.38 thousand rows, 5.81 MB (260.68 thousand rows/s., 92.43 MB/s.)

-- tokenbf 
-- 1 row in set. Elapsed: 0.328 sec. Processed 450.56 thousand rows, 185.22 MB (1.38 million rows/s., 565.39 MB/s.)
```

### 正则匹配

```sql
EXPLAIN indexes=1
SELECT count()FROM logs.hackernews WHERE hasToken(lower(comment), 'clickhouse');
-- 走索引 

EXPLAIN indexes=1
SELECT count()FROM logs.hackernews WHERE match(lower(comment), 'clickhou.{1}e');
-- RE2 正则表达式; EXPLAI显示走索引,SELECT好像真不走

EXPLAIN indexes=1
SELECT count()FROM logs.hackernews WHERE REGEXP_MATCHES(lower(comment), 'clickhou.e');
-- RE2 正则表达式; EXPLAI显示走索引,SELECT好像真不走

EXPLAIN indexes=1
SELECT count()FROM logs.hackernews WHERE comment regexp 'click.*ouse';
-- 不走索引 RE2

EXPLAIN indexes=1
SELECT count()FROM logs.hackernews WHERE multiMatchAny(lower(comment), ['clickhou.{1}e', 'flow']);
-- 不走索引 RE2

```

**结论**

1. 实际上只有hasToken 走索引
2. 正则查询不会走索引，开发的话建议使用 使用 match, multiMatchAny(因为match的 EXPLAI显示走索引,后面这个实验特性release后可能会支持索引)
