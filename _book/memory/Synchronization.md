# 同步

### 初始化

程序的初始化在单个 goroutine 中运行，但该 goroutine 可能会创建其他并发运行的 goroutine。

如果包`p` 导入包`q`，则`q` 的`init` 函数的完成发生在任何`p` 的开始之前。

`main.main` 函数的启动发生在所有 `init` 函数完成之后。

### 创建Goroutine 

 `go` 关键字开启新的 goroutine ，发生在这个 goroutine 开始执行之前。

例如，在这个程序中：

```go
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用 `hello` 将在未来的某个时间点打印 `"hello, world"`（可能在 `hello` 返回之后）。

### 销毁Goroutine

不能保证 goroutine 的退出发生在程序中的任何事件之前。 例如，在这个程序中：

```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

对 `a` 的赋值之后没有任何同步事件，因此不能保证任何其他 goroutine 都会观察到它。 事实上，激进的编译器可能会删除整个 `go` 语句。

如果一个 goroutine 的效果必须被另一个 goroutine 观察到，请使用同步机制，例如锁或通道通信来建立相对顺序。

### Channel通信

Channel 通信是 goroutine 之间同步的主要方式。 特定channel 上的每个发送都与来自该Channel 的相应接收匹配，通常在不同的 goroutine 中。

Channel 上的发送发生在来自该channel 的相应接收完成之前。

这个程序：

```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

保证打印“你好，世界”。 写入 `a` 发生在 `c` 上的发送之前，发生在 `c` 上的相应接收完成之前，发生在 `print` 之前。

Channel 的关闭发生在由于channel 关闭而接收到零值之前。

在前面的例子中，用 `close(c)` 替换 `c <- 0` 会产生一个具有相同保证行为的程序。

来自无缓冲channel 的接收发生在该channel 上的发送完成之前。

该程序（如上，但交换了发送和接收语句并使用了无缓冲channel ）：

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

还保证打印“你好，世界”。写入 `a` 发生在接收 `c` 之前，发生在 `c` 上的相应发送完成之前，发生在 `print` 之前。

如果channel 被缓冲（例如，`c = make(chan int, 1)`），那么程序将不能保证打印“hello, world”。 （它可能会打印空字符串、崩溃或执行其他操作。）

容量为 *C* 的channel 上的 *k*th 接收发生在来自该channel 的 *k*+*C*th 发送完成之前。

此规则将先前的规则推广到缓冲channel 。它允许通过缓冲channel 对计数信号量进行建模：channel 中的项目数对应于活动使用的数量，channel 的容量对应于同时使用的最大数量，发送项目获取信号量，以及接收一个项目释放信号量。这是限制并发的常用习惯用法。

该程序为工作列表中的每个条目启动一个 goroutine，但 goroutine 使用 `limit` channel 进行协调，以确保一次最多运行三个工作函数。

```
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### 锁

`sync` 包实现了两种锁数据类型，`sync.Mutex` 和 `sync.RWMutex`。

对于任何`sync.Mutex` 或`sync.RWMutex` 变量`l` 和*n* < *m*，`l.Unlock()` 的调用 *n* 发生在 `l.Lock()` 的调用 *m* 返回之前。

这个程序：

```
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

保证打印“你好，世界”。 第一次调用 `l.Unlock()`（在 `f` 中）发生在第二次调用 `l.Lock()`（在 `main` 中）返回之前，发生在 `print` 之前。

对于在 `sync.RWMutex` 变量 `l` 上对 `l.RLock` 的任何调用，都有一个 *n* 使得 `l.RLock` 在调用 *n* 到 `l.Unlock` 之后发生（返回） 并且匹配的 `l.RUnlock` 发生在调用 *n*+1 到 `l.Lock` 之前。

### 一次

`sync` 包通过使用 `Once` 类型为存在多个 goroutine 的情况下的初始化提供了一种安全机制。 多个线程可以为特定的 `f` 执行 `once.Do(f)`，但只有一个会运行 `f()`，而其他线程会阻塞直到 `f()` 返回。

来自`once.Do(f)` 的单个`f()` 调用在任何`once.Do(f)` 调用返回之前发生（返回）。

在这个程序中：

```
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

调用 `twoprint` 只会调用一次 `setup`。 `setup` 函数将在调用 `print` 之前完成。 结果将是 `"hello, world"` 将被打印两次。
