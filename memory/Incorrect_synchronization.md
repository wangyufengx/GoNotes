# 错误的同步

请注意，读操作 r 可能会观察到并发的写操作 w 。 即使这样，也不意味着在 r 之后发生的读取会观察到 w 之前的写入。

在这个程序中：

```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

`g`可能会先打印 2 然后打印 0。

这一事实使一些旧的习惯是错误的。

双重检查锁定是一种避免同步开销的尝试。 例如， twoprint 程序可能被错误地写为：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

但不能保证，在 doprint 中，观察对 done 的写入意味着观察对 a 的写入。 此程序可能会错误地输出一个空字符串而不是“hello, world”。

另一个错误的习惯用法是忙于等待一个值，例如：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

和以前一样，不能保证在 main 中，观察对 done 的写入意味着观察对 a 的写入，因此该程序也可以打印一个空字符串。 更糟糕的是，由于两个线程之间没有同步事件，因此无法保证 main 会观察到对 done 的写入。 main 中的循环不能保证完成。

这个主题有更微妙的变体，比如这个程序。

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

即使 main 观察到 g != nil 并退出它的循环，也不能保证它会观察到 g.msg 的初始化值。

在所有这些示例中，解决方案都是相同的：使用显式同步。
