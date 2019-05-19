---
title: Go语言context包学习笔记
date: "2019-05-16"
description: ""
url: /blog/go/context/
image: "/blog/go/context.png"
---
Go语言中的context是一个很常用的包，但想用好，也需要深入学习一下

<!--more-->
## 适用场景
处理单个请求的多个goroutine之间与请求域的数据、取消信号、截止时间等相关操作。
> 例如：在Go服务器程序中，每个请求都会有一个goroutine去处理。然而，处理程序往往还需要创建额外的goroutine去访问后端资源，比如数据库、RPC服务等。由于这些goroutine都是在处理同一个请求，所以它们往往需要访问一些共享的资源，比如用户身份信息、认证token、请求截止时间等。而且如果请求超时或者被取消后，所有的goroutine都应该马上退出并且释放相关的资源。这种情况就需要用Context来为我们取消掉所有goroutine

P.S
如果只是为了处理并发程序中的，由于超时，取消或者其他部分的故障需要的抢占操作，通过done channel的方式似乎也可以完成，让该channel 在程序中流动并取消所有阻塞的并发操作。context可以看作它的加强版，附加额外传递的信息：为什么取消，或者是否需要最后的完成期限（超时）

## 原理
### 结构
Context是一个interface，在golang里面，interface是一个使用非常广泛的结构，它可以接纳任何类型。Context定义很简单，一共4个方法，我们需要能够很好的理解这几个方法
``` go
type Context interface {
  // Deadline方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。
	Deadline() (deadline time.Time, ok bool)
  // Done方法返回一个只读的chan，类型为struct{}，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过Done方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。之后，Err 方法会返回一个错误，告知为什么 Context 被取消。
	Done() <-chan struct{}
  // Err方法返回取消的错误原因，因为什么Context被取消。
	Err() error
  // Value方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。
	Value(key interface{}) interface{}
}
```


### 使用方式
#### 直接使用
Context 虽然是个接口，但是并不需要使用方实现，golang内置的context 包，已经帮我们实现了2个方法，一般在代码中，开始上下文的时候都是以这两个作为最顶层的parent context，然后再衍生出子context。这些 Context 对象形成一棵树：当一个 Context 对象被取消时，继承自它的所有 Context 都会被取消。两个实现如下：
``` go
var (
	background = new(emptyCtx)
	todo = new(emptyCtx)
)

func Background() Context {
	return background

}

func TODO() Context {
	return todo
}
```
一个是Background，主要用于main函数、初始化以及测试代码中，作为Context这个树结构的最顶层的Context，也就是根Context，它不能被取消。
一个是TODO，如果我们不知道该使用什么Context的时候，可以使用这个。
他们两个本质上都是emptyCtx结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。


#### context继承
有了如上的根Context，那么是如何衍生更多的子Context的呢？这就要靠context包为我们提供的With系列的函数了。

``` go
// WithCancel函数，传递一个父Context作为参数，返回子Context，以及一个取消函数用来取消Context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// WithDeadline函数，和WithCancel差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消Context，当然我们也可以不等到这个时候，可以提前通过取消函数进行取消
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

// WithTimeout和WithDeadline基本上一样，这个表示是超时自动取消，是多少时间后自动取消Context的意思
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// WithValue函数和取消Context无关，它是为了生成一个绑定了一个键值对数据的Context，这个绑定的数据可以通过Context.Value方法访问到，这是我们实际用经常要用到的技巧，一般我们想要通过上下文来传递数据时，可以通过这个方法，如我们需要tarce追踪系统调用栈的时候
func WithValue(parent Context, key, val interface{}) Context
```

通过上面的结构和方法我们能明白，context主要有两个目的：
* 提供一个可以取消你的调用图中分支的API
* 提供用于通过呼叫传输请求范围数据的数据包

下面我们分别从这两个方面介绍context的各个方法

## 使用方式
### 取消
函数中的取消有三个方面：
* goroutine的父goroutine可能想要取消它
* 一个goroutine可能想要取消它的子goroutine
* goroutine中的任何阻塞操作都必须是可抢占的，以便它可以被取消

context包帮助管理所有这三个东西

> 注意：</br>context类型将是你的函数的第一个参数。如果你看看context接口上的方法，会发现没有任何东西可以改变底层结构的状态，接收context的函数并不能取消它。这避免了调用堆栈上的函数被子函数取消上下文的情况。</br>这就产生了一个问题：如果context是不可变的，那我们如何影响调用堆栈中当前函数下面的函数的取消 ？

这就要提高之前说的三个函数了
``` go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

所有这些函数都接受一个context参数，并返回一个新的context。将新的context传递给子元素。通过这种方式，调用图的连续图层可以创建符合需求的上下文，而不会影响其父母节点。

例如
``` go
package main

import (
	"fmt"
	"sync"
	"time"

	"golang.org/x/net/context"
)

var (
	wg sync.WaitGroup
)

func work(ctx context.Context) error {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		select {
		case <-time.After(2 * time.Second):
			fmt.Println("Doing some work ", i)

		// we received the signal of cancelation in this channel
		case <-ctx.Done():
			fmt.Println("Cancel the context ", i)
			return ctx.Err()
		}
	}
	return nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()

	fmt.Println("Hey, I'm going to do some work")

	wg.Add(1)
	go work(ctx)
	wg.Wait()

	fmt.Println("Finished. I'm going home")
}
```

### 存储数据
context的另一半功能是：用于存储和检索请求范围数据的context的数据包。
``` go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	valueCtx := context.WithValue(ctx, key, "add value")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			//get value
			fmt.Println(ctx.Value(key), "is cancel")

			return
		default:
			//get value
			fmt.Println(ctx.Value(key), "int goroutine")

			time.Sleep(2 * time.Second)
		}
	}
}

```
> 注意</br>* 你使用的键值必须满足Go语言的可比性概念，也就是运算符 == 和 != 在使用时需要返回正确的结果</br>* 返回值必须安全，才能从多个goroutine访问</br></br>由于context的键和值都被定义为interface{},所以当试图检索值时，我们会失去Go语言的类型安全性

Go语言作者建议你在从Context中存储和检索值时遵循一些规则
* 自定义键类型。防止其他软件包冲突
* 仅将上下文值用于传输进程和请求的请求范围数据，API边界，而不是将可选参数传递给函数
## 源码分析
### WithCancel
context.WithCancel生成了一个withCancel的实例以及一个cancelFuc，这个函数就是用来关闭ctxWithCancel中的 Done channel 函数。

下面来分析下源码实现，首先看看初始化，如下：
``` go
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{
		Context: parent,
		done:    make(chan struct{}),
	}
}

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
newCancelCtx返回一个初始化的cancelCtx，cancelCtx结构体继承了Context，实现了canceler方法：
``` go
//*cancelCtx 和 *timerCtx 都实现了canceler接口，实现该接口的类型都可以被直接canceled
type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}


type cancelCtx struct {
    Context
    done chan struct{} // closed by the first cancel call.
    mu       sync.Mutex
    children map[canceler]bool // set to nil by the first cancel call
    err      error             // 当其被cancel时将会把err设置为非nil
}

func (c *cancelCtx) Done() <-chan struct{} {
    return c.done
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}

//核心是关闭c.done
//同时会设置c.err = err, c.children = nil
//依次遍历c.children，每个child分别cancel
//如果设置了removeFromParent，则将c从其parent的children中删除
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
    close(c.done)
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c) // 从此处可以看到 cancelCtx的Context项是一个类似于parent的概念
    }
}
``` 

可以看到，所有的children都存在一个map中；Done方法会返回其中的done channel， 而另外的cancel方法会关闭Done channel并且逐层向下遍历，关闭children的channel，并且将当前canceler从parent中移除。

WithCancel初始化一个cancelCtx的同时，还执行了propagateCancel方法，最后返回一个cancel function。

propagateCancel 方法定义如下：
``` go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // parent is never canceled
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
propagateCancel 的含义就是传递cancel，从当前传入的parent开始（包括该parent），向上查找最近的一个可以被cancel的parent， 如果找到的parent已经被cancel，则将方才传入的child树给cancel掉，否则，将child节点直接连接为找到的parent的children中（Context字段不变，即向上的父亲指针不变，但是向下的孩子指针变直接了）； 如果没有找到最近的可以被cancel的parent，即其上都不可被cancel，则启动一个goroutine等待传入的parent终止，则cancel传入的child树，或者等待传入的child终结。

#### WithDeadLine
在withCancel的基础上进行的扩展，如果时间到了之后就进行cancel的操作，具体的操作流程基本上与withCancel一致，只不过控制cancel函数调用的时机是有一个timeout的channel所控制的。

## 小Tips
* 不要把Context放在结构体中，要以参数的方式传递，parent Context一般为Background
* 应该要把Context作为第一个参数传递给入口请求和出口请求链路上的每一个函数，放在第一位，变量名建议都统一，如ctx。
* 给一个函数方法传递Context的时候，不要传递nil，否则在tarce追踪的时候，就会断了连接
* Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
* Context是线程安全的，可以放心的在多个goroutine中传递
* 可以把一个 Context 对象传递给任意个数的 gorotuine，对它执行 取消 操作时，所有 goroutine 都会接收到取消信号。

## 参考资料
[掘金 -《Golang Context深入理解》](https://juejin.im/post/5a6873fef265da3e317e55b6)</br>
[《Go语言并发之道》 -- 中国电力出版社](https://github.com/kat-co/concurrency-in-go-src)
