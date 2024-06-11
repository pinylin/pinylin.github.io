---
layout: post
title: WaitGroup In Rust
author: pinylin
header-style: text
catalog: true
tags:
  - Rust
  - WaitGroup
  - TaskTracker
  - no_std
subtitle:
---
Golang的WaitGroup非常直观好用, 那么Rust中有类似的Crates吗？当然, 最近在社区就看到了

## wg

支持同步, 异步(不依赖特定运行时), no_std环境的的使用方法像Golang一样的WaitGroup
```rust
#[tokio::main]
async fn main() {
    let wg = AsyncWaitGroup::new();
    for _ in 0..5 {
        let t_wg = wg.add(1);
        spawn(async move {
            // mock task ...
            t_wg.done();
        });
    }
    wg.wait().await;
}
```

## TaskTracker

tokio官方, 如果你用tokio runtime的话, 更推荐这个
```rust
use tokio_util::task::TaskTracker;

#[tokio::main]
async fn main() {
    let tracker = TaskTracker::new();

    for i in 0..10 {
        tracker.spawn(async move {
            println!("Task {} is running!", i);
        });
    }
    // Once we spawned everything, we close the tracker.
    tracker.close();

    // Wait for everything to finish.
    tracker.wait().await;

    println!("This is printed after all of the tasks.");
}
```

## bench
写个简单的bench, 就用wg的demo, 结果如下, 看来作者还是很有实力的嘛, 值得看一看源码. 
所以, 在同步或其它runtime, no_std 环境, wg 确实很有用, 点赞

```sh
test tests::tt_benchmark ... bench:  52,789,960 ns/iter (+/- 534,919)
test tests::wg_benchmark ... bench:  52,433,269 ns/iter (+/- 315,961)
```


