# sync.pool

`sync.Pool` 是一个临时对象池。

```go
var p sync.Pool

func main() {
	p.Put("aaa")
	fmt.Println(p.Get())	// aaa

	time.Sleep(300 *time.Second)
	fmt.Println(p.Get())	// <nil>
}
```



https://golang.design/under-the-hood/zh-cn/part1basic/ch05sync/pool/

### 参考资料

[标准库](https://pkg.go.dev/sync#Pool)

[缓存池](https://golang.design/under-the-hood/zh-cn/part1basic/ch05sync/pool/)
