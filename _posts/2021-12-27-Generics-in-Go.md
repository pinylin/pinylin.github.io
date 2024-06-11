---
layout: post
title: Generics in Go
subtitle: Go 泛型
author: pinylin
header-style: text
catalog: true
tags:
  - Golang
  - Generics
---

> [gotip playground](https://gotipplay.golang.org/)
> [generics-proposal](https://go.dev/blog/generics-proposal)
> https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md
> https://github.com/golang/go/issues/45458
> https://colobu.com/2021/08/30/how-is-go-generic-implemented/

```sh
$ go install golang.org/dl/gotip@latest

$ gotip download
```

Generics 是指把类型抽象成一种 “参数” ， 数据和算法都针对这种抽象的类型参数(type parameter)来实现，而不是针对具体类型。当我们真正使用的时候，再具体化，实例化类型参数


# High level overview

- Function + type parameter list : `func F[T any](p T) { ... }`
- Type + type parameter list : `type M[T any] []T`
- Each type parameter has a type constraint: `func F[T Constraint](p T) { ... }`
    
- Type constraints are interface types
    
- The new predeclared name `any` is a type constraint that permits any type.
    
- Type parameter set
    
    - `T` : restricts that type
        
    - `~T` : restricts to all types whose underlying type is `T`. 底层类型是T
        
    - `T1 | T2 | ...`restricts to any of the listed elements
        
- Generic functions may only use operations supported by all the types permitted by the constraint.

泛型函数只可以使用 所有符合泛型约束的类型支持操作

- Using a generic function or type requires passing type arguments.
- Type inference permits omitting the type arguments of a function call in common cases.


# Background

- Why do we need generic types
	https://groups.google.com/g/golang-nuts/c/j_3n5wAZaXw/m/YkOdbCppAQAJ?pli=1
	https://github.com/golang/go/blob/master/src/math/dim.go#L35

- This design explicitly defined structural constraints. Not by a declared subtyping relationship
- This design does not support template metaprogramming or any other form of compile time programming.
- 蜡印(**stenciling**) as rust

# Design

## Type parameters

```go
// Print prints the elements of any slice.
// Print has a type parameter T and has a single (non-type)
// parameter s which is a slice of that type parameter.
func Print[T any](s []T) {
	for _, v := range s {
		fmt.Println(v)
	}
}
```

- Type inference(类型推断)

```go
        // pass a type argument explicitly
        Print[int]([]int{1, 2, 3})
        
        // type inference
        Print([]int{1, 2, 3})

```

## Constraints(约束？)

```
// This function is INVALID.
func Stringify[T any](s []T) (ret []string) {
        for _, v := range s {
                ret = append(ret, v.String()) // INVALID
        }
        return ret
}
```

- Any 可以被用在？
- The any constraints

```
func Print[T interface{}](s []T) { /*...*/ }
// same as 
func Print[T any](s []T) { /*...*/ }
```

- Defining & Use a constraint

```
type Stringer interface {
        String() string
}

func Stringify[T Stringer](s []T) (ret []string) {
        for _, v := range s {
                ret = append(ret, v.String())
        }
        return ret
}
```

## Multiple type parameters

```
// Print2 has two type parameters and two non-type parameters.
// may be slices of two different types
func Print2[T1, T2 any](s1 []T1, s2 []T2) { ... }

// Print2Same has one type parameter and two non-type parameters.
// must be slices of the same element type
func Print2Same[T any](s1 []T, s2 []T) { ... }
```

Each type parameter may have its own constraint. / 每个类型参数都可以有自己的约束

```
type Stringer interface {
        String() string
}

type Plusser interface {
        Plus(string) string
}

func ConcatTo[S Stringer, P Plusser](s []S, p []P) []string {
        r := make([]string, len(s))
        for i, v := range s {
                r[i] = p[i].Plus(v.String())
        }
        return r
}
```

当然，一个 constraint 可以用在多个 type parameters.

```
func Stringify2[T1, T2 Stringer](s1 []T1, s2 []T2) string {
        r := ""
        for _, v1 := range s1 {
                r += v1.String()
        }
        for _, v2 := range s2 {
                r += v2.String()
        }
        return r
}
```

## Generic types

```
type Vector[T any] []T

// to use; refer to the same "Vector[int]" type.
type Vector[int] []int  
var v Vector[int]

```

Generic types(泛型类型) 可以有自己的(method)方法, 方法声明不需要写constraint(约束)

```
func (v *Vector[T]) Push(x T) { *v = append(*v, x) }
```

方法声明中列出的类型参数不需要与类型声明中的类型参数的名称一致, 如果方法中用不到，可以写做 _

- 在某个类型可以指向自己的时候，它的泛型也可以指向自身。但是，注意类型参数的顺序

```
type List[T any] struct {
        next *List[T] // this reference to List[T] is OK
        val  T
}

// This type is INVALID.
type P[T1, T2 any] struct {
        F *P[T2, T1] // INVALID; must be [T1, T2]
}
```

间接引用也适用，

```
type ListHead[T any] struct {
        head *ListElement[T]
}

type ListElement[T any] struct {
        next *ListElement[T]
        val  T
        // Using ListHead[T] here is OK.
        // ListHead[T] refers to ListElement[T] refers to ListHead[T].
        head *ListHead[T]
}
```

Generic type 的类型参数，应该 使用any之外的约束(constraints). 上边只是示例

### 泛型类型的 方法不可以有额外的类型参数

## Operators

```
// This function is INVALID.
func Smallest[T any](s []T) T {
        r := s[0] // panic if slice is empty
        for _, v := range s[1:] {
                if v < r { // INVALID
                        r = v
                }
        }
        return r
}

type A []struct{}
```

除了我们将在后面讨论的两个例外(==, !=)情况，所有由语言定义的算术、比较和逻辑运算符只能用于预先声明的类型，或者用于底层类型是这些预先声明的类型之一的定义类型。

### Type set [Type set & approximation element](https://h42oe8b2o9.feishu.cn/docs/doccnEIaoei7cxvvqiEnYi71LXe)

### Type set of constraints

### Constraint elements

### Operations based on type sets

https://github.com/golang/go/blob/master/src/constraints/constraints.go

Type set 的目的是允许泛型函数使用运算符，比如<

```go
// Ordered is a constraint that permits any ordered type: any type
// that supports the operators < <= >= >.
// If future releases of Go add new ordered types,
// this constraint will be modified to include them.
type Ordered interface {
        Integer | Float | ~string
}
```

### Comparable types in constrains

`==` and `!=`

[Comparable](https://github.com/golang/go/blob/766f89b5c625f8c57492cf5645576d9e6f450cc2/src/builtin/builtin.go#L102)是内建类型，不支持用户实现这个constraint

Comparable 的 type set 包含 struct, array, and interface types. 没有func, 所以会有这个[问题](https://github.com/golang/go/issues/49587)。也没有slice

```go
// Index returns the index of x in s, or -1 if not found.
func Index[T comparable](s []T, x T) int {
        for i, v := range s {
                // v and x are type T, which has the comparable
                // constraint, so we can use == here.
                if v == x {
                        return i
                }
        }
        return -1
}
```

Since `comparable` is a constraint, it can be embedded in another interface type used as a constraint.

```go
// ComparableHasher is a type constraint that matches all
// comparable types with a Hash method.
type ComparableHasher interface {
        comparable
        Hash() uintptr
}
```

## Mutually referencing type parameters / 相互引用的类型参数

```
package graph

type NodeConstraint[Edge any] interface {
        Edges() []Edge
}

type EdgeConstraint[Node any] interface {
        Nodes() (from, to Node)
}

type Graph[Node NodeConstraint[Edge], Edge EdgeConstraint[Node]] struct { ... }

func New[Node NodeConstraint[Edge], Edge EdgeConstraint[Node]] (nodes []Node) *Graph[Node, Edge] {
        ...
}
```

```
// Vertex is a node in a graph.
type Vertex struct { ... }

// Edges returns the edges connected to v.
func (v *Vertex) Edges() []*FromTo { ... }

// FromTo is an edge in a graph.
type FromTo struct { ... }

// Nodes returns the nodes that ft connects.
func (ft *FromTo) Nodes() (*Vertex, *Vertex) { ... }

// 实例化一个graph
var g = graph.New[*Vertex, *FromTo]([]*Vertex{ ... })
```

## Type inference

## _Omissions_

- No specialization / 无特化，不能为一个generic function 根据不同的type arguments 实现特殊的版本

- No metaprogramming / 无元编程/宏
- No higher level abstraction. 无更高层次的抽象

There is no way to use a function with type arguments other than to call it or instantiate it. There is no way to use a generic type other than to instantiate it.

- No general type description.
    - Constraints使用 type set, 而不是描述一个类型必须具有哪些特征
- No covariance or contravariance of function parameters. / 无协变或逆变
- No operator methods.
- No currying. / 无柯里化 func(a,b,c T) -> func(a T) -> func(b T) -> func(c T)
- No variadic type parameters. / 不支持可变的类型参数
- No adaptors. 没有适配器
- No parameterization on non-type values such as constants. / 不支持不是类型的变量(比如常量)作为实参

## _Issues_ [link](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#issues)