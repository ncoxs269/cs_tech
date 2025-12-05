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
![[image-150.png]]

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

## 1.2 Golang方法集
Golang方法集 ：每个类型都有与之关联的方法集，这会影响到接口实现规则。
- 类型 T 方法集包含全部 receiver T 方法。
- 类型 \*T 方法集包含全部 receiver T + \*T 方法。
- 如类型 S 包含匿名字段 T，则 S 和 \*S 方法集包含 T 方法。
- 如类型 S 包含匿名字段 \*T，则 S 和 \*S 方法集包含 T + \*T 方法。
- 不管嵌入 T 或 \*T，\*S 方法集总是包含 T + \*T 方法。

`T` 可以调用 `*T` 的方法，但这是因为Go编译器会隐式地帮我们把 `T` 变成 `&T`，不代表 `T` 有 `*T` 的方法。

## 1.3 Go中各种类型\==比较
### 1.3.1 类型
golang 中的数据类型可以分为以下 4 大类：
1. 基本类型：整型（`int/uint/int8/uint8/int16/uint16/int32/uint32/int64/uint64/byte/rune`等）、浮点数（`float32/float64`）、复数类型（`complex64/complex128`）、字符串（`string`）。
2. 复合类型（又叫聚合类型）：数组和结构体类型。
3. 引用类型：切片（slice）、map、channel、指针。
4. 接口类型：如`error`。

### 1.3.2 基本类型
这是最简单的一种类型。比较操作也很简单，直接比较值是否相等。
需要特别注意，**尽量不要做浮点数比较，确实需要比较时，计算两个浮点数的差的绝对值，如果小于一定的值就认为它们相等，比如`1e-9`**。

### 1.3.3 复合类型
复合类型也叫做聚合类型。golang 中的复合类型只有两种：数组和结构体。它们是逐元素/字段比较的。
注意：**数组的长度视为类型的一部分，长度不同的两个数组是不同的类型，不能直接比较**。
- 如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的。对于数组来说，依次比较各个**元素**的值。所有元素全都相等，数组才是相等的。
- 如果一个结构体的字段类型是可以相互比较的，那么结构体类型也是可以相互比较的。对于结构体来说，依次比较各个**字段**的值。所有字段全都相等，结构体才是相等的。

### 1.3.4 引用类型
引用类型的比较实际判断的是两个变量是不是指向同一份数据，它不会去比较实际指向的数据。
**两个相同类型的channel可以使用\==运算符比较**。如果两个channel引用的是相同的对象，那么比较的结果为真。一个channel也可以和nil进行比较。指针也类似。
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

### 1.3.5 接口类型
接口类型的值，我们称为接口值。一个接口值是由两个部分组成的，具体类型（即该接口存储的值的类型）和该类型的一个值。分别称为**动态类型**和**动态值**。
接口值的比较涉及这两部分的比较，只有当**动态类型完全相同**且动态值相等（动态值使用`==`比较），两个接口值才是相等的。
**如果接口的动态值不可比较，强行比较会`panic`**。

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

### 1.3.6 使用`type`定义的类型
使用`type`可以基于现有类型定义新的类型。新类型会根据它们的底层类型来比较。

### 1.3.7 不可比较性
前面说过，golang 中的切片、map类型是不可比较的。所有含有切片的类型都是不可比较的。例如：
- 数组元素是切片类型。
- 结构体有切片类型的字段。
- 指针指向的是切片类型。

**不可比较性会传递，如果一个结构体由于含有切片字段不可比较，那么将它作为元素的数组不可比较，将它作为字段类型的结构体不可比较**。

## 1.4 for循环里append元素会死循环吗？
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

## 1.5 init和main函数
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

## 1.6 空结构体占不占内存空间？
空结构体是没有内存大小的结构体：
```Go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```
空结构体有一个特殊起点： `zerobase` 变量。`zerobase`是一个占用 8 个字节的`uintptr`全局变量。每次定义 `struct {}` 类型的变量，编译器只是把`zerobase`变量的地址给出去。也就是说空结构体的变量的内存地址都是一样的。

## 1.7 Go语言的接口类型是如何实现的？
在Go语言中，接口类型是通过**类型嵌入（embedding）** 的方式实现的。每个实现了接口的类型的结构体中都有一个隐含的成员，该成员是指向接口类型的指针。通过这种方式，接口实现了对类型的约束和定义。

## 1.8 make 和 new 的区别
`make`和`new`都是用于内存分配的内建函数，但它们在分配内存和初始化内存方面有所不同：
1. 分配对象的区别：
	- `new`可以分配任意类型的内存，并返回一个指向该类型的指针。
	- `make`专门用于分配`slice`、`map`和`channel`这三种内建类型，并返回一个引用类型本身。
2. 初始化的区别：
	1. new分配内存后，对于值类型，分配的是零值填充的内存空间，可直接使用。而对于引用类型，它的零值是nil，需要进一步初始化
	2. make在堆上分配内存后，内存会被初始化

## 1.9 nil接口和包含nil值的接口有什么区别
go的接口由两部分组成，一个是类型，一个是值。nil接口的类型和值都是nil，因此和nil做相等判断时，结果为true；而包含nil值的接口类型不是nil，因此和nil做相等判断时，结果为false。

# 2 并发
## 2.1 操作未初始化的 chan 会怎么样
未初始化的 chan 也就是 nil。对它进行读写会阻塞，close会panic。

## 2.2 操作已关闭的 chan 会怎么样
读会读到空值，写会panic，close会panic

## 2.3 无容量channel和有容量channel的区别
无容量channel在没有接受者or发送者时，会阻塞对应的发送者和接受者。因此它适用于发送和接收需要严格同步的情况。
有容量channel允许发送者不需要立即有接收者接收，适合需要高效数据传输并且不需要严格的同步的情况

## 2.4 `for`循环`select`时，如果通道已经关闭会怎么样？如果`select`中的`case`只有一个，又会怎么样？
- for循环`select`时，如果其中一个case通道已经关闭，则每次都会执行到这个case。在通道关闭后，通道能一直能读出内容(零值)。
- 如果select里边只有一个case，而这个case被关闭了，则会出现死循环。此时如果把它变成 nil，会一直阻塞

### 2.4.1 for循环里被关闭的通道
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

### 2.4.2 怎样才能不读关闭后通道
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

## 2.5 Go 中主协程如何等待其余协程
1. 使用`sync.WaitGroup`
2. 使用通道（Channel）：[[Go并发### 1.6.3 版本3：使用channel等待并发完成]]

## 2.6 Go chan的原理
[[Go并发原理与工具#2.5 总结]]

## 2.7 Go 锁的原理
互斥锁原理：[[Go并发原理与工具### 3.2.6 总结]]
读写锁原理：[[Go并发原理与工具### 3.3.3 总结]]

## 2.8 Go context原理
[[Go并发原理与工具## 1.5 总结]]

# 3 底层
## 3.1 Go GC
[[Go内存模型和GC## 4.5 总结]]

## 3.2 Go GMP
[[Go并发模型#1 面试题目]]

## 3.3 Go语言的栈空间管理
1. **栈的分配与增长**：每个goroutine都有自己的栈。起初，该栈的大小为2KB，随着goroutine运行，系统会动态地扩展栈
2. **栈的缩减**：当一个goroutine的栈经过一段时间后的使用情况显示其实际占用的内存已减少时，Go会尝试缩减栈的大小
3. **逃逸分析**：Go的编译器通过逃逸分析来决定变量的分配位置，根据变量是否“逃逸”到堆上。如果在函数内部不被返回就会分配在栈上；反之，如果被返回，则可能会分配在堆上。逃逸分析极大地影响了程序的性能，因为栈的分配和回收速度比堆要快得多。

## 3.4 为什么小对象多了会造成GC压力
1. 小对象通常会频繁分配和释放，这会导致 GC 的执行变得更为频繁
2. 小对象的频繁分配可能导致内存碎片化，导致可用的内存块变小，从而使得大对象的分配变得困难
3. 小对象数量庞大时，会增加 GC 的标记和清理时间

为了减轻 GC 的压力，可以考虑以下方法：
- 对象池（Object Pooling）：通过复用对象，减少对象的创建和销毁。
- 减少小对象分配的频率：优化代码，尽量使用大对象而不是多个小对象，来减少内存分配的频率。

## 3.5 defer原理
Go 的 defer 本质是**函数退出前（return 执行后、函数真正返回前）执行的延迟调用栈**。
defer 的生命周期分为 “注册” 和 “执行” 两个阶段：
1. **注册阶段**：代码执行到`defer`语句时，不会立即调用函数，而是将以下信息存入当前 Goroutine 的**defer 链表**（后进先出）中：
	- 待执行的函数地址（即 defer 后的函数）；
	- 函数的参数值（此时已确定，而非执行时确定）；
	- 函数的返回值接收者（若为方法）。
2. **触发阶段**：当所在函数满足 “退出条件” 时，触发 defer 执行。退出条件包括：
	- 函数执行到末尾（正常 return）；
	- 函数内发生`panic`；
	- 调用`runtime.Goexit()`主动退出 Goroutine。
3. **执行阶段**：从 defer 链表的**尾部（最新注册的 defer）开始**，依次取出并执行所有延迟函数，执行过程中会：
    - 若有`recover()`调用，会捕获之前的`panic`，恢复程序运行；
    - 若执行中再次发生`panic`，会终止当前 defer 执行，向上抛新`panic`。

从 Go runtime 源码（`runtime/runtime2.go`）来看，defer 的存储依赖两个核心结构：
1. **defer 结构体**：每个 defer 语句对应一个`_defer`实例，存储单个延迟调用的信息，关键字段包括：
    - `fn`：待执行的函数（`funcval`类型）；
    - `args`：函数参数的指针；
    - `link`：指向链表下一个`_defer`的指针（形成链表）。
2. **Goroutine 的 defer 链表**：每个 Goroutine（`g`结构体）中，有一个`_defer *defer`字段，指向当前 Goroutine 的 defer 链表头部。新注册的 defer 会通过`link`指针接入链表头部，执行时从头部依次取出（实现 LIFO）。

早期 defer 通过 “堆分配”`_defer`结构体实现，存在一定性能开销。Go 1.14 后引入**栈上分配 defer**优化，规则如下：
- 若 defer 在函数内可静态确定（无循环、无条件判断包裹），则`_defer`结构体分配在栈上，退出函数时自动释放，性能接近普通函数调用；
- 若 defer 在循环内（如`for`循环中的 defer）或条件判断内（如`if`中的 defer），仍需堆分配，性能略低。

## 3.6 为什么不能跨协程recover panic
每个goroutine 都有独立的调用栈。`panic` 发生时，会沿着调用栈向上查找，直到找到 `recover`，而 `recover` 只能在当前goroutine 的调用栈中查找，无法跨goroutine 查找。

## 3.7 slice底层结构和扩容原理
[[Go数据结构## 2.4 总结]]

## 3.8 map底层结构和扩容原理
[[Go数据结构#3.1 简介]]

# 4 问题排查和调优
## 4.1 Go在什么情况下会发生内存泄露
1. 资源未关闭
	1. 文件、网络连接、数据库连接等未正确关闭
	2. 这些资源通常关联底层系统资源，未关闭，Go 的 GC 无法回收其占用的内存
2. 协程泄露：
	1. 互斥锁未释放，其他协程获取锁时阻塞；
	2. 死锁，多个协程相互等待对方释放资源
	3. channel使用不当，例如对nil channel进行读写、无缓存channel发送、接收的请求数量不匹配。
	4. select 语句无 default 分支
	5. 定时器未stop
3. 切片或字符串的截取而引发的内存泄漏：
	1. 一个长的字符串被另一个字符串通过切片的方式截取，这两个字符串共用一个底层数组。
	2. 如果截取的字符串很小，但是原来的字符串很大，只要截取的小字符串还在活跃，则大的字符串将不会被回收，这样会造成临时的内存泄漏；
	3. 同理切片的截取也会存在这样的情况。
4. 全局变量或长生命周期对象的不当使用
	1. 全局变量（如 map、切片）或长生命周期对象（如单例）中存储了不再使用的数据，且未及时清理

# 5 应用
## 5.1 翻转含有中文、数字、英文字母的字符串
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

## 5.2 reflect（反射包）如何获取字段tag？为什么json包不能导出私有变量的tag？
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

## 5.3 怎么实现接口的延迟调用
注意是**延迟**，不是**延时**。
在 Golang 中实现接口的延迟调用，核心是将接口方法的调用逻辑封装起来（不立即执行），待后续指定时机（如异步处理、条件满足、批量执行时）再触发执行。

一个方案是闭包，适合已知接口 / 方法的场景，编译期就能做类型检查，无运行时风险，性能也更高。
```go
package main

import "fmt"

// 1. 定义示例接口
type Calculator interface {
	Add(a, b int) int
	Multiply(a, b int) int
}

// 2. 实现接口的结构体
type BasicCalculator struct{}

func (bc BasicCalculator) Add(a, b int) int {
	fmt.Println("执行Add方法（闭包版）")
	return a + b
}

func (bc BasicCalculator) Multiply(a, b int) int {
	fmt.Println("执行Multiply方法（闭包版）")
	return a * b
}

// 3. 定义延迟调用函数类型（按需封装返回值）
type DelayCallFunc func() int

func main() {
	// 初始化接口实例
	calc := BasicCalculator{}

	// 4. 封装延迟调用逻辑（闭包捕获参数和实例，不立即执行）
	var delayAdd DelayCallFunc = func() int {
		return calc.Add(3, 5) // 绑定参数：3和5
	}

	// 模拟「延迟」：先执行其他业务逻辑
	fmt.Println("执行前置业务逻辑...")

	// 5. 触发延迟调用（真正执行Add方法）
	result := delayAdd()
	fmt.Println("Add调用结果：", result)

	// 同理封装Multiply的延迟调用
	var delayMultiply DelayCallFunc = func() int {
		return calc.Multiply(4, 6)
	}
	fmt.Println("执行更多前置逻辑...")
	result2 := delayMultiply()
	fmt.Println("Multiply调用结果：", result2)
}
```

另一种是反射实现，适合通用化的延迟调用组件（如框架、中间件），但有运行时类型检查和性能开销（大概是10-100倍）。
```go
package main

import (
	"fmt"
	"reflect"
)

// 1. 复用上述Calculator接口和BasicCalculator实现

// 2. 定义延迟调用上下文结构体：保存接口实例、方法名、参数
type DelayCall struct {
	instance reflect.Value  // 接口实例的反射值
	method   string         // 要调用的方法名
	args     []reflect.Value // 方法参数的反射值切片
}

// 3. 构造延迟调用实例（入参：接口实例、方法名、方法参数）
func NewDelayCall(instance interface{}, method string, args ...interface{}) *DelayCall {
	// 将普通参数转换为reflect.Value切片
	argValues := make([]reflect.Value, len(args))
	for i, arg := range args {
		argValues[i] = reflect.ValueOf(arg)
	}

	return &DelayCall{
		instance: reflect.ValueOf(instance),
		method:   method,
		args:     argValues,
	}
}

// 4. 执行延迟调用（返回方法结果和错误）
func (dc *DelayCall) Call() (interface{}, error) {
	// 获取接口方法的反射值
	method := dc.instance.MethodByName(dc.method)
	if !method.IsValid() {
		return nil, fmt.Errorf("方法 %s 不存在或未导出（首字母需大写）", dc.method)
	}

	// 检查参数数量匹配
	if method.Type().NumIn() != len(dc.args) {
		return nil, fmt.Errorf("参数数量不匹配：期望 %d 个，实际 %d 个", method.Type().NumIn(), len(dc.args))
	}

	// 执行方法调用
	results := method.Call(dc.args)
	if len(results) == 0 {
		return nil, nil // 无返回值
	}

	// 返回方法的第一个返回值（示例中方法仅一个返回值）
	return results[0].Interface(), nil
}

func main() {
	// 初始化接口实例
	calc := BasicCalculator{}

	// 构造延迟调用：Add(3,5)
	delayAdd := NewDelayCall(calc, "Add", 3, 5)

	// 模拟延迟：执行其他逻辑
	fmt.Println("执行前置业务逻辑（反射版）...")

	// 触发延迟调用
	result, err := delayAdd.Call()
	if err != nil {
		fmt.Println("调用失败：", err)
		return
	}
	fmt.Println("Add调用结果（反射）：", result)

	// 构造Multiply的延迟调用
	delayMultiply := NewDelayCall(calc, "Multiply", 4, 6)
	fmt.Println("执行更多前置逻辑（反射版）...")
	result2, err := delayMultiply.Call()
	if err != nil {
		fmt.Println("调用失败：", err)
		return
	}
	fmt.Println("Multiply调用结果（反射）：", result2)
}
```

## 5.4 A服务调B服务，给B服务的处理时间只有200ms，怎么实现？
可以用context的取消
这个取消应该写在A里面还是B里面呢？其实应该写在A里面

---
# 6 引用
引用传递：https://zhuanlan.zhihu.com/p/542218435
\==比较：https://juejin.cn/post/6844903923166232589#heading-6