# GO

## 一、基础语法

### 1. 变量的定义语法

Go语言中，变量的定义语法是和其他语言相反的，即先命名，后写变量类型，并且， __Go严格规定所有变量必须要被使用__ 。

```go
var a int
var b string
// 直接初始化
var a int = 1
var b string = "this is b var"
```

或者，也可以不给定类型，Go将自动判断其类型，又或者，连`var`都懒得写了，直接用`:=`。

注: Go中不允许在包中(其他语言看类似全局变量)使用`:=`来定义变量。

```go
var c = "this is string"
var d = true
// 使用 :=
c := "this is string"
d := true
```

Go语言中的内建数据类型关键字

```go
bool, string,
(u)int, (u)int8, (u)int16, (u)int32, (u)int64, uintptr
byte, rune
float32, float64, complex64, complex128
```

`rune`类似C/C++中的`char`，但其占用空间为`32bit`，这是为了兼容各种语言。

#### 定义常量

```go
const name string = "this is name"
const a int = 4
// 当然我们也可以省略变量类型
const name = "this is name"
const a = 4
```

枚举类型的定义

```go
const (
	cpp = 0 // 0
  _				// 1
  python	// 2
  java		// 3
)
// b kb mb gb tb pb
const (
  b = 1 << (10 * iota)
  kb
  mb
  gb
  tb
  pb
)
```

### 2. 条件语句语法

和不同语言不通，`if`语句的判断不需要括号

```go
x := 1
if x > 0 {
  fmt.Println("x > 0")
}else {
  fmt.Println("x < 0")
}
```

Go语言中的`switch`语句不需要进行`break`，其默认是`break`操作，如果需要类型其他语言中的操作，则需要`fallthrough`。

```go
x := 1
switch x {
case 1:
  fmt.Println(1)
case 2:
  fmt.Println(2)
default:
  panic("error")
}
// 或者
switch {
case x == 1:
  fmt.Println(1)
case x == 2:
  fmt.Println(2)
default:
  panic("error")  
}
```

### 3. 循环语法

Go语言中没有`while`关键字，全部使用`for`来实现，并且`for`语句的条件不需要括号。

```go
for i := 1; i<100; i++ {
  fmt.Println(i)
}
// while 直接使用for替代
for {
  fmt.Println("this is while")
}
```

### 4. 函数语法

Go的函数定义使用func关键字，同样是相反的，参数名在前，参数类型在后，返回类型在参数和`{}`之间。并且允许函数返回多个值，具体语法如下:

```go
func myfunc(a int, b int, c string) int {
  return 1
}
// 返回多个值，更多的是返回一个错误
func myfunc(a int, b int, c string) (int, int){
  return 1, 2
}
// 也可以，但比较少使用
func myfunc(a,b int, c string)( q, r int){
  q = 10
  r = 20
  return
}
```

不仅如此，和其他语言一样，Go也可以将函数作为参数传入。

```go
func echo(a int){
  fmt.Println(a)
}

func myfunc(insideEcho func(int), a int){
  insideEcho(a)
}
```

### 5. 指针

Go中也有指针，其声明方式也是同样为反即：`*int`，Go中指针不能使用偏移量相加，这里更多的使用来解释参数传递的方式。









## GC

垃圾回收用来处理一些不用的内存空间，栈这种结构基本不需要用到，栈上垃圾很容易回收，而堆上的垃圾则需要垃圾回收机制进行回收。

#### * 引用计数算法

Python使用的就是这种垃圾回收的方式，使用一个rc来记录一个对象的被引用次数，如果引用次数归零，则我们认为这块内存为垃圾，进行回收。

```C
typedef struct_object {
    int ob_refcnt; // 引用计数
    struct_typeobject *ob_type;
}PyObject;
```

这种方法是高效的，并且不需要STW，但有个致命的弱点，就是循环引用，当变量间相互引用的时候，就会导致rc一直不为0，内存将得不到回收，Python并没有解决这种问题，而是使用了另外的遍历方式，从一个Root内存块开始，跟随其引用找出`可达`的对象，然后将`不可达`的对象销毁掉。


#### * 可达性分析算法

这种算法一般从一个Root内存块开始(栈上对象，全局对象)，通过这个Root内存块像下寻找，找出可达对象(可以访问到的)，剩下的则为不可达对象，需要进行回收。

当然这种方法也有缺点，无法并发执行GC，需要STW，并且当对象较多的时候，STW的时间是比较长的。

#### * 采用三色法的可达性分析

