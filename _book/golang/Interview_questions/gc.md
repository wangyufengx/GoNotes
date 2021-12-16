# golang 垃圾回收

### GC

在计算机科学中，垃圾回收(GC)是一种自动管理内存的机制，垃圾回收器会去尝试回收程序不再使用的对象及其占用的内存。

### GC触发的场景

- 系统触发：运行时自行根据内置的条件进行 GC 处理，维护整个应用程序的可用性。
- 手动触发：开发者在业务代码中自行调用`runtime.GC()`方法来触发 GC 行为。

### 什么时候触发GC

在系统触发的场景中，Go 源码的 src/runtime/mgc.go 文件，明确标识了 GC 系统触发的三种场景，分别如下：

```golang
const (
	gcTriggerHeap gcTriggerKind = iota
	gcTriggerTime
	gcTriggerCycle
)
```

- gcTriggerHeap：表示当堆大小达到控制器计算出的触发堆大小时开始gc。小对象：如果申请小对象时，发现当前内存空间不存在空闲跨度时，将会需要调用 nextFree 方法获取新的可用的对象，可能会触发 GC 行为。大对象：如果申请大于 32k 以上的大对象时，可能会触发 GC 行为。
- gcTriggerTime：表示当自上一个 GC 周期以来已经超过 forcegcperiod 纳秒时，应该开始gc。时间周期以 runtime.forcegcperiod 变量为准，默认 2 分钟。
-  gcTriggerCycle 如果当前没有开启垃圾收集，则启动GC。主要是 runtime.GC()。

| GC调用方式     | 所在位置                        | 代码                                                     |
| -------------- | ------------------------------- | -------------------------------------------------------- |
| 分配内存时调用 | runtime/malloc.go:mallocgc()    | gcTrigger{kind: gcTriggerHeap}                           |
| 定时调用       | runtime/proc.go:forcegchelper() | gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()}) |
| 手动调用       | runtime/mgc.go:GC()             | gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})       |

#### 是否需要触发GC源码

```golang
//是否满足触发条件
func (t gcTrigger) test() bool {
	if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
		return false
	}
	switch t.kind {
	case gcTriggerHeap:
		// Non-atomic access to gcController.heapLive for performance. If
		// we are going to trigger on this, this thread just
		// atomically wrote gcController.heapLive anyway and we'll see our
		// own write.
		return gcController.heapLive >= gcController.trigger
	case gcTriggerTime:
		if gcController.gcPercent < 0 {
			return false
		}
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
	case gcTriggerCycle:
		// t.n > work.cycles, but accounting for wraparound.
		return int32(t.n-work.cycles) > 0
	}
	return true
}
```

### 垃圾回收机制

[Golang三色标记+混合写屏障GC模式全分析](https://www.kancloud.cn/aceld/golang/1958308)

### 参考文档

[go:垃圾回收GC触发条件](https://blog.csdn.net/zoeou/article/details/104089630/)

[Go 什么时候会触发 GC？](http://www.zzvips.com/article/195472.html)

[Golang三色标记+混合写屏障GC模式全分析](https://www.kancloud.cn/aceld/golang/1958308)