

# golang有哪些陷阱

*以下来自网上整理*

### Golang中range指针数据的坑【转】

在Golang中使用`for range`语句进行迭代非常的便捷，但在涉及到指针时就得小心一点了。

下面的代码中定义了一个元素类型为`*int`的通道`ch`：

```golang
package main

import (
    "fmt"
)

func main() {
    ch := make(chan *int, 5)
    
    //sender
    input := []int{1,2,3,4,5}
    
    go func(){
        for _, v := range input {
            ch <- &v
        }
        close(ch)
    }()
    //receiver
    for v := range ch {
        fmt.Println(*v)
    }
}
```

在上面代码中，发送方将`input`数组发送给`ch`通道，接收方再从`ch`通道中接收数据，程序的预期输出应该是：

```
1
2
3
4
5
```

现在运行一下程序，得到的输出如下：

```
5
5
5
5
5
```

很明显，程序并没有达到预期的结果，那么问题出在哪里呢？我们将代码稍作修改：

```golang
//receiver
    for v := range ch {
        fmt.Println(v)
    }
```

得到如下输出：

```
0x416020
0x416020
0x416020
0x416020
0x416020
```

可以看到，5次输出变量`v`（`*int`）都指向了同一个地址，返回去检查一下发送部分代码：

```
for _, v := range input {
    ch <- &v
}
```

问题正是出在这里，在`for range`语句中，`v`变量用于保存迭代`input`数组所得的值，但是`v`只被声明了一次，此后都是将迭代`input`出的值赋值给`v`，`v`变量的内存地址始终未变，这样再将`v`的地址发送给`ch`通道，发送的都是同一个地址，当然无法达到预期效果。

解决方案是，引入一个中间变量，每次迭代都重新声明一个变量`temp`，赋值后再将其地址发送给`ch`：

```
for _, v := range input {
    temp := v
    ch <- &temp
}
```

抑或直接引用数据的内存（推荐，无需开辟新的内存空间）：

```
for k, _ := range input {
    c <- &input[k]
}
```

再次运行，就可看到预期的效果。以上方案是用于讨论`range`语句带来的问题，当然，平时还是尽量避免使用指针类型的通道。

### 数组和切片

下面程序的输出是

```go
func foo(a [2]int) {
	a[0] = 200
}

func main() {
	a := [2]int{1, 2}
	foo(a)
	fmt.Println(a)
}
```

正确的输出是 `[1 2]`，数组 `a` 没有发生改变。

- 在 Go 语言中，数组是一种值类型，而且不同长度的数组属于不同的类型。例如 `[2]int` 和 `[20]int` 属于不同的类型。
- 当值类型作为参数传递时，参数是该值的一个拷贝，因此更改拷贝的值并不会影响原值。

下面程序的输出是

```go
func foo(a []int) {
	a = append(a, 1, 2, 3, 4, 5, 6, 7, 8)
	a[0] = 200
}

func main() {
	a := []int{1, 2}
	foo(a)
	fmt.Println(a)
}
```

输出仍是 `[1 2]`，切片 `a` 没有发生改变。

传参时拷贝了新的切片，因此当新切片的长度发生改变时，原切片并不会发生改变。而且在函数 `foo` 中，新切片 `a` 增加了 8 个元素，原切片对应的底层数组不够放置这 8 个元素，因此申请了新的空间来放置扩充后的底层数组。这个时候新切片和原切片指向的底层数组就不是同一个了。因此，对新切片第 0 个元素的修改，并不会影响原切片的第 0 个元素。

如果如果希望 `foo` 函数的操作能够影响原切片呢？

两种方式：

- 设置返回值，将新切片返回并赋值给 `main` 函数中的变量 `a`。
- 切片也使用指针方式传参。

```
func foo(a []int) []int {
	a = append(a, 1, 2, 3, 4, 5, 6, 7, 8)
	a[0] = 200
	return a
}

func main() {
	a := []int{1, 2}
	a = foo(a)
	fmt.Println(a)
}
```

或

```
func foo(a *[]int) {
	*a = append(*a, 1, 2, 3, 4, 5, 6, 7, 8)
	(*a)[0] = 200
}

func main() {
	a := []int{1, 2}
	foo(&a)
	fmt.Println(a)
}
```

上述两个程序的输出均为：

```
[200 2 1 2 3 4 5 6 7 8]
```

从可读性上来说，更推荐第一种方式。

### map遍历是顺序不固定

map是一种hash表实现, 每次遍历的顺序都可能不一样.

```go
func main() {
    m := map[string]string{
        "1": "1",
        "2": "2",
        "3": "3",
    }

    for k, v := range m {
        println(k, v)
    }
}
```

## 可变参数是空接口类型

当参数的可变参数是空接口类型时，传入空接口的切片时需要注意参数展开的问题。

```go
func main() {
    var a = []interface{}{1, 2, 3}

    fmt.Println(a)
    fmt.Println(a...)
}
```

不管是否展开，编译器都无法发现错误，但是输出是不同的：

```
[1 2 3]
1 2 3
```

## recover必须在defer函数中运行

recover捕获的是祖父级调用时的异常，直接调用时无效：

```go
func main() {
    recover()
    panic(1)
}
```

直接defer调用也是无效：

```go
func main() {
    defer recover()
    panic(1)
}
```

defer调用时多层嵌套依然无效：

```go
func main() {
    defer func() {
        func() { recover() }()
    }()
    panic(1)
}
```

必须在defer函数中直接调用才有效：

```go
func main() {
    defer func() {
        recover()
    }()
    panic(1)
}
```

## main函数提前退出

后台Goroutine无法保证完成任务。

```go
func main() {
    go println("hello")
}
```

## 通过Sleep来回避并发中的问题

休眠并不能保证输出完整的字符串：

```go
func main() {
    go println("hello")
    time.Sleep(time.Second)
}
```

类似的还有通过插入调度语句：

```go
func main() {
    go println("hello")
    runtime.Gosched()
}
```

Goroutine是协作式抢占调度，Goroutine本身不会主动放弃CPU：

```go
func main() {
    runtime.GOMAXPROCS(1)

    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(i)
        }
    }()

    for {} // 占用CPU
}
```

解决的方法是在for循环加入runtime.Gosched()调度函数：

```go
func main() {
    runtime.GOMAXPROCS(1)

    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(i)
        }
    }()

    for {
        runtime.Gosched()
    }
}
```

或者是通过阻塞的方式避免CPU占用：

```go
func main() {
    runtime.GOMAXPROCS(1)

    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(i)
        }
        os.Exit(0)
    }()

    select{}
}
```

## 不同Goroutine之间不满足顺序一致性内存模型

因为在不同的Goroutine，main函数中无法保证能打印出`hello, world`:

```go
var msg string
var done bool

func setup() {
    msg = "hello, world"
    done = true
}

func main() {
    go setup()
    for !done {
    }
    println(msg)
}
```

解决的办法是用显式同步：

```go
var msg string
var done = make(chan bool)

func setup() {
    msg = "hello, world"
    done <- true
}

func main() {
    go setup()
    <-done
    println(msg)
}
```

msg的写入是在channel发送之前，所以能保证打印`hello, world`

## 闭包错误引用同一个变量

```go
func main() {
    for i := 0; i < 5; i++ {
        defer func() {
            println(i)
        }()
    }
}
```

改进的方法是在每轮迭代中生成一个局部变量：

```go
func main() {
    for i := 0; i < 5; i++ {
        i := i
        defer func() {
            println(i)
        }()
    }
}
```

或者是通过函数参数传入：

```go
func main() {
    for i := 0; i < 5; i++ {
        defer func(i int) {
            println(i)
        }(i)
    }
}
```

## 在循环内部执行defer语句

defer在函数退出时才能执行，在for执行defer会导致资源延迟释放：

```go
func main() {
    for i := 0; i < 5; i++ {
        f, err := os.Open("/path/to/file")
        if err != nil {
            log.Fatal(err)
        }
        defer f.Close()
    }
}
```

解决的方法可以在for中构造一个局部函数，在局部函数内部执行defer：

```go
func main() {
    for i := 0; i < 5; i++ {
        func() {
            f, err := os.Open("/path/to/file")
            if err != nil {
                log.Fatal(err)
            }
            defer f.Close()
        }()
    }
}
```

## 切片会导致整个底层数组被锁定

切片会导致整个底层数组被锁定，底层数组无法释放内存。如果底层数组较大会对内存产生很大的压力。

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = data[:1]
    }

    // do some thing
}
```

解决的方法是将结果克隆一份，这样可以释放底层的数组：

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = append([]byte{}, data[:1]...)
    }

    // do some thing
}
```

## 空指针和空接口不等价

比如返回了一个错误指针，但是并不是空的error接口：

```go
func returnsError() error {
    var p *MyError = nil
    if bad() {
        p = ErrBad
    }
    return p // Will always return a non-nil error.
}
```

## 内存地址会变化

Go语言中对象的地址可能发生变化，因此指针不能从其它非指针类型的值生成：

```go
func main() {
    var x int = 42
    var p uintptr = uintptr(unsafe.Pointer(&x))

    runtime.GC()
    var px *int = (*int)(unsafe.Pointer(p))
    println(*px)
}
```

当内存发送变化的时候，相关的指针会同步更新，但是非指针类型的uintptr不会做同步更新。

同理CGO中也不能保存Go对象地址。

## Goroutine泄露

Go语言是带内存自动回收的特性，因此内存一般不会泄漏。但是Goroutine确存在泄漏的情况，同时泄漏的Goroutine引用的内存同样无法被回收。

```go
func main() {
    ch := func() <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                ch <- i
            }
        } ()
        return ch
    }()

    for v := range ch {
        fmt.Println(v)
        if v == 5 {
            break
        }
    }
}
```

上面的程序中后台Goroutine向管道输入自然数序列，main函数中输出序列。但是当break跳出for循环的时候，后台Goroutine就处于无法被回收的状态了。

我们可以通过context包来避免这个问题：

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    ch := func(ctx context.Context) <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                select {
                case <- ctx.Done():
                    return
                case ch <- i:
                }
            }
        } ()
        return ch
    }(ctx)

    for v := range ch {
        fmt.Println(v)
        if v == 5 {
            cancel()
            break
        }
    }
}
```

当main函数在break跳出循环时，通过调用`cancel()`来通知后台Goroutine退出，这样就避免了Goroutine的泄漏。



### 原文地址

https://studygolang.com/articles/18087

[切片(slice)性能及陷阱](https://geektutu.com/post/hpg-slice.html)

[附录A：Go语言常见坑](https://chai2010.cn/advanced-go-programming-book/appendix/appendix-a-trap.html)