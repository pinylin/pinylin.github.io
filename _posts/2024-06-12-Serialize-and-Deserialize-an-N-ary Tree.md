---
layout: post
title: Serialize and Deserialize an N-ary Tree
subtitle: 多叉树的序列化与反序列化
author: pinylin
header-style: text
catalog: true
tags:
  - Tree
  - Serialize
  - Deserialize
  - Pool
  - Goroutine
---

> 今天面试，碰到了比较开放性的两个问题，如果当时有纸有笔可以写写画画的话，可能就碰巧碰出火花了，人脑缓存不够，深表遗憾

# Serialize and Deserialize an N-ary Tree

本来的题目是N 没有限制，但是在具体实现上child 用array 存储的话，没差别

具体参考: [Serialize and Deserialize an N-ary Tree](https://www.geeksforgeeks.org/serialize-deserialize-n-ary-tree/)   

blog 里没有Go实现我来补充一下
```go
package main  
  
import (  
   "bufio"  
   "fmt"   
   "os"
)  
  
type Node struct {  
   key    rune  
   Childs []*Node  
}  
  
func newNode(key rune) *Node {  
   return &Node{key: key, Childs: make([]*Node, 0)}  
}  
  
func serialize(root *Node, writer *os.File) {  
   if root == nil {  
      return  
   }  
   fmt.Fprintf(writer, "%c", root.key)  
   for _, child := range root.Childs {  
      serialize(child, writer)  
   }  
   fmt.Fprintf(writer, "%c", ')')  
}  
  
func deserialize(reader *bufio.Reader) *Node {  
   val, _, err := reader.ReadRune()  
   //slog.Warn("%s", val)  
   if err != nil || val == ')' {  
      return nil  
   }  
   root := newNode(rune(val))  
   for {  
      child := deserialize(reader)  
      if child == nil {  
         break  
      }  
      root.Childs = append(root.Childs, child)  
   }  
   return root  
}  
  
func createDummyTree() *Node {  
   root := newNode('A')  
   root.Childs = append(root.Childs, newNode('B'), newNode('C'), newNode('D'))  
   root.Childs[0].Childs = append(root.Childs[0].Childs, newNode('E'), newNode('F'))  
   root.Childs[2].Childs = append(root.Childs[2].Childs, newNode('G'), newNode('H'), newNode('I'), newNode('J'))  
   root.Childs[0].Childs[1].Childs = append(root.Childs[0].Childs[1].Childs, newNode('K'))  
   return root  
}  
  
func traverse(root *Node) {  
   //fmt.Println(fmt.Sprintf("%+v", root))  
   if root != nil {  
      fmt.Printf("%c ", root.key)  
      //slog.Warn("%c", root.key)  
      for _, child := range root.Childs {  
         traverse(child)  
      }  
   }  
}  
  
func main() {  
   root := createDummyTree()  
   file, err := os.Create("tree.txt")  
   if err != nil {  
      fmt.Println("Could not open file")  
      return  
   }  
   defer file.Close()  
   serialize(root, file)  
  
   //file, err := os.Open("tree.txt")  
   //if err != nil {   // fmt.Println("Could not open file")   // return   //}   //defer file.Close()   r := bufio.NewReader(file)  
   root1 := deserialize(r)  
   fmt.Println("Constructed N-Ary Tree from file is")  
   traverse(root1)  
}
```


# Goroutine Pool

背景： 是需要并发的对上面的Tree (10W+节点进行业务处理)，所以会需要 G Pool，需求确实合理

## Actor 
首先，最容易想到的就是前两天刚看的`Actor model`，只需要限制  `Actor` 的数量，然后将Msgs 负载均衡到 发送到 `Actor`，就能达到限制Goroutine 并发数量和重复利用Goroutine(这里Actor资源) 的目标

## 两个channel

这个最简单，应该是能想到的，毕竟之前循环打印数字和字母也是两个chan互相喂饭，，，脑子转不过来，下次还是自带纸笔吧


```go
type Pool struct {  
   ctrl chan struct{}  // 控制并发Goroutine 数量
   work chan func()    // 需要执行的task chan
}  

func NewPool(size, queue, spawn int) *Pool {  
   if spawn <= 0 && queue > 0 {  
      panic("dead queue configuration detected")  
   }
   if spawn > size {  
	   panic("spawn > workers")  
	}
   p := &Pool{  
      ctrl:  make(chan struct{}, size),  
      work: make(chan func(), queue),  
   }  
   for i := 0; i < spawn; i++ {  // 这里是先初始化spawn个Goroutine  warmup
      p.ctrl <- struct{}{}  
      go p.worker(func() {})  
   }  
   return p  
}

func (p *Pool) Schedule(task func()) error {  
   select {  
   case p.work <- task:         // task push到 work chan
		return nil
   case p.ctrl <- struct{}{}:    // 这里是为了控制warmup后，G的数量最后还是能达到size
		go p.worker(task)        // init worker的时候, 都会先写入令牌
		return nil
   }  
}

func (p *Pool) worker(task func()) {  
	defer func() { <-p.ctrl }()  // worker退出时，释放令牌
	task()  
	for task := range p.work {  
		task()  
	}
}
```

### 补一下benchmark
确实有提升，，，
```shell
BenchmarkNormalGoroutine
BenchmarkNormalGoroutine-8       1712344               694.2 ns/op
BenchmarkPool
BenchmarkPool-8                  2194712               599.1 ns/op

```

```go
func BenchmarkNormalGoroutine(b *testing.B) {  
   b.ResetTimer()  
   for i := 0; i < b.N; i++ {  
      finish := make(chan struct{})  
      go func() {  
         for k := 0; k < 100; k++ {  
            _ = k  
         }  
         finish <- struct{}{}  
      }()  
      <-finish  
   }  
}  
  
func BenchmarkPool(b *testing.B) {  
   p := NewPool(100, 60, 80)  
   //defer p.Close()  没实现
   b.ResetTimer()  
   for i := 0; i < b.N; i++ {  
      p.Schedule(func() {  
         for k := 0; k < 100; k++ {  
            _ = k  
         }  
      })  
   }  
}
```