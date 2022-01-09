# map

- map是一种无序的键值对集合。Map最重要的一点是通过key来快速检索数据,key类似于索引，指向数据的值。
- map是使用哈希表实现的。
- map并发写入会抛出异常。
- map的删除操作通过将当前map中的key所指向的位置设为nil来完成。

### map定义

```go
var a map[string]interface{}		//a是值为nil的map[]
b := make(map[string]interface{})	//b是值为空的map[]
c := map[string]interface{}{}		//c是值为空的map[]
```

### map的使用

```go
func main() {
	a := make(map[string]interface{})
	a["hello"]="你好"
	a["sorry"]="对不起"
	for k :=range a {
		fmt.Println(a[k])
	}
	for k, v := range a {
		fmt.Println(k, v)
	}
	if k, ok := a["hello"]; ok {
		fmt.Println(k)
	}
	delete(a, "hello")
	fmt.Println(len(a))
	if k, ok := a["hello"]; ok {
		fmt.Println(k)
	}
}
//结果
你好
对不起
hello 你好
sorry 对不起
你好
1
```

### map的底层实现（1.17.5）

**map的源码位于[src/runtime/map.go](https://github.com/golang/go/blob/go1.17.5/src/runtime/map.go#L9)中。**

#### map底层的官方介绍

> 1、底层 map实际上是一个hashtable，存放的数据被放置到一个buckets数组中。每个bucket最多能够存放8个key/value对。而对应的数据则根据其hash(key)的bits低阶位来选择对应的bucket，而对应的高阶位则用来区分在同一个bucket中不同的key-value内容。一旦当前的bucket里面的键值对个数超过8个，则会通过链表的方式拓展其他的bucket。
>
> 2、扩容 当hashtable增长需要扩容时，则通过分配一个容量是当前数组buckets两倍的新数组来存放键值对，并将旧的buckets中的key-value逐步copy到新的buckets中。需要注意一点：当map进行扩容时其对应的iterator遍历旧的table，同时也会检查新的table，防止对应的bucket移到新的table中。
>
> 3、查询 可以通过Map iterator遍历buckets数组，按照遍历的顺序返回keys(当前bucket--->chain序号[当存放键值对>8个时会以链表的存放到扩展的bucket]--->bucket索引)，在实现map iterator为了保证其iterator语义，并不会在bucket中移动keys，一旦这样做了则会导致keys会被获取到0次或2次。
>
> 4、负载系数 在实际的应用我们会面临一个问题：当table出现太多overflow情况就需要扩展更多的bucket来支撑，反之，若是太小的话又会造成空间的浪费。

#### map的数据结构

- hmap

```go
type hmap struct {
	//注意：hmap的格式也编码在cmd/compile/internal/gc/reflect.go中.
	//确保这与编译器的定义保持同步.
	count     int 				// map存储的个数与len的大小相等.
	flags     uint8				// 读、写、扩容、迭代等标记，用于记录map当前状态
	B         uint8  			// 最多可容纳1<<B*6.5个元素，6.5为装填因子.
    noverflow uint16			// 溢出的个数(拓展的buckets).
	hash0     uint32 			// 哈希种子.

	buckets    unsafe.Pointer 	// 桶的地址.如果count==0,2^B个Buckets可能是nil.
	oldbuckets unsafe.Pointer 	// 旧桶的地址，用于扩容
	nevacuate  uintptr       	// 搬迁进度，小于nevacuate的已经搬迁

	extra *mapextra 			// 可选字段
}
```

- mapextra(溢出桶)

```go
/* 
当kev/value不为指针时，溢出桶存放到mapextra结构中，overflow存放buckets中的溢出桶，oldoverflow存放oldbuckets中的溢出桶，nextOverflow预分配溢出桶空间
*/
type mapextra struct {
	overflow    *[]*bmap	//以切片形式存放buckets中的每个溢出桶
	oldoverflow *[]*bmap	// 以切片形式存放oldbuckets中的每个溢出桶

	// nextOverflow拥有指向overflow bucket的指针.
	nextOverflow *bmap
}
```

- bmap

```go

type bmap struct {
	//每个元素hash值的高8位,如果tophash[0] < minTopHash,表示这个桶的搬迁状态.
	tophash [bucketCnt]uint8
    // 接下来是8个key、8个value,但是我们不能直接看到;为了优化对齐,go采用了key放在一起,value放在一起的存储方式,
    // 再接下来是hash冲突发生时,下一个溢出桶的地址.
}
```

- hmap当前状态的类型

```go
// flags
const (
	iterator     = 1 // 可能有一个迭代器正在使用buckets.
	oldIterator  = 2 // 可能有一个迭代器正在使用oldbuckets.
	hashWriting  = 4 // 一个goroutine正在写入到map
	sameSizeGrow = 8 // 当前map增长到相同大小的新map
)
```

#### 创建map

- 计算哈希表占用的内存是否溢出或者超出能分配的最大值；
- 如果map指针的地址不存在，则通过`new`来为map分配一个地址；
- 调用`fastrand`获取一个随机的哈希种子；
- 根据传入的hint计算出至少需要多少桶；
- 使用`makeBucketArray`创建用于保存桶的数组。

```go
// makemap 通过make(map[k]v, hint)来创建map.
// 如果编译器确定可以在堆上创建map或第一个bucket,则h和（或）bucket可能是非nil.
// 如果 h != nil, map能直接在h中被创建.
// 如果 h.buckets != nil, 指向的bucket可以作为第一个bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()	//产生哈希种子

	// 如果 hint < 0 overLoadFactor返回false, 因为 hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// 分配初始hash table
    // 如果 B == 0, 则稍后会延迟分配buckets字段(在mapassign中)
	// 如果 hint 很大，则将该内存清零可能需要一段时间.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}



```

- 常量

```go
const (
    //bucket可以容纳的最大key/value数量
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits
    // 一个触发增长的bucket最大平均负载为 6.5.
	// 表示为loadFactorNum / loadFactDen，以允许整数数学运算。
	loadFactorNum = 13
    loadFactorDen = 2
)
```

- overLoadFactor查看放置在 1<<B buckets中的count数是否超过 loadFactor.

```go
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

- bucketShift 返回 1<<b, 针对代码生成进行了优化.

```go
func bucketShift(b uint8) uintptr {
	//掩盖移位量来消除溢出检查.
	return uintptr(1) << (b & (sys.PtrSize*8 - 1))
}
```

- makeBucketArray用于初始化一个bucket 数组。

当桶的数量小于 16 时，由于数据较少、使用溢出桶的可能性较低，会省略创建的过程以减少额外开销；

当桶的数量多于 16 时，会额外创建 2^B−4 个溢出桶；

```go
//makeBucketArray 初始化map存储桶的后备数组。
//1<<b 是要分配的最小桶数。
//dirtyalloc 应该是 nil 或之前由 makeBucketArray 分配的具有相同 t 和 b 参数的存储桶数组。
//如果dirtyalloc 为 nil，则将分配一个新的后备数组，否则将清除dirtyalloc 并重新用作后备数组。
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// 对于小 b，溢出桶是不可能的。
	// 避免计算的开销。
	if b >= 4 {
		// 添加插入与此 b 值一起使用的元素的中位数所需的溢出桶的估计数量。
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		//dirtyalloc 之前由上面的 newarray(t.bucket, int(nbuckets)) 生成，但可能不为空。
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		//我们预先分配了一些溢出桶。
		//为了将跟踪这些溢出桶的开销保持在最低限度，我们使用约定，如果预先分配的溢出桶的溢出指针为零，则通过碰撞指针有更多可用的。
		//对于最后一个溢出桶，我们需要一个安全的非 nil 指针； 只需使用桶。
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

#### 插入

1.查看否是并发写入map，如果是并发写入会抛出异常。

2.计算key的hash。

3.设置写标记。

4.判断当前的buckets是否为nil，若为nil，创建bucket，bucket大小为1。

5.计算bucket的位置。

6.判断是否需要扩容，若需要则扩容。

7.计算tophash。

8.遍历buckets，查看key是否存在，若存在，跳转到12。

9.如果达到最大负载因数，或者我们有太多的溢出buckets，而我们还没有处于增长的中间，那就开始增长。跳转回5。

10.是否所有的bucket都是满的，无空闲的bocket，创建一个bucket。

11.插入key和value

12.map中key的数目加1(即h.count加1)

13.清空写标记。

14.插入value。

```go
//与mapaccess类似,但是如果map中不存在key则为该key分配一个位置.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil { //插入key的前提map不能为nil.
		panic(plainError("assignment to entry in nil map"))
	}
    //如果竞争检测启用
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
    //
	if msanenabled {
		msanread(key, t.key.size)
	}
    //检查是否是并发写入,并发写入会抛出异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
    //计算key的hash
	hash := t.hasher(key, uintptr(h.hash0))

	// 在调用t.hasher之后设置hashWriting,因为t.hasher可能会出现panic,在这种情况下,我们实际上并未执行写入操作.
    //设置写标记
	h.flags ^= hashWriting
	// 如果buckets地址为nil,为buckets分配地址空间
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    //计算bucket的位置以及tophash
	//nucketMask返回1<<b-1
	bucket := hash & bucketMask(h.B)
	if h.growing() { //是否扩容,如果扩容可能达到相同大小
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)
	//记录buckets中第一个空闲key/value的位置
	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
    //遍历buckets的每个bucket
	for {
        //遍历bucket内的元素
		for i := uintptr(0); i < bucketCnt; i++ {
            //tophash不相等，判断下一个元素
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
                    //记录buckets中第一个空闲key/value的位置
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
                //表示此bucket内接下来的元素都时不存在的
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
            //tophash相等
            //获取当前位置对应的key的值
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            //当前位置的key与要插入的key不相等则比较下一个
			if !t.key.equal(key, k) {
				continue
			}
			// key已存在. 更新key的值.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
        //计算下一个bucket的地址
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// key不存在. 分配空间并添加.

	// 如果达到最大负载因数，或者我们有太多的溢出buckets，而我们还没有处于增长的中间，那就开始增长。
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again //扩大表格会使所有内容失效，因此请重试
	}
	//所有的bucket都是满的  无空闲的bucket
	if inserti == nil {
		// 分配一个新的bucket.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// 插入key
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
    //插入value
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
    //map元素数目加一
	h.count++

done:
    //再次判断并发写
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
    //清空写标记
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

- **竞争检测是否启用**

```go
//竞争检测是否启用
const raceenabled = false
```

[src/runtime/stubs.go](https://github.com/golang/go/blob/2be7788f8383c2330cd96db53273e2995d4468f8/src/runtime/stubs.go#L237)

```go
// getcallerpc返回其调用者的调用者的程序计数器(PC).
// getcallersp返回其调用者的调用者的堆栈指针(SP).
// 该实现是编译器固有的.不必在每个平台上都实现此代码.
//
// 例如:
//
//	func f(arg1, arg2, arg3 int) {
//		pc := getcallerpc()
//		sp := getcallersp()
//	}
//
// 这两行在对f调用之后立即找到PC和SP(f将返回).
//
//
//getcallersp的结果在返回时是正确的，但是随后对函数的任何调用都可能使该结果无效，该函数可能会重新定位堆栈以增大或缩小堆栈。
//一般规则是，应立即使用getcallersp的结果，并且只能将其传递给nosplit函数。

//go:noescape 禁止逃逸
func getcallerpc() uintptr
// funcPC返回函数f的入口PC.
// 假定f是一个func值.否则行为是不正确的.
// CAREFUL: 在带有插件的程序中，funcPC可以为同一函数返回不同的值(因为在地址空间中实际上存在同一函数的多个副本).
// 			为了安全起见，请勿在任何==表达式中使用此函数的结果. 仅将结果用作开始执行代码的地址是安全的.
//go:nosplit 跳出堆栈溢出的检测
func funcPC(f interface{}) uintptr {
	return *(*uintptr)(efaceOf(&f).data)
}
const msanenabled = false
```

- bucketMask返回`1<<b-1`

```go
// bucketMask 返回 1<<b - 1.
func bucketMask(b uint8) uintptr {
	return bucketShift(b) - 1
}
```

- 判断h是否在增长

```go
// growing报告h是否在增长。 增长可能达到相同的大小或更大.
func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}
```

- 搬迁操作

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    //搬迁旧桶，这样assign和delete都直接在新桶集合中进行
	evacuate(t, h, bucket&h.oldbucketmask())

    // //再搬迁一次搬迁过程中的桶
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

- 搬迁的主函数

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```

#### 删除

1. map为nil或空直接返回。

2. 查看否是并发写入map，如果是并发写入会抛出异常。

3. 获取key的hash。

4. 设置写标记。

5. 计算bucket的首地址。

6. 判断是否需要扩容，若需要则扩容。

7. 计算tophash。

8. 遍历桶找到key,将该key值标记为emptyone。

9. map总数减一。

10. 查看否是并发写入map，如果是并发写入会抛出异常。

11. 清除写标记。

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    //竞争检测是否启用
    //h是否为空
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
    //h为空或者count为0直接返回
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return
	}
    //查看否是并发写入map，如果是并发写入会抛出异常。
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	//获取key的hash
	hash := t.hasher(key, uintptr(h.hash0))

    //设置写标记
	h.flags ^= hashWriting
	//计算bucket的首地址
	bucket := hash & bucketMask(h.B)
    //判断是否扩容
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
    //计算tophash
	top := tophash(hash)
search:
    //通过overflow遍历bucket
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
            //tophash不相等
			if b.tophash[i] != top {
                //如果tophash==emptyTest,表示该bucket为空，并且bucket中不再有较高的索引或溢出的非空的单元.说明key不存在,跳出循环.
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
            //tophash相等
            //获取当前位置的key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
            //判断是否为目标key
			if !t.key.equal(key, k2) {
				continue
			}
            // 处理指针.
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
            // 计算出value位置，处理指针
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
            //将该tophash标记为emptyOne
			b.tophash[i] = emptyOne
			//删除key后bucket的处理.
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
            //map数目减一
			h.count--
			break search
		}
	}
	//判断是否并发写
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
    //清除写标记
	h.flags &^= hashWriting
}
```

### 参考文献

[Golang map源码详解](https://blog.csdn.net/fengshenyun/article/details/100582679)

[golang源码之map](https://www.jianshu.com/p/d2eaf9133ca0)

[golang map源码分析](https://blog.csdn.net/weixin_34185364/article/details/92072558)

[golang源码之map](https://studygolang.com/articles/19069?fr=sidebar)

[golang map源码详解](https://blog.yiz96.com/golang-map/)

[Golang map 的底层实现](https://www.jianshu.com/p/aa0d4808cbb8)

[Go是如何设计Map的](https://mp.weixin.qq.com/s/hmOBmHFiybS0JqoCpQHmSw)