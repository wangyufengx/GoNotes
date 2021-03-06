# golang调度

### 第一代调度模型的问题

在 Go 语言1.0版本时，只有 G-M 模型，Google 工程师 Dmitry Vyukov 在[**《Scalable Go Scheduler Design Doc》**](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)中指出了该模型在并发伸缩性方面的问题：

第一代的 Go 调度程序只有一个全局运行队列，并有一个互斥锁保护它。线程经常被阻塞，等待互斥锁解锁。它限制了用 Go 并发程序的伸缩性，特别是高吞吐量服务器和并行计算程序。

>1. 所有对 G 的操作：创建、完成、重新调用等由单个全局互斥锁(Sched.Lock)保护，浪费时间。
>
>2. 当 M 阻塞时，G 需要传递给别的 M 执行，G的切换调度会导致延迟增加以及额外的性能损耗；
>
>3. 内存缓存和其他缓存（堆栈分配的）跟所有的 M 有关。不仅和正在运行 Go 代码的 M 关联，而且还和阻塞在系统调用中的 M 关联。导致了过多无用的资源消耗以及非常不好的作用域局部性。
>
>4. 由于 syscall 导致 M 的阻塞和恢复，导致了额外的开销。

### 线程模型

操作系统的线程分为用户级线程和内核级线程

- 一对一模型：一个执行线程与一个内核线程匹配。它利用了机器上的所有内核，但是上下文切换很慢，因为它必须通过操作系统进行捕获。
- 多对一模型：多个用户线程在一个内核线程上运行。这样做的优点是可以非常快速地进行上下文切换，但不能利用多核系统。
- 多对多模型：将多个用户线程在不止一个内核线程上运行。Go 试图通过使用 M:N 调度程序来实现两全其美。它将任意数量的 goroutine 调度到任意数量的 OS 线程上。您可以获得快速的上下文切换，并利用系统中的所有核心。这种方法的主要缺点是它增加了调度程序的复杂性。

### GMP调度模型

- G：表示 Goroutine，G 存储了 goroutine 执行的栈信息，状态，任务函数，可重用。

- M（Machine）代表一个操作系统线程。它是真正执行计算的部分。M 在绑定 P 后会在队列中获取 G，切换到 G 的执行栈并执行 G 的函数。M 数量不定，但同时只有 P 个 M 在执行，为了防止创建过多系统线程导致系统调度出现问题，目前默认最大限制10000个。
- P：Processor，表示逻辑处理器，拥有一个本地队列。对于 G 来说，P 相当于 CPU 内核，只有进入到 P 的队列中，才可以被调度。对于 M 来说，P 提供了相关的执行环境（Context），如内存分配状态，任务队列等。P 的数量就是程序可最大可并行的 G 的数量（前提：物理CPU核数 >= P的数量），由用户设置的 GOMAXPROCS 决定。

#### 调度过程

​		当通过 go 关键字创建一个新的 goroutine 的时候，它会**优先**被放入 P 的本地队列。为了运行 goroutine，M 需要持有（绑定）一个 P，接着 M 会启动一个 OS 线程，循环从 P 的本地队列里取出一个 goroutine 并执行。执行调度算法：当 M 执行完了当前 P 的 Local 队列里的所有 G 后，P 也不会就这么在那划水啥都不干，它会先尝试从 Global 队列寻找 G 来执行，如果 Global 队列为空，它会随机挑选另外一个 P，从它的队列里中拿走一半的 G 到自己的队列中执行。

#### 特殊的m0和g0

- m0是启动程序后的主线程，这个m对应的实例会在全局变量m0中，M0负责执行初始化操作和启动第一个g， 在之后M0就和其他的m一样了。

- 每个m都会有一个自己的管理堆栈g0，g0不指向任何可执行的函数,，g0仅在m执行管理和调度逻辑时使用。在调度或系统调用时会使用g0的栈空间, 全局变量的g0是m0的g0。

#### Go启动初始化过程

1. 分配和查找栈空间。
2. 初始化参数和环境变量。
3. 当前运行线程标记为m0，m0是程序启动的主线程。
4. 调用运行时初始化函数runtime.schedinit进行初始化。
5. 在m0上调度第一个G，这个G运行runtime.main函数。（runtime.main函数会拉起运行时的监控线程，然后调用main包的初始化函数，最后执行main函数。）

#### 阻塞

- 阻塞 I/O
- select操作
- 阻塞在channel
- 等待锁
- 主动调用 runtime.Gosched()

#### 抢占调度

##### 原因

1. 不让某个G长久地被系统调用阻塞，阻碍其他G运行。
2. 不让某个G一直占用某个M不释放。
3. 避免全局队列里面的G得不到执行。

##### 策略

1. 在进入系统调用（syscall）前后，各封装一层代码检测G的状态，当检测到当前G已经被监控线程抢占调度，则M停止执行当前G,进行调度切换。
2. 监控线程经过一段时间检测感知到P运行超过一定时间，取消P和M的关联，这也是一种更高层次的调度。
3. 监控线程经过一段时间检测感知到G一直运行，超过了一定的时间，设置G标记。G执行栈扩展逻辑检测到抢占标记，根据相关条件决定是否抢占调度。

#### 监控线程（sysmon）

sysmon每20us~10ms启动一次，sysmon主要完成如下工作：

- 释放闲置超过5分钟的span物理内存；
- 如果超过2分钟没有垃圾回收，强制执行；
- 将长时间未处理的netpoll结果添加到任务队列；
- 向长时间运行的G任务发出抢占调度；
- 收回因syscall长时间阻塞的P；

### CSP并发模型

CSP模型是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。 CSP中channel是第一类对象，它不关注发送消息的实体，而关注与发送消息时使用的channel。

#### golang CSP

Golang 就是借用CSP模型的一些概念为之实现并发进行理论支持，其实从实际上出发，go语言并没有，完全实现了CSP模型的所有理论，仅仅是借用了 process和channel这两个概念。process是在go语言上的表现就是 goroutine 是实际并发执行的实体，每个实体之间是通过channel通讯来实现数据共享。

### 参考资料

[The Go scheduler](https://morsmachine.dk/go-scheduler)

[golang scheduler](https://yizhi.ren/2019/06/03/goscheduler/)

[Go 的并发性与调度器](https://www.jianshu.com/p/56c0c51ab537)

[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

[Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#)

[Golang调度器GMP原理与调度全分析](https://studygolang.com/articles/26921)

[详解Go语言调度循环源码实现](https://www.cnblogs.com/luozhiyun/p/14426737.html)

[Go 调度模型 GPM](https://studygolang.com/articles/29179)

[Golang 调度器和 GMP 模型](https://segmentfault.com/a/1190000040080290)

[Golang CSP并发模型](https://studygolang.com/articles/5155)