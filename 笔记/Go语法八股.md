2025-04-02 21:22
Status: #idea
Tags: [[Go]]

# 1 语法
## 1.1 Go是值传递还是引用传递
先说结论，Go里面没有`引用传递`，Go语言是`值传递`。Go语言中的一些让你觉得它是`引用传递`的原因，是因为Go语言有`值类型`和`引用类型`，但是它们都是`值传递`。
Go中类型的定义：
- **值类型** - int、float、bool、string、array、struct等
- **引用类型** - slice，map，channel，interface，func等
	- 严格来说，Go 语言没有引用类型，但是我们可以把 `map、chan、func、interface、slice` 称为引用类型，这样便于理解。
	- 指针类型也可以理解为是一种引用类型

传递的定义：
- 值传递：指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。
- 引用传递：指在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。
更专业的说法：
- 值传递：被调函数的形式参数作为被调函数的局部变量处理，即在堆栈中开辟了内存空间以存放由主调函数放进来的实参的值，从而成为了实参的一个副本。值传递的特点是被调函数对形式参数的任何操作都是作为局部变量进行，不会影响主调函数的实参变量的值。
- 引用传递：被调函数的形式参数虽然也作为局部变量在堆栈中开辟了内存空间，但是这时存放的是由主调函数放进来的实参变量的地址。被调函数对形参的任何操作都被处理成间接寻址，即通过堆栈中存放的地址访问主调函数中的实参变量。正因为如此，被调函数对形参做的任何操作都影响了主调函数中的实参变量。

> [!info] Go官网的说法
> 像 C 家族中的其他所有语言一样，Go 语言中的所有传递都是传值。也就是说，函数接收到的永远都是参数的一个副本，就好像有一条将值赋值给参数的赋值语句一样。例如，传递一个 int 值给一个函数，函数收到的是这个 int 值得副本，传递指针值，获得的是指针值的副本，而不是指针指向的数据。
> 
> Map 和 Slice 的值表现和指针一样：它们是对内部映射或者切片数据的指针的描述符。复制 Map 或者 Slice 的值，不会复制它们指向的数据。
> 复制接口的值，会产生一个接口值存储的数据的副本。
 >- 如果接口值存储的是一个结构体，复制接口值将产生一个结构体的副本。
 >- 如果接口值存储的是指针，复制接口值会产生一个指针的副本，而不是指针指向的数据的副本。

### 1.1.1 cpp中引用的概念
简单说下cpp中的引用概念吧，引用的底层是会占据内存的，引用的本质就是一个指针变量，只不过是一个常量指针。
比如：int a = 2; int &b = a; b的本质是一个int const \*的类型，他是有地址的，只不过你无法看到当你对b取地址：&b，编译器会帮你自动翻译成a的地址，所以看着b好像仅仅只是a的别名，但底层他就是一个常指针而已。编译器的翻译是通过标记b为引用类型，并在符号表中建立b到a的地址的映射关系来实现的。
大概解释清楚了引用的底层后，我在简单说下我对于go中的引用类型的理解吧，我觉得不像作者说的那样go是没有引用传递的，而是引用类型的底层就是一个指针，常指针而已。引用和指针的本质区别就是引用是只读的，指针是可读写的，你可以修改指针的指向，但不能修改引用的指向。
函数的传参也是如此，无论是指针还是引用，都会发生一次拷贝，你在函数内部对指针的修改是无效的（除非你用二级指针），但你对指针指向的修改是有效的，因为拷贝后的内容是一样的。对于引用而言，你是无法修改指针的本身（即使二级引用也不行），但能通过像指针的方式来修改指向的内容。
go的引用类型就是这个意思，你没法改变它的指向，但能修改他的内容。

### 1.1.2 Go的引用类型
对于Map类型来说，可能觉得还是迷惑，一来我们可以通过函数修改它的内容，二来它没有明显的指针。
通过查看源码我们可以看到，实际上`make`底层调用的是`makemap`函数，主要做的工作就是初始化`hmap`结构体的各种字段：
```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    //...
}
```
通过查看`src/runtime/hashmap.go`源代码发现，`make`函数返回的是一个`hmap`类型的指针`*hmap`。也就是说`map===*hmap`。

`chan`类型本质上和`map`类型是一样的，这里不做过多的介绍，参考下源代码:
```go
func makechan(t *chantype, size int64) *hchan {
    //...
}
```

slice虽然也是引用类型，但是它又有点不一样。slice本身是个结构体，但它内部第一个元素是一个指针类型，指向底层的具体数组。slice在传递时，其内部指针的值也被拷贝了，都是指向同一个数组。
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
下面举个例子：
```go
func main() {
    arr := make([]int, 0)
    arr = append(arr, 1, 2, 3)
    fmt.Printf("outer1: %p, %p\n", &arr, &arr[0])
    modify(arr)
    fmt.Println(arr)  // 10, 2, 3
}

func modify(arr []int) {
    fmt.Printf("inner1: %p, %p\n", &arr, &arr[0])
    arr[0] = 10
    fmt.Printf("inner2: %p, %p\n", &arr, &arr[0])
}

//输出：
//outer1: 0x14000112018, 0x14000134000
//inner1: 0x14000112030, 0x14000134000
//inner2: 0x14000112030, 0x14000134000
//[10 2 3]
```
可以看到，在函数内外，arr本身的地址`&arr`变了，但是两个指针指向的底层数据，也就是`&arr[0]`数组首元素的地址是不变的。

## 1.2 Go中各种类型\==比较
### 1.2.1 类型
golang 中的数据类型可以分为以下 4 大类：
1. 基本类型：整型（`int/uint/int8/uint8/int16/uint16/int32/uint32/int64/uint64/byte/rune`等）、浮点数（`float32/float64`）、复数类型（`complex64/complex128`）、字符串（`string`）。
2. 复合类型（又叫聚合类型）：数组和结构体类型。
3. 引用类型：切片（slice）、map、channel、指针。
4. 接口类型：如`error`。

### 1.2.2 基本类型
这是最简单的一种类型。比较操作也很简单，直接比较值是否相等。
需要特别注意，**尽量不要做浮点数比较，确实需要比较时，计算两个浮点数的差的绝对值，如果小于一定的值就认为它们相等，比如`1e-9`**。

### 1.2.3 复合类型
复合类型也叫做聚合类型。golang 中的复合类型只有两种：数组和结构体。它们是逐元素/字段比较的。
注意：**数组的长度视为类型的一部分，长度不同的两个数组是不同的类型，不能直接比较**。
- 如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的。对于数组来说，依次比较各个**元素**的值。所有元素全都相等，数组才是相等的。
- 如果一个结构体的字段类型是可以相互比较的，那么结构体类型也是可以相互比较的。对于结构体来说，依次比较各个**字段**的值。所有字段全都相等，结构体才是相等的。

### 1.2.4 引用类型
引用类型的比较实际判断的是两个变量是不是指向同一份数据，它不会去比较实际指向的数据。
关于引用类型，有两个比较特殊的规定：
- 切片之间不允许比较。切片只能与`nil`值比较。
- `map`之间不允许比较。`map`只能与`nil`值比较。

因为切片是引用类型，它可以间接的指向自己。例如：
```Go
a := []interface{}{ 1, 2.0 }
a[1] = a
fmt.Println(a)

// !!!
// runtime: goroutine stack exceeds 1000000000-byte limit
// fatal error: stack overflow
```
- 切片如果直接比较引用地址，是不合适的。首先，切片与数组是比较相近的类型，比较方式的差异会造成使用者的混淆。另外，长度和容量是切片类型的一部分，不同长度和容量的切片如何比较？
- 切片如果像数组那样比较里面的元素，又会出现上来提到的循环引用的问题。虽然可以在语言层面解决这个问题，但是 golang 团队认为不值得为此耗费精力。
基于上面两点原因，golang 直接规定**切片类型不可比较**。使用`==`比较切片直接编译报错。

因为`map`的值类型可能为不可比较类型，所以`map`类型也不可比较。

### 1.2.5 接口类型
接口类型的值，我们称为接口值。一个接口值是由两个部分组成的，具体类型（即该接口存储的值的类型）和该类型的一个值。分别称为**动态类型**和**动态值**。
接口值的比较涉及这两部分的比较，只有当**动态类型完全相同**且动态值相等（动态值使用`==`比较），两个接口值才是相等的。
如果接口的动态值不可比较，强行比较会`panic`。

**接口值的比较不要求接口类型（注意不是动态类型）完全相同，只要一个接口可以转化为另一个就可以比较**。例如：
```Go
var f *os.File

var r io.Reader = f
var rc io.ReadCloser = f
fmt.Println(r == rc) // true

var w io.Writer = f
// invalid operation: r == w (mismatched types io.Reader and io.Writer)
fmt.Println(r == w)
```

### 1.2.6 使用`type`定义的类型
使用`type`可以基于现有类型定义新的类型。新类型会根据它们的底层类型来比较。

### 1.2.7 不可比较性
前面说过，golang 中的切片类型是不可比较的。所有含有切片的类型都是不可比较的。例如：
- 数组元素是切片类型。
- 结构体有切片类型的字段。
- 指针指向的是切片类型。

**不可比较性会传递，如果一个结构体由于含有切片字段不可比较，那么将它作为元素的数组不可比较，将它作为字段类型的结构体不可比较**。

## 1.3 for循环里append元素
代码：
```Go
package main  
  
import "fmt"  
  
func main() {  
 s := []int{1,2,3,4,5}  
 for _, v:=range s {  
  s =append(s, v)  
  fmt.Printf("len(s)=%v\n",len(s))  
 }  
}
```
**这个代码会造成死循环吗？**
**不会死循环**，`for range`其实是`golang`的`语法糖`，在循环开始前会获取切片的长度 `len(切片)`，然后再执行`len(切片)`次数的循环。
上面的代码会被编译器认为是:
```Go
func main() {  
 s := []int{1,2,3,4,5}  
 for_temp := s  
 len_temp := len(for_temp)  
 for index_temp := 0; index_temp < len_temp; index_temp++ {  
  value_temp := for_temp[index_temp]  
  _ = index_temp  
  v := value_temp  
  // 以下是 original body  
  s =append(s, value)  
  fmt.Printf("len(s)=%v\n",len(s))  
 }  
}
```

## 1.4 init和main函数
init函数（没有输入参数、返回值）的主要作用：
- 可以在init里面初始化不能采用初始化表达式初始化的变量。
- 程序运行前的注册。
- 实现sync.Once功能。
- 其他

init顺序：
1、在同一个 package 中，可以多个文件中定义 init 方法
2、**在同一个 go 文件中，可以重复定义 init 方法**
3、**在同一个 package 中，不同文件中的 init 方法的执行按照文件名先后执行各个文件中的 init 方法**
4、在同一个文件中的多个 init 方法，按照在代码中编写的顺序依次执行不同的 init 方法
5、对于不同的 package，如果不相互依赖的话，按照 main 包中 import 的顺序调用其包中的 init() 函数
6、如果 package 存在依赖，调用顺序为**最后被依赖的最先被初始化**，例如：导入顺序 main –> A –> B –> C，则初始化顺序为 C –> B –> A –> main，一次执行对应的 init 方法。

## 1.5 空结构体占不占内存空间？
空结构体是没有内存大小的结构体：
```Go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```
空结构体有一个特殊起点： `zerobase` 变量。`zerobase`是一个占用 8 个字节的`uintptr`全局变量。每次定义 `struct {}` 类型的变量，编译器只是把`zerobase`变量的地址给出去。也就是说空结构体的变量的内存地址都是一样的。

## 1.6 Go语言的接口类型是如何实现的？
在Go语言中，接口类型是通过**类型嵌入（embedding）**的方式实现的。每个实现了接口的类型的结构体中都有一个隐含的成员，该成员是指向接口类型的指针。通过这种方式，接口实现了对类型的约束和定义。

## 1.7 make 和 new 的区别
`make`和`new`都是用于内存分配的内建函数，但它们在分配内存和初始化内存方面有所不同：
1. 分配内存的区别：
	- `new`可以分配任意类型的内存，并返回一个指向该类型的指针。
	- `make`专门用于分配`slice`、`map`和`channel`这三种内建类型，并返回一个引用类型本身。
2. 初始化的区别：
	1. new分配内存后，对于值类型，分配的是零值填充的内存空间，可直接使用。而对于引用类型，它的零值是nil，需要进一步初始化。例如new(int)之后，对应内存空间会有一个0；而new(map[string]int)，对应内存空间是nil，而不是{"":0}
	2. make在堆上分配内存后，内存会被初始化，这意味着切片的长度和容量会被设置（默认情况下长度为0）；Map会初始化为空（没有任何键值对）；chanel 会处于准备就绪状态，可以立即用于发送和接收数据。
3. 返回类型的区别：
    - `new`返回的是指针类型。
    - `make`返回的是与参数相同类型的值，而不是指针。
4. 语法上的区别：
    - `new`的语法是 `func new(Type) *Type`。
    - `make`的语法是 `func make(t Type, size ...IntegerType) Type`。

## 1.8 为什么不能跨协程recover panic
因为 `panic` 和 `recover` 都是针对单个goroutine 的，每个goroutine 都有独立的调用栈。`panic` 发生时，会沿着调用栈向上查找，直到找到 `recover`，而 `recover` 只能在当前goroutine 的调用栈中查找，无法跨goroutine 查找。

# 2 并发
## 2.1 操作未初始化的 chan 会怎么样
未初始化的 chan 也就是 nil。对它进行读写会阻塞，close会panic。

## 2.2 `for`循环`select`时，如果通道已经关闭会怎么样？如果`select`中的`case`只有一个，又会怎么样？
- for循环`select`时，如果其中一个case通道已经关闭，则每次都会执行到这个case。在通道关闭后，通道能一直能读出内容(零值)。
- 如果select里边只有一个case，而这个case被关闭了，则会出现死循环。此时如果把它变成 nil，会一直阻塞

### 2.2.1 for循环里被关闭的通道
代码：
```Go
package main

import (
	"fmt"
	"time"
)

func main() {
	const fmat = "2006-01-02 15:04:05"

	c := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		c <- 10
		close(c)
	}()

	for {
		select {
		case x, ok := <-c:
			fmt.Printf("%v, 通道读取到：x=%v,ok=%v\n", time.Now().Format(fmat), x, ok)
			time.Sleep(1 * time.Second)

		default:
			fmt.Printf("%v, 没读到信息进入default\n", time.Now().Format(fmat))
			time.Sleep(1 * time.Second)
		}
	}
}
```
输出：
```text
2025-04-15 09:50:49, 没读到信息进入default
2025-04-15 09:50:50, 通道读取到：x=10,ok=true
2025-04-15 09:50:51, 通道读取到：x=0,ok=false
2025-04-15 09:50:52, 通道读取到：x=0,ok=false
2025-04-15 09:50:53, 通道读取到：x=0,ok=false
```

### 2.2.2 怎样才能不读关闭后通道
代码：
```Go
package main

import (
	"fmt"
	"time"
)

func main() {
	const fmat = "2006-01-02 15:04:05"

	c := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		c <- 10
		close(c)
	}()

	for {
		select {
		case x, ok := <-c:
			fmt.Printf("%v, 通道读取到：x=%v,ok=%v\n", time.Now().Format(fmat), x, ok)
			time.Sleep(1 * time.Second)

			if !ok {
				c = nil
			}

		default:
			fmt.Printf("%v, 没读到信息进入default\n", time.Now().Format(fmat))
			time.Sleep(1 * time.Second)
		}
	}
}
```
效果：
```Go
2025-04-15 09:59:18, 没读到信息进入default
2025-04-15 09:59:19, 通道读取到：x=10,ok=true
2025-04-15 09:59:20, 通道读取到：x=0,ok=false
2025-04-15 09:59:21, 没读到信息进入default
2025-04-15 09:59:22, 没读到信息进入default
2025-04-15 09:59:23, 没读到信息进入default
```
或者用break退出for循环。

# 3 应用
## 3.1 翻转含有中文、数字、英文字母的字符串
实现：
```go
package main

import"fmt"

func main() {
 src := "你好abc啊哈哈"
 dst := reverse([]rune(src))
 fmt.Printf("%v\n", string(dst))
}

func reverse(s []rune) []rune {
 for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
  s[i], s[j] = s[j], s[i]
 }
 return s
}
```

## 3.2 reflect（反射包）如何获取字段tag？为什么json包不能导出私有变量的tag？
例子：
```Go
package main  
  
import (  
    "fmt"  
    "reflect"  
)  
  
type J struct {  
    a string //小写无tag  
    b string `json:"B"` //小写+tag  
    C string //大写无tag  
    D string `json:"DD" otherTag:"good"` //大写+tag  
}  

func printTag(stru interface{}) {
	// Type.Elem() 方法返回它持有元素的 Type，只能用在数组、chan、map、slice、指针上面
    t := reflect.TypeOf(stru).Elem()  
    for i := 0; i < t.NumField(); i++ {  
        fmt.Printf("结构体内第%v个字段 %v 对应的json tag是 %v , 还有otherTag？ = %v \n", i+1, t.Field(i).Name, t.Field(i).Tag.Get("json"), t.Field(i).Tag.Get("otherTag"))  
 }  
}  
  
func main() {  
    j := J{  
      a: "1",  
      b: "2",  
      C: "3",  
      D: "4",  
    }  
    printTag(&j)  
}
```
输出:
```text
结构体内第1个字段 a 对应的json tag是  , 还有otherTag？ =    
结构体内第2个字段 b 对应的json tag是 B , 还有otherTag？ =    
结构体内第3个字段 C 对应的json tag是  , 还有otherTag？ =    
结构体内第4个字段 D 对应的json tag是 DD , 还有otherTag？ = good
```

`json`包里不能导出私有变量的`tag`是因为`json`包里认为私有变量为不可导出的`Unexported`，所以**跳过获取**名为`json`的`tag`的内容。

# 4 Go 中主协程如何等待其余协程
1. 使用`sync.WaitGroup`
2. 使用通道（Channel）
3. 使用`context`包

---
# 5 引用
引用传递：https://zhuanlan.zhihu.com/p/542218435
\==比较：https://juejin.cn/post/6844903923166232589#heading-6