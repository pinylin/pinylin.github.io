---
layout: post
title: hollywood(actors for Golang) source code reading Part2
subtitle: hollywood(actors for Golang) 源码阅读 Part2
author: pinylin
header-style: text
catalog: true
tags:
  - Actor
  - Golang
---
> 上篇看了基本概念，今天来尝试了解一下Actor的调度问题，看下`ULTRA fast actor engine`
> 到底是如何实现的

首先从收到消息开始，

# Actor 是如何Recv 消息的
## Loacl
如果是发送给Engine本地的`Actor`
1.  e.Registry.get(pid),  获取`Processer`
2. 如果没有`Processer`，发送给`DeadLetter process`
3. `DeadLetter process` 也没有， panic
```Go
// SendLocal will send the given message to the given PID. If the recipient is not found in the
// registry, the message will be sent to the DeadLetter process instead. If there is no deadletter
// process registered, the function will panic.
func (e *Engine) SendLocal(pid *PID, msg any, sender *PID) {
	proc := e.Registry.get(pid)
	if proc == nil {
		// broadcast a deadLetter message
		e.BroadcastEvent(DeadLetterEvent{
			Target:  pid,
			Message: msg,
			Sender:  sender,
		})
		return
	}
	proc.Send(pid, msg, sender)
}
```

### 消息写入了Inbox
无论是Local还是Remote, 收到消息后，最后都走到了process.Send，然后 `Inbox.Send`

```Go
func (p *process) Send(_ *PID, msg any, sender *PID) {
	p.inbox.Send(Envelope{Msg: msg, Sender: sender})
}
```

```Go
func (in *Inbox) Send(msg Envelope) {
	in.rb.Push(msg)
	in.schedule()
}
```

我们来详细看下 Inbox
```Go
type Inbox struct {
	// 虽然README写的超出size后会block, 但代码确实是resize后push
	rb         *ringbuffer.RingBuffer[Envelope]  // 环形队列, buffer resize规则是*2
	proc       Processer
	scheduler  Scheduler // 
	procStatus int32
}
```

## Remote

上篇说到了有Remote的Engine 会有两个 新的Actor 
1. `streamRouter` Actor
2. `streamWriter` Actor 
其实还有第三个 
3. `streamReader` Actor
`streamWriter` 调用底层的 `drpc` 发送， 然后 `streamReader` 这边的 `drpc` `/remote.Remote/Receive` 收到 `envelope`(这个概念就是Msg的封装，加了发送和接收的`*actor.PID`)  

```Go
type DRPCRemote_ReceiveStream interface {
	drpc.Stream
	Send(*Envelope) error
	Recv() (*Envelope, error)
}
```

下面来看一下 `streamReader` Actor 的 `Receive`,  r.remote.engine 就是本地Engine, 后面的逻辑和 上面的 Local 一致
```Go
func (r *streamReader) Receive(stream DRPCRemote_ReceiveStream) error {
	for {
		envelope, err := stream.Recv()
		for _, msg := range envelope.Messages {
			// ... ...
			
			r.remote.engine.SendLocal(target, payload, sender)
		}
	}
	return nil
}
```


# Actor收到消息之后

上面说到，消息最后是被写入了 `Inbox`，后续对 `Inbox` 中消息的处理是如何进行的呢？
处理消息的方法如下, 最后会执行 `process` 实现的`Receive` 方法
```Go
func (p *process) Invoke(msgs []Envelope) {
	p.invokeMsg(msg)
}
func (p *process) invokeMsg(msg Envelope) {
		recv.Receive(p.context)
}
```

调起 `Invoke` 方法的地方有两处
## 1. process.Start() 
在 process 注册到 Registry 后， 会执行 p.Start()，里面有 p.Invoke(msgs), 这里主要的作用是
process刚启动注册，先执行一下Inbox里所有的缓存Msgs

## 2. Inbox  收到消息之后

```Go
func (in *Inbox) Send(msg Envelope) {
		in.rb.Push(msg)  // 先向rb push msg
		in.schedule()    // 如果in.procStatus 是 idle, 改为 running
}
func (in *Inbox) schedule() {
	if atomic.CompareAndSwapInt32(&in.procStatus, idle, running) {
		// procStatus 改动之后, 开始执行 in.process
		in.scheduler.Schedule(in.process)
		// 这个 就是把 传入的 fn ->  go fn()
		// func (goscheduler) Schedule(fn func()) { go fn() }
}

	}
}
func (in *Inbox) process() {
	in.run()
	// 本次执行完成，in.procStatus 变为 idle
	atomic.StoreInt32(&in.procStatus, idle) 
}
```

重点在 Inbox.run() 

```Go
func (in *Inbox) run() {
	// Throughput 项目默认300， 也就是说, 每执行300次in.rb.PopN 后会主动调用
	// runtime.Gosched() 让出processor(GMP: P)
	i, t := 0, in.scheduler.Throughput()
	for atomic.LoadInt32(&in.procStatus) != stopped {
		if i > t {
			i = 0
			runtime.Gosched()
		}
		i++

		if msgs, ok := in.rb.PopN(messageBatchSize); ok && len(msgs) > 0 {
			in.proc.Invoke(msgs)
		} else {
			return
		}
	}
}
```

### 总结

有些理解作者 宣称 `Blazingly fast, low latency actors for Golang` 的原因，因为
1. Actor 从 streamReader(drpc) 收到消息，到写入 `process` 的 `Inbox` 是同步的， 这一步很难在提升了，
	- 以后优化的方向可能就是 QUIC, 还有更快的 序列化/反序列化 方案 这两个方向
2. `Inbox` 后立即执行  `Schedule(in.process)` 是一个新的Goroutine，从ringBuffer读取msgs进行处理， 没有多余的调度逻辑，就是在每次`Inbox.Send` 后执行, 简洁又强大，长时间执行的话，又有主动让出 `processor(GMP: P)` 的设计，感觉很合理
	-  除了golang runtime 对 Goroutine，这块感觉已经是零成本抽象了
