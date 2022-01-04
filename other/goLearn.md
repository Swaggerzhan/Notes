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
  _       // 1
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
Go中还会for还会和select关键字进行配合，来接收管道中的信息，比如：
```go
func main() {
  timer := time.NewTicker(100 * time.Millisecond) 
  for {
    select{
    case <- timer.C:
      fmt.Println("timeout!")
    }
  }
}
```
如果没有使用for来包裹select，则select成功一次后将直接退出。

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

### 6. 容器

#### 数组：

Go中的数组定义同样是相反，即现`[]`其次才是类型，例如:

```go
var arr [3]int
// 当然，也可以直接使用 := 来进行定义，只不过这种方式需要数组具有初始值
arr := [3]int {1, 2, 3}
// 你也可以直接省略其[]中的数字，由编译器来进行判别
arr := []int {1, 2, 3}
```

如果我们需要定义多维数组，那么就再增加`[]`即可。

```go
// 多维数组
var arr [3][4]int
fmt.Println(arr)
// [[0 0 0 0] [0 0 0 0] [0 0 0 0]]
```

遍历数组最简单的就是直接使用一个index来进行遍历，如:

```go
arr := [3]int {3, 5, 7}
for i := 0; i < len(arr); i++ {
  fmt.Println(arr[i])
}
// 或者我们也可以用range来进行遍历
for i := range arr {
  fmt.Println(arr[i])
}
// 如果不使用索引来取值，我们也可以直接从range中获取到值
for index, value := range arr {
  fmt.Println(index, value)
}
```

注：数组是值类型，也就是说，当进行函数参数传递时，属于深拷贝。

```go
func printArray(arr [5]int){
  for i:=0; i<len(arr); i++ {
    arr[i] = i
  }
  fmt.Println(arr) // 0, 1, 2, 3, 4
}
func main(){
  arr := [5]int{10, 20, 30, 40, 50}
  printArray(arr)
  fmt.Println(arr) // 10, 20, 30, 40, 50
  // printArray中对于值的修改属于副本，这点不同于C/C++
}
```

并且，Go中数组长度是严格的不同类型

```go
arr := [4]int{0, 1, 2, 3}
arr := [5]int{0, 1, 2, 3, 4} // 两者数组长度不同，属于完全不同的类型！
```

以上使用属实有些麻烦，不过有了切片的引入，就使得程序编写简单了很多。

#### 切片(Slice)：

Slice是一个`结构`，其本质是对于一个数组的`view`，也就是类似引用的东西，其中保存有数组的起始位置(内存地址)，长度等等各种信息，对Slice修改会直接影响真正的地址空间上的数据。

```go
func updateSlice(s []int){ // 函数接受一个Slice
	for i := 0; i < len(s); i++ {
		s[i] = i
	}
	fmt.Println(s)
}
func main() {
  arr := [5]int {10, 20, 30, 40, 50}
  // 函数使用了其"view"，修改了真正的地址空间上的内容
  updateSlice(arr[:]) // 0, 1, 2, 3, 4
	fmt.Println(arr) // 0, 1, 2, 3, 4
}
```

想要理解Slice就要从其源码下手了，在`runtime/slice.go`中存在关于Slice的定义

```go
type slice struct {
	array unsafe.Pointer
  len int
  cap int
}
```

学过C/C++的看到这个源码应该立马就恍然大悟了，array即指向真正的地址空间，len代表当前Slice可用长度，而cap也就是capacity代表array真正的容量。

例如:

```go
arr := []int{0, 1, 2, 3, 4, 5}
s1 := arr[0:2] 
s2 := s1[1:4] 
fmt.Println(s1) // 0, 1
fmt.Println(s2) // 1, 2, 3
```

Slice总是向后扩展的，在超过len的长度，不超过cap的前提下，是完全允许的操作，只有当超过cap才会出现报错。 

除了这点，还需要看一个append函数，例子如下:

```go
arr := [5]int {0, 1, 2, 3, 4}
s1 := arr[2:4] // 2, 3
s2 := append(s1, 40) // 2, 3, 40
s3 := append(s2, 50)	// 2, 3, 40, 50 不在引用arr
fmt.Println(s1) // 2, 3
fmt.Println(s2) // 2, 3, 40 此时 arr中元素为 0, 1, 2, 3, 40
fmt.Println(s3) // arr中cap不够，go开辟新空间存储s3，内容为2, 3, 40, 50
fmt.Println(arr)// 0, 1, 2, 3, 40
```

综上:

```go
arr  := [5]int {0, 1, 2, 3, 4} // 数组
arr1 := []int{0, 1, 2, 3, 4}   // Slice
arr2 := [...]int{}             // Slice
arr3 := make([]int, 8)         // Slice len = 8
arr4 := make([]int, 8, 16)     // Slice len = 8 cap = 16
```

#### Map

Go中内置有Map这种结构，类似C/C++标准库中的std::map，创建一个map也非常简单，语法为:

```go
m := make(map[int]string) // 创建一个 key为int类型，value为string类型的map
// 遍历:
for k, v := range m {
  fmt.Println(k, v)
}
value, exist := m[10] // 如果key为10的健值对存在，则exist为true，否则为false

// 删除map中的元素
delete(m, 10) // 删除key为10的健值对
```

## 一些网络编程中比较常用的内置库

### 时间time

```go
// 定时器
// 200ms - 350ms 之间一个随机值醒来，timer.C管道将收到一个time的结构
timer := time.NewTimer(time.Duration(200) + rand.Int31n(150) * time.Millisecond)

// 重置定时器
timer.Reset(time.Duration(2000) * time.Millisecond) 

// 稳定的定时器，设定一个持续间隔，一直触发
ticker := timer.NewTicker(time.Duration(100) * time.Millisecond)
// 停止触发
ticker.Stop()

```


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

