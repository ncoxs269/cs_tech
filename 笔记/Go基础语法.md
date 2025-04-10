2025-03-24 21:58
Status: #idea
Tags: [[Go]]

# 1 程序结构
## 1.1 变量
### 1.1.1 指针
**在Go语言中，返回函数中局部变量的地址也是安全的**。例如下面的代码，调用f函数时创建局部变量v，在局部变量地址被返回之后依然有效，因为指针p依然引用这个变量。
```go
var p = f()

func f() *int {
    v := 1
    return &v
}
```
每次调用f函数都将返回不同的结果：
```go
fmt.Println(f() == f()) // "false"
```

### 1.1.2 new函数
另一个创建变量的方法是调用内建的new函数。表达式new(T)将创建一个T类型的匿名变量，初始化为T类型的零值，然后返回变量地址，返回的指针类型为`*T`。
```go
p := new(int)   // p, *int 类型, 指向匿名的 int 变量
fmt.Println(*p) // "0"
*p = 2          // 设置 int 匿名变量的值为 2
fmt.Println(*p) // "2"
```

用new创建变量和普通变量声明语句方式创建变量没有什么区别，除了不需要声明一个临时变量的名字外，我们还可以在表达式中使用new(T)。换言之，new函数类似是一种语法糖，而不是一个新的基础概念。
下面的两个newInt函数有着相同的行为：
```go
func newInt() *int {
    return new(int)
}

func newInt() *int {
    var dummy int
    return &dummy
}
```

**当然也可能有特殊情况：如果两个类型都是空的，也就是说类型的大小是0，例如`struct{}`和`[0]int`，有可能有相同的地址（依赖具体的语言实现）**（译注：请谨慎使用大小为0的类型，因为如果类型的大小为0的话，可能导致Go语言的自动垃圾回收器有不同的行为，具体请查看`runtime.SetFinalizer`函数相关文档）。

### 1.1.3 生命周期
那么Go语言的自动垃圾收集器是如何知道一个变量是何时可以被回收的呢？这里我们可以避开完整的技术细节，基本的实现思路是，从每个包级的变量和每个当前运行函数的每一个局部变量开始，通过指针或引用的访问路径遍历，是否可以找到该变量。如果不存在这样的访问路径，那么说明该变量是不可达的，也就是说它是否存在并不会影响程序后续的计算结果。
**编译器会自动选择在栈上还是在堆上分配局部变量的存储空间**，但可能令人惊讶的是，这个选择并不是由用var还是new声明变量的方式决定的。
```go
var global *int

func f() {
    var x int
    x = 1
    global = &x
}

func g() {
    y := new(int)
    *y = 1
}
```
f函数里的x变量必须在堆上分配，因为它在函数退出后依然可以通过包一级的global变量找到，虽然它是在函数内部定义的；用Go语言的术语说，这个x局部变量从函数f中**逃逸**了。相反，当g函数返回时，变量`*y`将是不可达的，也就是说可以马上被回收的。因此，`*y`并没有从函数g中逃逸，编译器可以选择在栈上分配`*y`的存储空间（译注：也可以选择在堆上分配，然后由Go语言的GC回收这个变量的内存空间）

## 1.2 包和文件
### 1.2.1 包注释
在每个源文件的包声明前紧跟着的注释是包注释。通常，包注释的第一句应该先是包的功能概要说明。一个包通常只有一个源文件有包注释（如果有多个包注释，目前的文档工具会根据源文件名的先后顺序将它们链接为一个包注释）。如果包注释很大，通常会放到一个独立的`doc.go`文件中。

### 1.2.2 内部包
Go语言的构建工具对包含**internal**名字的路径段的包导入路径做了特殊处理。这种包叫internal包，一个internal包只能被和internal目录有同一个父目录的包所导入。
例如，net/http/internal/chunked内部包只能被net/http/httputil或net/http包导入，但是不能被net/url包导入。不过net/url包却可以导入net/http/httputil包。

# 2 基础数据类型
## 2.1 整型
Go语言同时提供了有符号和无符号类型的整数运算。这里有int8、int16、int32和int64四种截然不同大小的有符号整数类型，分别对应8、16、32、64bit大小的有符号整数，与此对应的是uint8、uint16、uint32和uint64四种无符号整数类型。
这里还有两种一般对应特定CPU平台机器字大小的有符号和无符号整数int和uint；其中int是应用最广泛的数值类型。这两种类型都有同样的大小，32或64bit，但是我们不能对此做任何的假设；因为不同的编译器即使在相同的硬件平台上可能产生不同的大小。
**Unicode字符rune类型是和int32等价的类型**，通常用于表示一个Unicode码点。这两个名称可以互换使用。**同样byte也是uint8类型的等价类型**，byte类型一般用于强调数值是一个原始的数据而不是一个小的整数。
最后，**还有一种无符号的整数类型uintptr，没有指定具体的bit大小但是足以容纳指针**。uintptr类型只有在底层编程时才需要，特别是Go语言和C语言函数库或操作系统接口相交互的地方。

## 2.2 浮点数
一个float32类型的浮点数可以提供大约6个十进制数的精度，而float64则可以提供约15个十进制数的精度；通常应该优先使用float64类型，因为float32类型的累计计算误差很容易扩散，并且float32能精确表示的正整数并不是很大（译注：因为float32的有效bit位只有23个，其它的bit位用于指数和符号；当整数大于23bit能表达的范围时，float32的表示将出现误差）

## 2.3 复数
Go语言提供了两种精度的复数类型：complex64和complex128，分别对应float32和float64两种浮点数精度。内置的complex函数用于构建复数，内建的real和imag函数分别返回复数的实部和虚部：
```go
var x complex128 = complex(1, 2) // 1+2i
var y complex128 = complex(3, 4) // 3+4i
fmt.Println(x*y)                 // "(-5+10i)"
fmt.Println(real(x*y))           // "-5"
fmt.Println(imag(x*y))           // "10"
```

如果一个浮点数面值或一个十进制整数面值后面跟着一个i，例如3.141592i或2i，它将构成一个复数的虚部，复数的实部是0：
```go
fmt.Println(1i * 1i) // "(-1+0i)", i^2 = -1
```

在常量算术规则下，一个复数常量可以加到另一个普通数值常量（整数或浮点数、实部或虚部），我们可以用自然的方式书写复数，就像1+2i或与之等价的写法2i+1。上面x和y的声明语句还可以简化：
```go
x := 1 + 2i
y := 3 + 4i
```

复数也可以用\==和!=进行相等比较。只有两个复数的实部和虚部都相等的时候它们才是相等的（译注：浮点数的相等比较是危险的，需要特别小心处理精度问题）。

## 2.4 字符串
### 2.4.1 不可变
因为字符串是不可修改的，因此尝试修改字符串内部数据的操作也是被禁止的。
当你将一个字符串变量 `a` 赋值给另一个字符串变量 `b` 时，**`b` 和 `a` 会共享同一块底层存储**，而不会复制字符串的内容。
Go 的字符串在底层是一个结构体，包含两个字段：
1. 一个指向底层字节数组（`[]byte`）的指针。
2. 字符串的长度（`int`）。
当你执行 `b := a` 时，Go 只是复制了这个结构体（即复制了指针和长度），而不会复制底层的字节数组。因此，`a` 和 `b` 会指向同一块内存数据。
```Go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	a := "hello"
	b := a

	// 获取 a 和 b 底层数据的指针
	ptrA := (*uintptr)(unsafe.Pointer(&a))
	ptrB := (*uintptr)(unsafe.Pointer(&b))

	fmt.Printf("a: %v, pointer: %v\n", a, *ptrA)
	fmt.Printf("b: %v, pointer: %v\n", b, *ptrB)

	// 判断底层指针是否相同
	fmt.Println("底层数据指针相同:", *ptrA == *ptrB)
}
```
运行结果：
```bash
a: hello, pointer: 0x10c0a30  (具体值可能不同)
b: hello, pointer: 0x10c0a30
底层数据指针相同: true
```

同样，一个字符串s和对应的子字符串切片s[7:]的操作也可以安全地共享相同的内存，因此字符串切片操作代价也是低廉的。在这两种情况下都没有必要分配新的内存。
### 2.4.2 字符串面值
一个原生的字符串面值形式是`...`，使用反引号代替双引号。在原生的字符串面值中，没有转义操作；全部的内容都是字面的意思，包含退格和换行，因此一个程序中的原生字符串面值可能跨越多行（译注：在原生字符串面值内部是无法直接写\`字符的，可以用八进制或十六进制转义或+"\`"连接字符串常量完成）。唯一的特殊处理是会删除回车以保证在所有平台上的值都是一样的，包括那些把回车也放入文本文件的系统（译注：Windows系统会把回车和换行一起放入文本文件中）。
原生字符串面值用于编写正则表达式会很方便，因为正则表达式往往会包含很多反斜杠。原生字符串面值同时被广泛应用于HTML模板、JSON面值、命令行提示信息以及那些需要扩展到多行的场景。
```Go
const GoUsage = `Go is a tool for managing Go source code.

Usage:
    go command [arguments]
...`
```

### 2.4.3 Go中的字符编码
基础知识参见 [[字符编码]]
Go语言的源文件采用UTF8编码，并且Go语言处理UTF8编码的文本也很出色。unicode包提供了诸多处理rune字符相关功能的函数（比如区分字母和数字，或者是字母的大写和小写转换等），unicode/utf8包则提供了用于rune字符序列的UTF8编码和解码的功能。

Go语言字符串面值中的Unicode转义字符让我们可以通过Unicode码点输入特殊的字符。有两种形式：`\uhhhh`对应16bit的码点值，`\Uhhhhhhhh`对应32bit的码点值，其中h是一个十六进制数字。每一个对应码点的UTF8编码。例如：下面的字母串面值都表示相同的值：
```Go
"世界"
"\xe4\xb8\x96\xe7\x95\x8c"
"\u4e16\u754c"
"\U00004e16\U0000754c"
```
Unicode转义也可以使用在rune字符中。下面三个字符是等价的：
```Go
'世' '\u4e16' '\U00004e16'
```
对于小于256的码点值可以写在一个十六进制转义字节中，例如`\x41`对应字符'A'，但是对于更大的码点则必须使用`\u`或`\U`转义形式。因此，`\xe4\xb8\x96`并不是一个合法的rune字符，虽然这三个字节对应一个有效的UTF8编码的码点。

下面的字符串包含13个字节，以UTF8形式编码，但是只对应9个Unicode字符：
```Go
import "unicode/utf8"

s := "Hello, 世界"
fmt.Println(len(s))                    // "13"
fmt.Println(utf8.RuneCountInString(s)) // "9"
```
为了处理这些真实的字符，我们需要一个UTF8解码器。unicode/utf8包提供了该功能，我们可以这样使用：
```Go
for i := 0; i < len(s); {
    r, size := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%d\t%c\n", i, r)
    i += size
}
```
幸运的是，Go语言的range循环在处理字符串的时候，会自动隐式解码UTF8字符串：
```Go
for i, r := range "Hello, 世界" {
    fmt.Printf("%d\t%q\t%d\n", i, r, r)
}
```

如果用于循环的字符串只是一个普通的二进制数据，或者是含有错误编码的UTF8数据，将会发生什么呢？
每一个UTF8字符解码，不管是显式地调用utf8.DecodeRuneInString解码或是在range循环中隐式地解码，如果遇到一个错误的UTF8编码输入，将生成一个特别的Unicode字符`\uFFFD`，在印刷中这个符号通常是一个黑色六角或钻石形状，里面包含一个白色的问号"?"。

将[]rune类型转换应用到UTF8编码的字符串，将返回字符串编码的Unicode码点序列：
```Go
// "program" in Japanese katakana
s := "プログラム"
fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
r := []rune(s)
fmt.Printf("%x\n", r)  // "[30d7 30ed 30b0 30e9 30e0]"
```
（在第一个Printf中的`% x`参数用于在每个十六进制数字前插入一个空格。）

如果是将一个[]rune类型的Unicode字符slice或数组转为string，则对它们进行UTF8编码：
```Go
fmt.Println(string(r)) // "プログラム"
```
将一个整数转型为字符串意思是生成以只包含对应Unicode码点字符的UTF8字符串：
```Go
fmt.Println(string(65))     // "A", not "65"
fmt.Println(string(0x4eac)) // "京"
```

## 2.5 常量
### 2.5.1 基础
常量间的所有算术运算、逻辑运算和比较运算的结果也是常量，对常量的类型转换操作或以下函数调用都是返回常量结果：len、cap、real、imag、complex和unsafe.Sizeof。
因为它们的值是在编译期就确定的，因此常量可以是构成类型的一部分，例如用于指定数组类型的长度：
```Go
const IPv4Len = 4

// parseIPv4 parses an IPv4 address (d.d.d.d).
func parseIPv4(s string) IP {
    var p [IPv4Len]byte
    // ...
}
```

如果是批量声明的常量，除了第一个外其它的常量右边的初始化表达式都可以省略，如果省略初始化表达式则表示使用前面常量的初始化表达式写法，对应的常量类型也一样的。例如：
```Go
const (
    a = 1
    b
    c = 2
    d
)

fmt.Println(a, b, c, d) // "1 1 2 2"
```

### 2.5.2 iota 常量生成器
在一个const声明语句中，在第一个声明的常量所在的行，iota将会被置为0，然后在每一个有常量声明的行加一。
下面是来自time包的例子，它首先定义了一个Weekday命名类型，然后为一周的每天定义了一个常量，从周日0开始。在其它编程语言中，这种类型一般被称为枚举类型：
```Go
type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```

我们也可以在复杂的常量表达式中使用iota，下面是来自net包的例子，用于给一个无符号整数的最低5bit的每个bit指定一个名字：
```Go
type Flags uint

const (
    FlagUp Flags = 1 << iota // is up
    FlagBroadcast            // supports broadcast access capability
    FlagLoopback             // is a loopback interface
    FlagPointToPoint         // belongs to a point-to-point link
    FlagMulticast            // supports multicast access capability
)
```

下面是一个更复杂的例子，每个常量都是1024的幂：
```Go
const (
    _ = 1 << (10 * iota)
    KiB // 1024
    MiB // 1048576
    GiB // 1073741824
    TiB // 1099511627776             (exceeds 1 << 32)
    PiB // 1125899906842624
    EiB // 1152921504606846976
    ZiB // 1180591620717411303424    (exceeds 1 << 64)
    YiB // 1208925819614629174706176
)
```

不过iota常量生成规则也有其局限性。例如，它并不能用于产生1000的幂（KB、MB等），因为Go语言并没有计算幂的运算符。

### 2.5.3 无类型常量
Go语言的常量有个不同寻常之处。虽然一个常量可以有任意一个确定的基础类型，例如int或float64，或者是类似time.Duration这样命名的基础类型，但是许多常量并没有一个明确的基础类型。
编译器为这些没有明确基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度。
这里有六种未明确类型的常量类型，分别是无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串。
对于常量面值，不同的写法可能会对应不同的类型。例如0、0.0、0i和`\u0000`虽然有着相同的常量值，但是它们分别对应无类型的整数、无类型的浮点数、无类型的复数和无类型的字符等不同的常量类型。同样，true和false也是无类型的布尔类型，字符串面值常量是无类型的字符串类型。

通过延迟明确常量的具体类型，无类型的常量不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换。
例如，例子中的ZiB和YiB的值已经超出任何Go语言中整数类型能表达的范围，但是它们依然是合法的常量，而且像下面的常量表达式依然有效（译注：YiB/ZiB是在编译期计算出来的，并且结果常量是1024，是Go语言int变量能有效表示的）：
```Go
fmt.Println(YiB/ZiB) // "1024"
```

对于一个没有显式类型的变量声明（包括简短变量声明），常量的形式将隐式决定变量的默认类型，就像下面的例子：
```Go
i := 0      // untyped integer;        implicit int(0)
r := '\000' // untyped rune;           implicit rune('\000')
f := 0.0    // untyped floating-point; implicit float64(0.0)
c := 0i     // untyped complex;        implicit complex128(0i)
```
如果要给变量一个不同的类型，我们必须显式地将无类型的常量转化为所需的类型，或给声明的变量指定明确的类型，像下面例子这样：
```Go
var i = int8(0)
var i int8 = 0
```

# 3 复合数据类型
## 3.1 数组
因为数组的长度是固定的，因此在Go语言中很少直接使用数组。和数组对应的类型是Slice（切片），它是可以增长和收缩的动态序列，slice功能也更灵活。
数组的每个元素可以通过索引下标来访问，索引下标的范围是从0开始到数组长度减1的位置。内置的len函数将返回数组中元素的个数。
```Go
var a [3]int             // array of 3 integers
fmt.Println(a[0])        // print the first element
fmt.Println(a[len(a)-1]) // print the last element, a[2]

// Print the indices and elements.
for i, v := range a {
    fmt.Printf("%d %d\n", i, v)
}

// Print the elements only.
for _, v := range a {
    fmt.Printf("%d\n", v)
}
```

在数组字面值中，如果在数组的长度位置出现的是“...”省略号，则表示数组的长度是根据初始化值的个数来计算。因此，上面q数组的定义可以简化为:
```Go
q := [...]int{1, 2, 3}
fmt.Printf("%T\n", q) // "[3]int"
```

数组的长度是数组类型的一个组成部分，因此[3]int和[4]int是两种不同的数组类型。数组的长度必须是常量表达式，因为**数组的长度需要在编译阶段确定**。

也可以指定一个索引和对应值列表的方式初始化，就像下面这样：
```Go
type Currency int

const (
    USD Currency = iota // 美元
    EUR                 // 欧元
    GBP                 // 英镑
    RMB                 // 人民币
)

symbol := [...]string{USD: "$", EUR: "€", GBP: "￡", RMB: "￥"}

fmt.Println(RMB, symbol[RMB]) // "3 ￥"
```
在这种形式的数组字面值形式中，初始化索引的顺序是无关紧要的，而且没用到的索引可以省略，和前面提到的规则一样，未指定初始值的元素将用零值初始化。例如:
```Go
r := [...]int{99: -1}
```

如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这时候我们可以直接通过\==比较运算符来比较两个数组，只有当两个数组的所有元素都是相等的时候数组才是相等的。不相等比较运算符!=遵循同样的规则。

当调用一个函数的时候，函数的每个调用参数将会被赋值给函数内部的参数变量，所以函数参数变量接收的是一个复制的副本，并不是原始调用的变量。因为函数参数传递的机制导致传递大的数组类型将是低效的，并且对数组参数的任何的修改都是发生在复制的数组上。
当然，我们可以显式地传入一个数组指针，那样的话函数通过指针对数组的任何修改都可以直接反馈到调用者。下面的函数用于给[32]byte类型的数组清零：
```Go
func zero(ptr *[32]byte) {
    for i := range ptr {
        ptr[i] = 0
    }
}
```

## 3.2 Slice
### 3.2.1 基础
一个slice是一个轻量级的数据结构，提供了访问数组子序列（或者全部）元素的功能，而且slice的底层确实引用一个数组对象。
一个slice由三个部分构成：指针、长度和容量。
- 指针指向第一个slice元素对应的底层数组元素的地址，要注意的是slice的第一个元素并不一定就是数组的第一个元素。
- 长度对应slice中元素的数目；长度不能超过容量。
- 容量一般是从slice的开始位置到底层数据的结尾位置。
内置的len和cap函数分别返回slice的长度和容量。

### 3.2.2 底层数组和切片
多个slice之间可以共享底层的数据，并且引用的数组部分区间可能重叠。作为例子，数组这样定义：
```Go
months := [...]string{1: "January", /* 省略其他月份的展示 */, 12: "December"}
```
slice的切片操作s[i:j]，其中0 ≤ i≤ j≤ cap(s)，用于创建一个新的slice，引用s的从第i个元素开始到第j-1个元素的子序列。
如果i位置的索引被省略的话将使用0代替，如果j位置的索引被省略的话将使用len(s)代替。months[:]切片操作则是引用整个数组。

如果切片操作超出cap(s)的上限将导致一个panic异常，但是超出len(s)则是意味着扩展了slice，因为新slice的长度会变大：
```Go
summer := months[6:9]
fmt.Println(summer[:20]) // panic: out of range

endlessSummer := summer[:5] // extend a slice (within capacity)
fmt.Println(endlessSummer)  // "[June July August September October]"
```

### 3.2.3 不可比较
和数组不同的是，slice之间不能比较，因此我们不能使用\==操作符来判断两个slice是否含有全部相等元素。不过标准库提供了高度优化的bytes.Equal函数来判断两个字节型slice是否相等（[]byte），但是对于其他类型的slice，我们必须自己展开每个元素进行比较：
```Go
func equal(x, y []string) bool {
    if len(x) != len(y) {
        return false
    }
    for i := range x {
        if x[i] != y[i] {
            return false
        }
    }
    return true
}
```
slice唯一合法的比较操作是和nil比较。

### 3.2.4 零值slice
一个零值的slice等于nil，一个nil值的slice的长度和容量都是0，但是也有非nil值的slice的长度和容量也是0的，例如[]int{}或make([]int, 3)[3:]。与任意类型的nil值一样，我们可以用[]int(nil)类型转换表达式来生成一个对应类型slice的nil值：
```Go
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```

如果你需要测试一个slice是否是空的，使用len(s) == 0来判断，而不应该用s == nil来判断。除了和nil相等比较外，一个nil值的slice的行为和其它任意0长度的slice一样；例如reverse(nil)也是安全的。**除了文档已经明确说明的地方，所有的Go语言函数应该以相同的方式对待nil值的slice和0长度的slice**。

### 3.2.5 copy函数
虽然通过循环复制元素更直接，不过内置的copy函数可以方便地将一个slice复制另一个相同类型的slice。
copy函数的第一个参数是要复制的目标slice，第二个参数是源slice，目标和源的位置顺序和`dst = src`赋值语句是一致的。两个slice可以共享同一个底层数组，甚至有重叠也没有问题。
```Go
copy( destSlice, srcSlice []T) int
```
copy函数将返回成功复制的元素的个数（我们这里没有用到），等于两个slice中较小的长度，所以我们不用担心覆盖会超出目标slice的范围。

## 3.3 Map
map类型可以写为map[K]V，其中K和V分别对应key和value。其中K对应的key必须是支持\==比较运算符的数据类型。虽然浮点数类型也是支持相等运算符比较的，但是将浮点数用做key类型则是一个坏的想法，正如第三章提到的，最坏的情况是可能出现的NaN和任何浮点数都不相等。

即使元素不在map中也没有关系；如果一个查找失败将返回value类型对应的零值，例如，即使map中不存在“bob”下面的代码也可以正常工作，因为ages["bob"]失败时将返回0：
```Go
ages["bob"] = ages["bob"] + 1 // happy birthday!
```
而且`x += y`和`x++`等简短赋值语法也可以用在map上。

但是map中的元素并不是一个变量，因此我们不能对map的元素进行取址操作。禁止对map元素取址的原因是map可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。

Map的迭代顺序是不确定的，并且不同的哈希函数实现可能导致不同的遍历顺序。在实践中，遍历的顺序是随机的，每一次遍历的顺序都不相同。这是故意的，每次都使用随机的遍历顺序可以强制要求程序不会依赖具体的哈希函数实现。如果要按顺序遍历key/value对，我们必须显式地对key进行排序。

有时候可能需要知道对应的元素是否真的是在map之中。例如，如果元素类型是一个数字，你可能需要区分一个已经存在的0，和不存在而返回零值的0，可以像下面这样测试：
```Go
if age, ok := ages["bob"]; !ok { /* ... */ }
```

和slice一样，map之间也不能进行相等比较；唯一的例外是和nil进行比较。

## 3.4 结构体
### 3.4.1 基础
下面两个语句声明了一个叫Employee的命名的结构体类型，并且声明了一个Employee类型的变量dilbert：
```Go
type Employee struct {
    ID        int
    Name      string
    Address   string
    DoB       time.Time
    Position  string
    Salary    int
    ManagerID int
}

var dilbert Employee
```

通常一行对应一个结构体成员，成员的名字在前类型在后，不过如果相邻的成员类型如果相同的话可以被合并到一行，就像下面的Name和Address成员那样：
```Go
type Employee struct {
    ID            int
    Name, Address string
    DoB           time.Time
    Position      string
    Salary        int
    ManagerID     int
}
```
结构体成员的输入顺序也有重要的意义。我们也可以将Position成员合并（因为也是字符串类型），或者是交换Name和Address出现的先后顺序，那样的话就是**定义了不同的结构体类型**。

对成员取地址，然后通过指针访问：
```Go
position := &dilbert.Position
*position = "Senior " + *position // promoted, for outsourcing to Elbonia
```

结构体类型的零值是每个成员都是零值。通常会将零值作为最合理的默认值。例如，对于bytes.Buffer类型，结构体初始值就是一个随时可用的空缓存，还有在第9章将会讲到的sync.Mutex的零值也是有效的未锁定状态。有时候这种零值可用的特性是自然获得的，但是也有些类型需要一些额外的工作。

如果结构体没有任何成员的话就是空结构体，写作struct{}。它的大小为0，也不包含任何信息，但是有时候依然是有价值的。有些Go语言程序员用map来模拟set数据结构时，用它来代替map中布尔类型的value，只是强调key的重要性，但是因为节约的空间有限，而且语法比较复杂，所以我们通常会避免这样的用法。
```Go
seen := make(map[string]struct{}) // set of strings
// ...
if _, ok := seen[s]; !ok {
    seen[s] = struct{}{}
    // ...first time seeing s...
}
```

**如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的**，那样的话两个结构体将可以使用\==或!=运算符进行比较。相等比较运算符\==将比较两个结构体的每个成员。

### 3.4.2 结构体嵌入和匿名成员
考虑一个二维的绘图程序，提供了一个各种图形的库，例如矩形、椭圆形、星形和轮形等几何形状。这里是其中两个的定义：
```Go
type Circle struct {
    X, Y, Radius int
}

type Wheel struct {
    X, Y, Radius, Spokes int
}
```

随着库中几何形状数量的增多，我们一定会注意到它们之间的相似和重复之处，所以我们可能为了便于维护而将相同的属性独立出来：
```Go
type Point struct {
    X, Y int
}

type Circle struct {
    Center Point
    Radius int
}

type Wheel struct {
    Circle Circle
    Spokes int
}
```
这样改动之后结构体类型变的清晰了，但是这种修改同时也导致了访问每个成员变得繁琐：
```Go
var w Wheel
w.Circle.Center.X = 8
w.Circle.Center.Y = 8
w.Circle.Radius = 5
w.Spokes = 20
```

Go语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字；这类成员就叫匿名成员。匿名成员的数据类型必须是命名的类型或指向一个命名的类型的指针。下面的代码中，Circle和Wheel各自都有一个匿名成员。我们可以说Point类型被嵌入到了Circle结构体，同时Circle类型被嵌入到了Wheel结构体:
```Go
type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}
```
得益于匿名嵌入的特性，我们可以直接访问叶子属性而不需要给出完整的路径：
```Go
var w Wheel
w.X = 8            // equivalent to w.Circle.Point.X = 8
w.Y = 8            // equivalent to w.Circle.Point.Y = 8
w.Radius = 5       // equivalent to w.Circle.Radius = 5
w.Spokes = 20
```
匿名成员并不是真的无法访问了。其中匿名成员Circle和Point都有自己的名字——就是命名的类型名字——但是这些名字在点操作符中是可选的。我们在访问子成员的时候可以忽略任何匿名成员部分。

不幸的是，结构体字面值并没有简短表示匿名成员的语法，结构体字面值必须遵循形状类型声明时的结构，所以我们只能用下面的两种语法，它们彼此是等价的：
```Go
w = Wheel{Circle{Point{8, 8}, 5}, 20}

w = Wheel{
    Circle: Circle{
        Point:  Point{X: 8, Y: 8},
        Radius: 5,
    },
    Spokes: 20, // NOTE: trailing comma necessary here (and at Radius)
}

fmt.Printf("%#v\n", w)
// Output:
// Wheel{Circle:Circle{Point:Point{X:8, Y:8}, Radius:5}, Spokes:20}

w.X = 42

fmt.Printf("%#v\n", w)
// Output:
// Wheel{Circle:Circle{Point:Point{X:42, Y:8}, Radius:5}, Spokes:20}
```

因为匿名成员也有一个隐式的名字，因此不能同时包含两个类型相同的匿名成员，这会导致名字冲突。同时，因为成员的名字是由其类型隐式地决定的，所以匿名成员也有可见性的规则约束。

# 4 函数
## 4.1 匿名函数
**拥有函数名的函数只能在包级语法块中被声明**，通过**函数字面量（function literal）**，我们可绕过这一限制，在任何表达式中表示一个函数值。函数字面量的语法和函数声明相似，区别在于`func`关键字后没有函数名。函数值字面量是一种表达式，它的值被称为**匿名函数（anonymous function）**。
通过这种方式定义的函数可以访问完整的词法环境（lexical environment），这意味着在函数中定义的内部函数可以引用该函数的变量，如下例所示：
```Go
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}
func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```

squares的例子证明，**函数值不仅仅是一串代码，还记录了状态**。在squares中定义的匿名内部函数可以访问和更新squares中的局部变量，这意味着匿名函数和squares中，存在变量引用。这就是函数值属于引用类型和函数值不可比较的原因。Go使用**闭包(closures)**技术实现函数值，Go程序员也把函数值叫做闭包。

### 4.1.1 警告：捕获迭代变量
考虑这样一个问题：你被要求首先创建一些目录，再将目录删除:
```Go
var rmdirs []func()
for _, d := range tempDirs() {
    dir := d // NOTE: necessary!
    os.MkdirAll(dir, 0755) // creates parent directories too
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir)
    })
}
// ...do some work…
for _, rmdir := range rmdirs {
    rmdir() // clean up
}
```
需要注意，下面的代码是错误的:
```Go
var rmdirs []func()
for _, dir := range tempDirs() {
    os.MkdirAll(dir, 0755)
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir) // NOTE: incorrect!
    })
}
```
问题的原因在于循环变量的作用域。在上面的程序中，for循环语句引入了新的词法块，循环变量dir在这个词法块中被声明。在该循环中生成的所有函数值都共享相同的循环变量。
**需要注意，函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值**。以dir为例，后续的迭代会不断更新dir的值，当删除操作执行时，for循环已完成，dir中存储的值等于最后一次迭代的值。这意味着，每次对os.RemoveAll的调用删除的都是相同的目录。
通常，为了解决这个问题，我们会引入一个与循环变量同名的局部变量，作为循环变量的副本。

## 4.2 defer
### 4.2.1 defer 和返回值
在函数中定义的匿名函数可以访问该函数包括返回值变量在内的所有变量，所以，对匿名函数采用defer机制，可以使其观察函数的返回值。以double函数为例：
```Go
func double(x int) (result int) {
    defer func() { fmt.Printf("double(%d) = %d\n", x,result) }()
    return x + x
}
_ = double(4)
// Output:
// "double(4) = 8"
```

被延迟执行的匿名函数甚至可以修改函数返回给调用者的返回值：
```Go
func triple(x int) (result int) {
    defer func() { result += x }()
    return double(x)
}
fmt.Println(triple(4)) // "12"
```

### 4.2.2 循环体中的 defer
循环体中的defer语句需要特别注意，因为只有在函数执行完毕后，这些被延迟的函数才会执行。
#### 4.2.2.1 资源耗尽问题
下面的代码会导致系统的文件描述符耗尽，因为在所有文件都被处理之前，没有文件会被关闭。
```Go
for _, filename := range filenames {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // NOTE: risky; could run out of file descriptors
    // ...process f…
}
```

一种解决方法是将循环体中的defer语句移至另外一个函数。在每次循环时，调用这个函数。
```Go
for _, filename := range filenames {
    if err := doFile(filename); err != nil {
        return err
    }
}
func doFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    // ...process f…
}
```

#### 4.2.2.2 this指针问题
```Go
package main

import "fmt"

type Test struct {
    name string
}

func (t *Test) Close() {
    fmt.Println(t.name, " closed")
}
func main() {
    ts := []Test{{"a"}, {"b"}, {"c"}}
    for _, t := range ts {
        defer t.Close()
    }
}
```
这个输出并不会像我们预计的输出c b a,而是输出c c c。

那我们换一种方式来调用一下：
```Go
package main

import "fmt"

type Test struct {
    name string
}

func (t *Test) Close() {
    fmt.Println(t.name, " closed")
}
func Close(t Test) {
    t.Close()
}
func main() {
    ts := []Test{{"a"}, {"b"}, {"c"}}
    for _, t := range ts {
        defer Close(t)
    }
}
```
输出结果：
```
c  closed
b  closed
a  closed
```

当然,如果你不想多写一个函数,也很简单,可以像下面这样,同样会输出c b a:
```Go
package main

import "fmt"

type Test struct {
    name string
}

func (t *Test) Close() {
    fmt.Println(t.name, " closed")
}
func main() {
    ts := []Test{{"a"}, {"b"}, {"c"}}
    for _, t := range ts {
        t2 := t
        defer t2.Close()
    }
}
```

### 4.2.3 defer 保证调用
多个 defer 注册，按 FILO 次序执行 ( 先进后出 )。哪怕函数或某个延迟调用发生错误，这些调用依旧会被执行。
```Go
package main

func test(x int) {
    defer println("a")
    defer println("b")

    defer func() {
	    println("c")
        println(100 / x) // div0 异常未被捕获，逐步往外传递，最终终止进程。
    }()

    defer println("d")
}

func main() {
    test(0)
}
```
输出：
```
d
c
b
a
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.test.func1(0x0)
	/box/main.go:8 +0x6f
main.test(0x0)
	/box/main.go:12 +0x140
main.main()
	/box/main.go:15 +0x2a

```

### 4.2.4 defer 与闭包
```Go
package main

import (
    "errors"
    "fmt"
)

func foo(a, b int) (i int, err error) {
    defer fmt.Printf("first defer err %v\n", err)
    defer func(err error) { fmt.Printf("second defer err %v\n", err) }(err)
    defer func() { fmt.Printf("third defer err %v\n", err) }()
    if b == 0 {
        err = errors.New("divided by zero!")
        return
    }

    i = a / b
    return
}

func main() {
    foo(2, 0)
}
```
输出结果：
```Go
third defer err divided by zero!
second defer err <nil>
first defer err <nil>
```
解释：**如果 defer 后面跟的不是一个 closure 最后执行的时候我们得到的并不是最新的值**。

## 4.3 Panic 异常
### 4.3.1 Must 编程规范
考虑`regexp.Compile`函数，该函数将正则表达式编译成有效的可匹配格式。当输入的正则表达式不合法时，该函数会返回一个错误。当调用者明确的知道正确的输入不会引起函数错误时，要求调用者检查这个错误是不必要和累赘的。我们应该假设函数的输入一直合法，就如前面的断言一样：当调用者输入了不应该出现的输入时，触发panic异常。
在程序源码中，大多数正则表达式是字符串字面值（string literals），因此regexp包提供了包装函数`regexp.MustCompile`检查输入的合法性。
```Go
package regexp
func Compile(expr string) (*Regexp, error) { /* ... */ }
func MustCompile(expr string) *Regexp {
    re, err := Compile(expr)
    if err != nil {
        panic(err)
    }
    return re
}

var httpSchemeRE = regexp.MustCompile(`^https?:`) //"http:" or "https:"
```
显然，MustCompile不能接收不合法的输入。函数名中的Must前缀是一种针对此类函数的命名约定，比如template.Must。

### 4.3.2 runtime 包和堆栈信息
为了方便诊断问题，runtime包允许程序员输出堆栈信息。在下面的例子中，我们通过在main函数中延迟调用printStack输出堆栈信息。
```Go
func main() {
    defer printStack()
    f(3)
}

func printStack() {
    var buf [4096]byte
    n := runtime.Stack(buf[:], false)
    os.Stdout.Write(buf[:n])
}

func f(x int) {
    fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
    defer fmt.Printf("defer %d\n", x)
    f(x - 1)
}
```
printStack的简化输出如下（下面只是printStack的输出，不包括panic的日志信息）：
```Go
goroutine 1 [running]:
main.printStack()
src/gopl.io/ch5/defer2/defer.go:20
main.f(0)
src/gopl.io/ch5/defer2/defer.go:27
main.f(1)
src/gopl.io/ch5/defer2/defer.go:29
main.f(2)
src/gopl.io/ch5/defer2/defer.go:29
main.f(3)
src/gopl.io/ch5/defer2/defer.go:29
main.main()
src/gopl.io/ch5/defer2/defer.go:15
```
将panic机制类比其他语言异常机制的读者可能会惊讶，`runtime.Stack`为何能输出已经被释放函数的信息？在Go的panic机制中，**延迟函数的调用在释放堆栈信息之前**。

## 4.4 Recover 捕获异常
### 4.4.1 Recover 陷阱
#### 4.4.1.1 多个错误
延迟调用中引发的错误，可被后续延迟调用捕获，但仅最后一个错误可被捕获。
```Go
package main

import "fmt"

func test() {
    defer func() {
        fmt.Println(recover())
    }()

    defer func() {
        panic("defer panic")
    }()

    panic("test panic")
}

func main() {
    test()
}
```
输出：
```
defer panic
```

#### 4.4.1.2 recover 和延迟调用
捕获函数 recover 只有在延迟调用内直接调用才会终止错误，否则总是返回 nil。任何未捕获的错误都会沿调用堆栈向外传递。
```Go
package main

import "fmt"

func test() {
    defer func() {
        fmt.Println(recover()) //有效
    }()
    defer recover()              //无效！
    defer fmt.Println(recover()) //无效！
    defer func() {
        func() {
            println("defer inner")
            recover() //无效！
        }()
    }()

    panic("test panic")
}

func main() {
    test()
}
```
输出:
```
defer inner
<nil>
test panic
```

# 5 方法
## 5.1 基于指针对象的方法
当这个接受者变量本身比较大时，我们就可以用其指针而不是对象来声明方法，如下：
```Go
package geometry

import "math"

type Point struct{ X, Y float64 }

// same thing, but as a method of the Point type
func (p Point) Distance(q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}

func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```
这个方法的名字是`(*Point).ScaleBy`。这里的括号是必须的；没有括号的话这个表达式可能会被理解为`*(Point.ScaleBy)`。

只有类型（Point）和指向他们的指针`(*Point)`，才可能是出现在接收器声明里的两种接收器。此外，为了避免歧义，在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个例子：
```Go
type P *int
func (P) f() { /* ... */ } // compile error: invalid receiver type
```

就像一些函数允许nil指针作为参数一样，方法理论上也可以用nil指针作为其接收器：
```Go
// An IntList is a linked list of integers.
// A nil *IntList represents the empty list.
type IntList struct {
    Value int
    Tail  *IntList
}
// Sum returns the sum of the list elements.
func (list *IntList) Sum() int {
    if list == nil {
        return 0
    }
    return list.Value + list.Tail.Sum()
}
```
当你定义一个允许nil作为接收器值的方法的类型时，在类型前面的注释中指出nil变量代表的意义是很有必要的.

## 5.2 通过嵌入结构体来扩展类型
来看看ColoredPoint这个类型：
```Go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
    Point
    Color color.RGBA
}
```
我们可以把ColoredPoint类型当作接收器来调用Point里的方法，即使ColoredPoint里没有声明这些方法：
```Go
red := color.RGBA{255, 0, 0, 255}
blue := color.RGBA{0, 0, 255, 255}
var p = ColoredPoint{Point{1, 1}, red}
var q = ColoredPoint{Point{5, 4}, blue}
fmt.Println(p.Distance(q.Point)) // "5"
p.ScaleBy(2)
q.ScaleBy(2)
fmt.Println(p.Distance(q.Point)) // "10"
```
读者如果对基于类来实现面向对象的语言比较熟悉的话，可能会倾向于将Point看作一个基类，而ColoredPoint看作其子类或者继承类。但这是错误的理解。
请注意上面例子中对Distance方法的调用。Distance有一个参数是Point类型，但q并不是一个Point类，所以尽管q有着Point这个内嵌类型，我们也必须要显式地选择它。

内嵌字段会指导编译器去生成额外的包装方法来委托已经声明好的方法，和下面的形式是等价的：
```Go
func (p ColoredPoint) Distance(q Point) float64 {
    return p.Point.Distance(q)
}

func (p *ColoredPoint) ScaleBy(factor float64) {
    p.Point.ScaleBy(factor)
}
```

在类型中内嵌的匿名字段也可能是一个命名类型的指针，这种情况下字段和方法会被间接地引入到当前的类型中。

一个struct类型也可能会有多个匿名字段。我们将ColoredPoint定义为下面这样：
```Go
type ColoredPoint struct {
    Point
    color.RGBA
}
```
如果选择器有二义性的话编译器会报错，比如你在同一级里有两个同名的方法。

方法只能在命名类型（像Point）或者指向类型的指针上定义，但是多亏了内嵌，有些时候我们给匿名struct类型来定义方法也有了手段。
下面是一个小trick。这个例子展示了简单的cache，其使用两个包级别的变量来实现，一个mutex互斥锁（§9.2）和它所操作的cache：
```Go
var (
    mu sync.Mutex // guards mapping
    mapping = make(map[string]string)
)

func Lookup(key string) string {
    mu.Lock()
    v := mapping[key]
    mu.Unlock()
    return v
}
```
下面这个版本在功能上是一致的，但将两个包级别的变量放在了cache这个struct一组内：
```Go
var cache = struct {
    sync.Mutex
    mapping map[string]string
}{
    mapping: make(map[string]string),
}


func Lookup(key string) string {
    cache.Lock()
    v := cache.mapping[key]
    cache.Unlock()
    return v
}
```
我们给新的变量起了一个更具表达性的名字：cache。因为sync.Mutex字段也被嵌入到了这个struct里，其Lock和Unlock方法也就都被引入到了这个匿名结构中了，这让我们能够以一个简单明了的语法来对其进行加锁解锁操作。

## 5.3 方法值和方法表达式
下面例子中的time.AfterFunc这个函数的功能是在指定的延迟时间之后来执行一个（译注：另外的）函数。且这个函数操作的是一个Rocket对象r:
```Go
type Rocket struct { /* ... */ }
func (r *Rocket) Launch() { /* ... */ }
r := new(Rocket)
time.AfterFunc(10 * time.Second, func() { r.Launch() })
```
直接用方法“值”传入AfterFunc的话可以更为简短：
```Go
time.AfterFunc(10 * time.Second, r.Launch)
```

和方法“值”相关的还有方法表达式。当T是一个类型时，方法表达式可能会写作`T.f`或者`(*T).f`，会返回一个函数“值”，这种函数会将其第一个参数用作接收器，所以可以用不写选择器的方式来对其进行调用：
```Go
p := Point{1, 2}
q := Point{4, 6}

distance := Point.Distance   // method expression
fmt.Println(distance(p, q))  // "5"
fmt.Printf("%T\n", distance) // "func(Point, Point) float64"

scale := (*Point).ScaleBy
scale(&p, 2)
fmt.Println(p)            // "{2 4}"
fmt.Printf("%T\n", scale) // "func(*Point, float64)"
```

当你根据一个变量来决定调用同一个类型的哪个函数时，方法表达式就显得很有用了:
```Go
type Point struct{ X, Y float64 }

func (p Point) Add(q Point) Point { return Point{p.X + q.X, p.Y + q.Y} }
func (p Point) Sub(q Point) Point { return Point{p.X - q.X, p.Y - q.Y} }

type Path []Point

func (path Path) TranslateBy(offset Point, add bool) {
    var op func(p, q Point) Point
    if add {
        op = Point.Add
    } else {
        op = Point.Sub
    }
    for i := range path {
        // Call either path[i].Add(offset) or path[i].Sub(offset).
        path[i] = op(path[i], offset)
    }
}
```

## 5.4 方法集
Golang方法集 ：每个类型都有与之关联的方法集，这会影响到接口实现规则。
- 类型 T 方法集包含全部 receiver T 方法。
- 类型 \*T 方法集包含全部 receiver T + \*T 方法。
- 如类型 S 包含匿名字段 T，则 S 和 \*S 方法集包含 T 方法。
- 如类型 S 包含匿名字段 \*T，则 S 和 \*S 方法集包含 T + \*T 方法。
- 不管嵌入 T 或 \*T，\*S 方法集总是包含 T + \*T 方法。

例如：
```Go
type Man struct {}

type Woman struct {}

// Say()方法的全名为(*Man).Say()，即只有指针类型*Man才有Say()方法
func (*Man) Say() {
    fmt.Println("man say")
}

// Say()方法的全名为(Woman).Say()，即只有值类型Woman才有Say()方法
func (Woman) Say() {
    fmt.Println("woman say")
}

// 定义一个说话接口
type CanTalk interface {
	Say()
}
```
如果将 `指针类型的 *man` 或者 `值类型/指针类型的 woman` 的变量赋值给`CanTalk`则没有问题。

# 6 接口
## 6.1 接口类型
io包中定义了很多其它有用的接口类型。Reader可以代表任意可以读取bytes的类型，Closer可以是任意可以关闭的值，例如一个文件或是网络链接。（到现在你可能注意到了很多Go语言中单方法接口的命名习惯）
```Go
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
```

再往下看，我们发现有些新的接口类型通过组合已有的接口来定义。下面是两个例子：
```Go
type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```
这种方式称为接口内嵌。

## 6.2 实现接口的条件
一个类型如果拥有一个接口需要的所有方法，那么这个类型就实现了这个接口。
对于每一个命名过的具体类型T；它的一些方法的接收者是类型T本身然而另一些则是一个`*T`的指针。还记得在T类型的参数上调用一个`*T`的方法是合法的，只要这个参数是一个变量；编译器隐式的获取了它的地址。但这仅仅是一个语法糖：**T类型的值不拥有所有`*T`指针的方法，这样它就可能只实现了更少的接口**。

interface{}被称为空接口类型是不可或缺的。因为空接口类型对实现它的类型没有要求，所以我们可以将任意一个值赋给空接口类型。

因为接口与实现只依赖于判断两个类型的方法，所以没有必要定义一个具体类型和它实现的接口之间的关系。也就是说，有意地在文档里说明或者程序上断言这种关系偶尔是有用的，但程序上不强制这么做。下面的定义在编译期断言一个`*bytes.Buffer`的值实现了io.Writer接口类型:
```Go
// *bytes.Buffer must satisfy io.Writer
var _ io.Writer = (*bytes.Buffer)(nil)
```

## 6.3 接口值
概念上讲一个接口的值，接口值，由两个部分组成，一个具体的类型和那个类型的值。它们被称为接口的动态类型和动态值。
对于像Go语言这种静态类型的语言，类型是编译期的概念；因此一个类型不是一个值。
在我们的概念模型中，一些提供每个类型信息的值被称为类型描述符，比如类型的名称和方法。在一个接口值中，类型部分代表与之相关类型的描述符。

下面4个语句中，变量w得到了3个不同的值。（开始和最后的值是相同的）：
```Go
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = nil
```
让我们进一步观察在每一个语句后的w变量的值和动态行为。第一个语句定义了变量w:
```Go
var w io.Writer
```
在Go语言中，变量总是被一个定义明确的值初始化，即使接口类型也不例外。对于一个接口的零值就是它的类型和值的部分都是nil:
![[Pasted image 20250326232522.png]]
一个接口值基于它的动态类型被描述为空或非空，所以这是一个空的接口值。你可以通过使用w\==nil或者w!=nil来判断接口值是否为空。

第二个语句将一个`*os.File`类型的值赋给变量w:
```Go
w = os.Stdout
```
这个赋值过程调用了一个具体类型到接口类型的隐式转换，这和显式的使用io.Writer(os.Stdout)是等价的。
这个接口值的动态类型被设为`*os.File`指针的类型描述符，它的动态值持有os.Stdout的拷贝；这是一个代表处理标准输出的os.File类型变量的指针:
![[Pasted image 20250326232702.png]]
调用一个包含`*os.File`类型指针的接口值的Write方法，使得`(*os.File).Write`方法被调用:
```Go
w.Write([]byte("hello")) // "hello"
```
通常在编译期，我们不知道接口值的动态类型是什么，所以一个接口上的调用必须使用动态分配。因为不是直接进行调用，所以编译器必须把代码生成在类型描述符的方法Write上，然后间接调用那个地址。这个调用的接收者是一个接口动态值的拷贝，os.Stdout。效果和下面这个直接调用一样:
```Go
os.Stdout.Write([]byte("hello")) // "hello"
```

第三个语句给接口值赋了一个\*bytes.Buffer类型的值:
```Go
w = new(bytes.Buffer)
```
现在动态类型是\*bytes.Buffer并且动态值是一个指向新分配的缓冲区的指针:
![[Pasted image 20250326233005.png]]

最后，第四个语句将nil赋给了接口值：
```
w = nil
```
这个重置将它所有的部分都设为nil值，把变量w恢复到和它之前定义时相同的状态。

**接口值可以使用\==和!＝来进行比较**。两个接口值相等仅当它们都是nil值，或者它们的动态类型相同并且动态值也根据这个动态类型的\==操作相等。因为接口值是可比较的，所以它们可以用在map的键或者作为switch语句的操作数。
然而，如果两个接口值的动态类型相同，但是这个动态类型是不可比较的（比如切片），将它们进行比较就会失败并且panic。

### 6.3.1 警告：一个包含nil指针的接口不是nil接口
思考下面的程序。当debug变量设置为true时，main函数会将f函数的输出收集到一个bytes.Buffer类型中:
```Go
const debug = true

func main() {
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    f(buf) // NOTE: subtly incorrect!
    if debug {
        // ...use buf...
    }
}

// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    // ...do something...
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
```
我们可能会预计当把变量debug设置为false时可以禁止对输出的收集，但是实际上在out.Write方法调用时程序发生了panic。
当main函数调用函数f时，它给f函数的out参数赋了一个\*bytes.Buffer的空指针，所以out的动态值是nil。然而，它的动态类型是\*bytes.Buffer，意思就是out变量是一个包含空指针值的非空接口：
![[Pasted image 20250326233357.png]]

解决方案就是将main函数中的变量buf的类型改为io.Writer，因此可以避免一开始就将一个不完整的值赋值给这个接口：
```Go
var buf io.Writer
if debug {
    buf = new(bytes.Buffer) // enable collection of output
}
f(buf) // OK
```

或者使用更准确的判断nil方式：
```Go
// IsNil 判断接口是否是 nil
func IsNil(item any) (ret bool) {
    defer func() {
        // 如果 item 是结构体or基本类型，则 reflect.ValueOf(item).IsNil() 会抛出异常
        recover()
    }()
    return item == nil || reflect.ValueOf(item).IsNil()
}
```

## 6.4 类型断言
类型断言是一个使用在接口值上的操作。语法上它看起来像x.(T)被称为断言类型，这里x表示一个接口的类型和T表示一个类型。一个类型断言检查它操作对象的动态类型是否和断言的类型匹配。
如果这个检查成功了，类型断言的结果是x的动态值，当然它的类型是T。如果检查失败，接下来这个操作会抛出panic。
如果断言操作的对象是一个nil接口值，那么不论被断言的类型是什么这个类型断言都会失败。

如果类型断言出现在一个预期有两个结果的赋值操作中，例如如下的定义，这个操作不会在失败的时候发生panic，但是替代地返回一个额外的第二个结果，这个结果是一个标识成功与否的布尔值：
```Go
var w io.Writer = os.Stdout
f, ok := w.(*os.File)      // success:  ok, f == os.Stdout
b, ok := w.(*bytes.Buffer) // failure: !ok, b == nil
```

## 6.5 类型分支
一个调用的例子可能看起来像这样：
```Go
func sqlQuote(x interface{}) string {
    if x == nil {
        return "NULL"
    } else if _, ok := x.(int); ok {
        return fmt.Sprintf("%d", x)
    } else if _, ok := x.(uint); ok {
        return fmt.Sprintf("%d", x)
    } else if b, ok := x.(bool); ok {
        if b {
            return "TRUE"
        }
        return "FALSE"
    } else if s, ok := x.(string); ok {
        return sqlQuoteString(s) // (not shown)
    } else {
        panic(fmt.Sprintf("unexpected type %T: %v", x, x))
    }
}
```

一个相似的type switch（类型分支）可以简化类型断言的if-else链:
```Go
switch x.(type) {
case nil:       // ...
case int, uint: // ...
case bool:      // ...
case string:    // ...
default:        // ...
}
```
类型分支语句有一个扩展的形式，它可以将提取的值绑定到一个在每个case范围内都有效的新变量：
```Go
switch x := x.(type) { /* ... */ }
```

使用类型分支的扩展形式来重写sqlQuote函数会让这个函数更加的清晰：
```Go
func sqlQuote(x interface{}) string {
    switch x := x.(type) {
    case nil:
        return "NULL"
    case int, uint:
        return fmt.Sprintf("%d", x) // x has type interface{} here.
    case bool:
        if x {
            return "TRUE"
        }
        return "FALSE"
    case string:
        return sqlQuoteString(x) // (not shown)
    default:
        panic(fmt.Sprintf("unexpected type %T: %v", x, x))
    }
}
```

---
# 7 引用
https://gopl-zh.github.io/index.html