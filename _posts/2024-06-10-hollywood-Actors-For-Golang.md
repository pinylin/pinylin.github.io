---
layout: post
title: hollywood(actors for Golang) source code reading
subtitle: hollywood(actors for Golang) 源码阅读
author: pinylin
header-style: text
catalog: true
tags:
  - Actor
  - Golang
---

- actor model

> The Actor Model is a computational model used to build highly concurrent and distributed systems. It was introduced by Carl Hewitt in 1973 as a way to handle complex systems in a more scalable and fault-tolerant manner


## Engine

actor model核心, 负责生产`actor`, 向`actor`发送消息， 终止`actor`

```Go
type Engine struct {
	Registry *Registry
	address string
	remote Remoter
	eventStream *PID
}
```
## Reciver/Actor

