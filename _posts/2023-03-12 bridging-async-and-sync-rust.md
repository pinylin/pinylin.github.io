---
layout: post
title: bridging-async-and-sync-rust
author: pinylin
header-style: text
catalog: true
tags:
  - Rust
  - async
---


> refer: [bridging-async-and-sync-rust]([Bridging Async and Sync Rust Code - A lesson learned while working with Tokio | Greptime](https://greptime.com/blogs/2023-03-09-bridging-async-and-sync-rust))

```rust
impl Sequencer for PlainSequencer {
    fn generate(&self) -> Vec<i32> {
        let bound = self.bound;
        futures::executor::block_on(async move {
            RUNTIME.spawn(async move {
                let mut res = vec![];
                for i in 0..bound {
                    res.push(i);
                    tokio::time::sleep(Duration::from_millis(100)).await;
                }
                res
            }).await.unwrap()
        })
    }
}

```

## Best practice

Based on the above analysis and Rust's generator-based cooperative asynchronous characteristics, we can summarize some tips when bridging asynchronous and synchronous code in Rust:

- Combining asynchronous code with synchronous code that can cause blocking is never a wise choice.
- When calling asynchronous code from a synchronous context, use `futures::executor::block_on` and spawn the async code to a dedicated runtime, because the former will block the current thread.
- On the other hand, if you have to call blocking synchronous code from an asynchronous context, it is recommended to use `tokio::task::spawn_blocking` to execute the code on a dedicated executor that handles blocking operations.