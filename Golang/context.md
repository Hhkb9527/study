# 1. 说在前面

context 是 golang 中的经典工具，主要在异步场景中用于实现并发协调控制以及对 goroutine 的生命周期管理。除此之外，context 还兼有一定的数据存储能力。本文旨在剖析 context 的核心工作原理。

本文使用到的 Go 版本为 1.18，源码位置 src/context/context.go

# 2. 场景分析

## 2.1 链式传递

在 Go 中可以认为协程的组织是一种链式传递，每一个子协程的创建都是基于父协程，但是父协程对子协程的控制则是通过 context 实现；同样的，每一个 context 也都是基于父 context 创建，最终形成链式结构，根 context 就是 emptyCtx。

## 2.2 主动取消

取消场景，是父协程任务取消的时候，将子协程一并取消。

在下面这个案例中，子协程的任务需要 2s 才能执行完，但是父协程 1s 后执行报错主动 cancel 任务的执行，cancel() 方法会通知子协程一并取消任务的执行。

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func testCtx(ctx context.Context) {
	cancelCtx, cancel := context.WithCancel(ctx)
	defer cancel()

	go func(ctx context.Context) {
		cancelCtx, cancel := context.WithCancel(ctx)
		defer cancel()

		// do something

		select {
		case <-time.After(2 * time.Second):
			// 假设任务需要 2s 完成
			fmt.Println("work done")
		case <-cancelCtx.Done():
			fmt.Println("work canceled")
			return
		}

		// do something
	}(cancelCtx)

	// 模拟执行过程中父协程报错取消任务
	<-time.After(1 * time.Second)
	err := errors.New("fake err")
	if err != nil {
		cancel()
	}
}

func main() {
	ctx := context.Background()
	testCtx(ctx)
	<-time.After(3 * time.Second)
}
```

## 2.3 任务超时

超时场景，是父协程任务超时的时候会触发取消流程，需将子协程一并取消。

在下面这个案例中，子协程的任务需要 2s 才能执行完，但是父协程 1s 后任务超时，开始执行取消任务的流程，通知子协程一并取消任务的执行。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func testCtx(ctx context.Context) {
	timerCtx, cancel := context.WithTimeout(ctx, 1*time.Second)
	defer cancel()

	go func(ctx context.Context) {
		timerCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
		defer cancel()

		// do something

		select {
		case <-time.After(2 * time.Second):
			// 假设任务需要 2s 完成
			fmt.Println("work done")
		case <-timerCtx.Done():
			fmt.Println("work canceled")
			return
		}

		// do something
	}(timerCtx)

	// 模拟等待子协程退出（偷懒）
	<-time.After(2 * time.Second)
}

func main() {
	ctx := context.Background()
	testCtx(ctx)
	<-time.After(3 * time.Second)
}
```

## 2.4 数据存储

context 可以用于数据存储，通常是用于存储一些元数据。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func testCtx(ctx context.Context) {
	valueCtx := context.WithValue(ctx, "name", "father")
	fmt.Println(valueCtx.Value("name"))

	go func(ctx context.Context) {
		valueCtx := context.WithValue(ctx, "name", "child")
		fmt.Println(valueCtx.Value("name"))
	}(valueCtx)
}

func main() {
	ctx := context.Background()
	testCtx(ctx)
	<-time.After(3 * time.Second)
}
```

# 3. 源码解读

## 3.1 一个核心数据结构

### 3.1.1 Context

首先 Context 本质是官方提供的一个 interface，实现了该 interface 定义的都被称之为 context。

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

- Deadline() 返回 ctx 生命终了时间，如果没有则 ok 为 false
- Done() 返回一个 channel 用于判断 ctx 是否已经结束
- Err() 用于 ctx 结束后获取错误信息
- Value() 获取 ctx 中存入的键值对

## 3.2 四种具体实现

### 3.2.1 emptyCtx

官方实现的一个空 ctx 版本，默认都是返回空值，通常是作为所有 ctx 的根。

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key any) any {
	return nil
}
```

### 3.2.2 cancelCtx

具有取消功能，父 ctx 取消的时候将所有子 ctx 一并取消。

> 类型定义
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // 保护临界资源
	done     atomic.Value          // chan struct{} 类型，用于判断 ctx 是否取消，第一次调用 cancel() 后 close
	children map[canceler]struct{} // 存储所有的子 ctx
	err      error                 // ctx 取消后记录错误信息
}
```

> Done() 方法实现
```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
```
- 如果 done 不为空则直接返回
- 加锁🔒
- 如果 done 为空则创建，这里也说说明一点：done 是懒加载的，第一次调用 Done() 方法才会创建 done
- 返回 done
- 解锁🔒

> Value() 方法实现
```go
var cancelCtxKey int

func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}
```
- 如果传入的 key 是 cancelCtxKey 则返回自身，这里制定了一种私有协议（外部无法访问 cancelCtxKey），用于后面判断一个父 ctx 是否是一个 cancelCtx 类型
- 否则调用 value 方法找到 key 对应的 value

> Err() 方法实现
```go
func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```
- 加锁🔒保护，返回 err

> Deadline() 方法实现
- 未实现，继承父 Context

### 3.2.3 timerCtx

具有定时取消的功能，因为是继承自 cancelCtx 所以同样具有主动取消功能

> 类型定义
```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer  // timerCtx 内部维护的一个定时器

	deadline time.Time // ctx 的终了时间
}
```

> Done() 方法实现
- 未实现，继承父 cancelCtx

> Value() 方法实现
- 未实现，继承父 cancelCtx

> Err() 方法实现
- 未实现，继承父 cancelCtx

> Deadline() 方法实现
```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```
- 返回 dealline

### 3.2.4 valueCtx

具有存储数据的功能，通常是一些元数据信息

> 类型定义
```go
type valueCtx struct {
	Context
	key, val any // 记录数据
}
```

> Done() 方法实现
- 未实现，继承父 Context

> Value() 方法实现
```go
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```
- 判断传入的 key 是否等于当前 ctx 的 k，如果相等则返回
- 否则就从父 ctx 中找，一直到 emptyCtx 找不到就返回 nil

> Err() 方法实现
- 未实现，继承父 Context

> Deadline() 方法实现
- 未实现，继承父 Context

## 3.3 六个核心方法

### 3.3.1 Background() && TODO()

用于获取 emptyCtx，本质上没有区别，仅仅是语义上的区别。

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

### 3.3.2 WithCancel()

函数说明：传入一个父 ctx 返回一个子 ctx 和一个 cancel 函数

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```
- 如果传入的 parent 是空的，panic
- 基于 parent 创建 cancelCtx
- propagateCancel 用于传递 cancel 的特性，用于保证父 ctx 取消的时候，子 ctx 也取消
- 返回创建的 cancelCtx 和一个闭包函数，闭包函数调用 cancel 取消创建的 ctx

```go
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```
- parent 是一个永远不会被取消的 ctx，直接 return
- 如果 parent 已经被取消了，把当前新创建的 cancelCtx 也取消，并 return
- 如果 parent 是一个 cancelCtx 类型则将新创建的 cancelCtx 加入到 parent 的 children 中
- 如果 parent 不是 cancelCtx 类型，则启动一个 goroutine 来保证 parent 取消的时候当前 cancelCtx 也被取消掉

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
- 前置判断
- 将 done chan 关闭
- 遍历 children，将所有的子 ctx 都一并取消掉
- 如果 parent 是 cancelCtx 类型，需要将当前 ctx 从 parent 的 children 中删除

### 3.3.3 WithDeadline()

函数说明：传入一个父 ctx，和一个终了时间，返回一个子 ctx 和一个 cancel 函数；timerCtx 继承自 cancelCtx，拥有 cancelCtx 的一切特性

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
- 如果 parent 也是一个 timerCtx 并且传入的终了时间在 parent 的终了时间之后，那么新创建的 ctx 就没必要拥有定时特性，使用 WithCancel 构造一个 cancelCtx 返回即可
- 否则创建一个 timerCtx
- propagateCancel 传递 cancel 的特性
- 如果 deadline 时间已经过了，直接 cancel 然后 return
- 创建一个定时任务，定时结束触发 cancel

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
- 调用 parent 的 cancel 函数
- 如果 parent 是 cancelCtx 类型，需要将当前 ctx 从 parent 的 children 中删除
- 定时器 stop

### 3.3.4 WithTimeout()

仅仅对 WithDeadline 进行了简单封装

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

### 3.3.5 WithValue()

```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

# 4. 一些思考

## 思考1：emptyCtx 为什么不是 struct{}类型？

struct{} 作为一个空类型并不占用底层存储空间，所以它的多个不同对象有可能会使用相同的地址，无法区分出 background 和 todo 对象。

## 思考2：backgound 和 todo 有什么区别？

本质没有区别，都是 emptyCtx，更多的是语义上的区别，background 通常作为所有 ctx 链的最顶层。

## 思考3：cancelCtx 怎么保证父亲 👨 取消的同时取消儿子 👦？

机制 1：cancelCtx 有一个 children 字段记录了所有的子节点，当父节点被取消的时候会给所有子节点来一刀 🔪，依次传递最终将所有子、孙子、孙孙子都刀 🔪 了。父亲取消的时候也会通知爷爷，让爷爷从 children 中删除父亲。

机制 2：如果父亲不是一个 cancelCtx 类型，则不会有 children 属性怎么办？当使用 WithCancel()创建的时候，发现父亲不是 cancelCtx 就会启动一个守护协程判断父亲是否 Done()，如果父亲 over 了，就会干掉儿子并退出；否则儿子先挂了，也会退出。

## 思考4：valueCtx 可以用于数据存储吗？

valueCtx 不适合视为存储介质，存放大量的 kv 数据，它的定位类似于请求头，只适合存放少量作用域较大的全局 meta 数据： 一个 valueCtx 实例只能存一个 kv 对，因此 n 个 kv 对会嵌套 n 个 valueCtx，造成空间浪费；
基于 k 寻找 v 的过程是线性的，时间复杂度 O(N)； 不支持基于 k 的去重，相同 k 可能重复存在，并基于起点的不同，返回不同的 v。
