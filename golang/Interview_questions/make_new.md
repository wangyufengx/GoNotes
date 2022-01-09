# make与new区别和联系

# 内置函数make

- 内置函数make只为slice、map、chan类型分配内存。
- 与new一样，make的第一个参数是类型，不是值。
- 与new不同，make的返回类型与其参数的类型相同，而不是指向它们的指针。结果取决于类型。
- slice:大小指定长度。切片的容量等于其长度。可以提供第二个整数参数来指定不同的容量。它必须不小于长度。例如，make（[] int，0，10）分配一个大小为10的基础数组，并返回一个长度为0且容量为10的切片，该切片由该基础数组支持。
- map:为空的map分配足够的空间来容纳指定数量的元素。该大小可以省略，在这种情况下，分配的起始大小较小。
- Channel:使用指定的缓冲区容量初始化通道的缓冲区。如果为零或忽略大小，则通道不缓冲。

### 函数定义

```go
func make(t Type, size ...IntegerType) Type
```

### 实例

```go
func main() {
	a := make([]int, 0)
	b := make(map[string]interface{})
	c := make(chan string)
	fmt.Println(a, b, c)
}
```

# 内置函数new

- 内置函数new用于分配内存。
- 传入的参数是类型而不是值，返回的值是指向该类型新分配内存的零值指针。

### 函数定义

```go
func new(Type) *Type
```

### 实例

```go
func main() {
	a := new(int)						//0xc0000160f0
	b := new(string)					//0xc00003c1f0
	c := new(bool)						//0xc0000160f8
	d := new(map[string]interface{})	//&map[]
	e := new(interface{})				//0xc00003c200
	f := new(struct{})					//&{}
	g := new([]int)						//&[]
	h := new(chan string)				//0xc000006030
	fmt.Println(a, b, c, d, e, f, g, h)
}
type Point struct {
	X, Y float64
}

func (p *Point) Abs() float64 {
	return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func main() {
	var p *Point
	fmt.Println(p.Abs())
}
```

上面的程序会`panic`

```go
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x0 addr=0x0 pc=0x49deca]

goroutine 1 [running]:
main.(*Point).Abs(...)
        E:/Project/Algo/main.go:13
main.main()
        E:/Project/Algo/main.go:18 +0x2a
```

为什么会出现`panic`呢？

从`invalid memory address or nil pointer dereference`可以看出我们的p的值是nil。

```go
// 这种声明方式 p 是一个 nil 值
var p *Point

// 改为
var p *Point = new(Point)

// 或者
var p *Point = &Point{}
```

### 参考资料

https://www.kancloud.cn/aceld/golang/1958307