---
layout: post
title: hollywood(actors for Golang) source code reading Part1
subtitle: hollywood(actors for Golang) 源码阅读 Part1
author: pinylin
header-style: text
catalog: true
tags:
  - Actor
  - Golang
---
 - [workflows-as-actors-is-it-really-possible](https://temporal.io/blog/workflows-as-actors-is-it-really-possible)

>  最近看到 [hollywood](https://github.com/anthdm/hollywood)项目, 刚好之前了解过temporal, 发现 temporal跟  actor model真的好像呀。 所以有时间看看hollywood代码


- actor model

> The Actor Model is a computational model used to build highly concurrent and distributed systems. It was introduced by Carl Hewitt in 1973 as a way to handle complex systems in a more scalable and fault-tolerant manner


## Engine

actor model核心, 负责生产`actor`, 向`actor`发送消息， 终止`actor`
一个Engine 对应着 一个 post:port, 所以本地可以启动多个Engine, 参考bench代码

```go
type Engine struct {
	Registry *Registry //注册、查找 Processer
	address string     // host:prot
	remote Remoter     // 向其它Engine发送消息的设计，主要是对 Send()的抽象
	eventStream *PID   // engine本地的eventStream，也是一个Actor, 主要是接收Msg、Event 等, 然后转发到Actor
	// 说一下PID，格式: address: {host:port}, id:{strconv.Itoa(rand.Intn(math.MaxInt))}
}
```

```go
//  
type Registry struct {
	mu sync.RWMutex
	lookup map[string]Processer
	engine *Engine
}
```
## Receiver / Actor

```go
// Producer is any function that can return a Receiver
type Producer func() Receiver

// Receiver is an interface that can receive and process messages.
type Receiver interface {
	Receive(*Context)
}
```

所有`actor` 必须实现 `Receiver` `interface`.  它(Receiver func)是engine和actor沟通的接口.
所以，某种意义上，Receiver 等价与 Actor

## Process

A process is an abstraction over the actor. 
简单来说就是，先提供一个实现Receiver interface的 `p Producer`, 根据 `p Producer` 生成  `Process`, 然后 注册到 engine Registry, 并启动后, 就成了 Actor

### Spawn
很明显，是用来生产Actor的
```go
func newProcess(e *Engine, opts Opts) *process {
	pid := NewPID(e.address, opts.Kind+pidSeparator+opts.ID)
	ctx := newContext(opts.Context, e, pid)
	p := &process{
		pid: pid,
		inbox: NewInbox(opts.InboxSize),
		Opts: opts,
		context: ctx,
		mbuffer: nil,
	}
	p.inbox.Start(p)
	return p
}
```

```go
func (e *Engine) Spawn(p Producer, kind string, opts ...OptFunc) *PID {
	options := DefaultOpts(p)
	... ...
	proc := newProcess(e, options)
	return e.SpawnProc(proc)
}

func (e *Engine) SpawnProc(p Processer) *PID {
	e.Registry.add(p)
	p.Start()
	return p.PID()
}
```
### SpawnFunc
也可以直接把 `f func(*Context)` 直接传给 `e.SpawnFunc` 内部会自己实现一个 Receiver, 然后 P -> process -> Actor
```go
// SpawnFunc spawns the given function as a stateless receiver/actor.
func (e *Engine) SpawnFunc(f func(*Context), kind string, opts ...OptFunc) *PID {
	return e.Spawn(newFuncReceiver(f), kind, opts...)
}
```
## Processer
看注释，就是对 process 能力的抽象， 实现的接口的struct, 就可以被 Engine注册, 启动, 成为Actor
```go
// Processer is an interface the abstracts the way a process behaves.
type Processer interface {
	Start()
	PID() *PID
	Send(*PID, any, *PID)
	Invoke([]Envelope)
	Shutdown(*sync.WaitGroup)
}
```
## Remoter
Remoter接口是一个用于打破 `Engine` 和 `Remote` 之间循环依赖的接口。  `Engine` 需要能够向远程发送消息,但 `Remote` 也需要能够向 `Engine` 发送消息
```go
type Remoter interface {
	Address() string
	Send(*PID, any, *PID)
	Start(*Engine) error
	Stop() *sync.WaitGroup
}

type Remote struct {
	addr string
	engine *actor.Engine
	config Config
	streamReader *streamReader
	streamRouterPID *actor.PID
	stopCh chan struct{} // Stop closes this channel to signal the remote to stop listening.
	stopWg *sync.WaitGroup
	state atomic.Uint32
}
```

如下，初始化一个 有 `Remoter` 的 Engine, 在 NewEngine 中 `Remoter`会 自己`Start()`建立连接
```go
rem := remote.New(*listenAt, remote.NewConfig())
actor.NewEngine(actor.NewEngineConfig().WithRemote(rem))
```

在`Start()` 中 还Spawn 了 一个 Actor `streamRouterPID`
```go
func (r *Remote) Start(e *actor.Engine) error {
// ... ...
r.streamRouterPID = r.engine.Spawn(
	newStreamRouter(r.engine, r.config.TLSConfig),
	"router", actor.WithInboxSize(1024*1024))
```


### 向Remote Engine发送一条消息的过程

向Remote 发送Msg, 我们首先确定一下 本地的资源
- Engine  : NewEngine WithRemote
- serverPID ->  `*PID`  remote 地址
- clientPID :  `*PID` 必须有 本地Actor

现在开始发送

1. 直接用调用Engine 发送消息
```go
	e.SendWithSender(serverPID, msg, clientPID)

		// 1. 如果serverPID是本地pid, 
		e.SendLocal(pid, msg, sender)
		// 2. 如果remote == nil 
		e.BroadcastEvent(EngineRemoteMissingEvent{
		// 3. remote
		e.remote.Send(pid, msg, sender)
```
2. 如果需要Send 到 remote 的Msg, 会 route 到 `streamRouterPID` 对应的Actor
```go
func (r *Remote) Send(pid *actor.PID, msg any, sender *actor.PID) {
	r.engine.Send(r.streamRouterPID, &streamDeliver{
		target: pid,
		sender: sender,
		msg:    msg,
	})
}
```
3.  `streamRouter` Actor, `streamRouterPID` 对应的Actor,  就是 实现了 Receive 的 `streamRouter`
```go
// ... ...
func (s *streamRouter) Receive(ctx *actor.Context) {
	switch msg := ctx.Message().(type) {
	case actor.Started:
		s.pid = ctx.PID()
	case *streamDeliver:
		s.deliverStream(msg)
	case terminateStream:
		s.handleTerminateStream(msg)
	}
}
// in deliverStream
swpid = s.engine.SpawnProc(newStreamWriter(s.engine, s.pid, address, s.tlsConfig))
```

4. `streamWriter` 也是 一个Actor, Receive 消息之后 放入Inbox, 然后通过 Inbox.run() 发送到remote

```go
func (in *Inbox) run() {
			in.proc.Invoke(msgs)
}

func (s *streamWriter) Invoke(msgs []actor.Envelope) {
	if err := s.stream.Send(env); err != nil {}
}
```


### 总结
1. e.SendWithSender
2. `streamRouter` Actor
3. `streamWriter` Actor -> Inbox -> Inbox.run() -> s.stream.Send
