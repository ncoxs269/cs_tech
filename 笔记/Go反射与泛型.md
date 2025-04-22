2025-03-31 22:59
Status: #idea
Tags: [[Go]]


# 1 反射
## 1.1 reflect.Type 和 reflect.Value
### 1.1.1 介绍
一个 `Type` 表示一个Go类型。它是一个接口，有许多方法来区分类型以及检查它们的组成部分，例如一个结构体的成员或一个函数的参数等。
函数 `reflect.TypeOf` 接受任意的 `interface{}` 类型，并以 `reflect.Type` 形式返回其动态类型：
```Go
t := reflect.TypeOf(3)  // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t)          // "int"
```
因为 `reflect.TypeOf` 返回的是一个动态类型的接口值，**它总是返回具体的类型**。因此，下面的代码将打印 "\*os.File" 而不是 "io.Writer"。
```Go
var w io.Writer = os.Stdout
fmt.Println(reflect.TypeOf(w)) // "*os.File"
```


一个 `reflect.Value` 可以装载任意类型的值。函数 `reflect.ValueOf` 接受任意的 `interface{}` 类型，并返回一个装载着其动态值的 `reflect.Value`。和 `reflect.TypeOf` 类似，`reflect.ValueOf` 返回的结果也是具体的类型，但是 `reflect.Value` 也可以持有一个接口值。
```Go
v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v)          // "3"
fmt.Printf("%v\n", v)   // "3"
fmt.Println(v.String()) // NOTE: "<int Value>"
```
和 `reflect.Type` 类似，`reflect.Value` 也满足 `fmt.Stringer` 接口，但是除非 `Value` 持有的是字符串，否则 `String` 方法只返回其类型。而使用 fmt 包的 %v 标志参数会对 `reflect.Values` 特殊处理。

对 `Value` 调用 `Type` 方法将返回具体类型所对应的 `reflect.Type`：
```Go
t := v.Type()           // a reflect.Type
fmt.Println(t.String()) // "int"
```
`reflect.ValueOf` 的逆操作是 `reflect.Value.Interface` 方法。它返回一个 `interface{}` 类型，装载着与 `reflect.Value` 相同的具体值：
```Go
v := reflect.ValueOf(3) // a reflect.Value
x := v.Interface()      // an interface{}
i := x.(int)            // an int
fmt.Printf("%d\n", i)   // "3"
```

### 1.1.2 示例：格式化方法——Value.Kind()
我们使用 `reflect.Value` 的 `Kind` 方法。虽然有无穷多的类型，但是它们的 kinds 类型却是有限的：`Bool`、`String` 和 所有数字类型的基础类型；`Array` 和 `Struct` 对应的聚合类型；`Chan`、`Func`、`Ptr`、`Slice` 和 `Map` 对应的引用类型；`interface` 类型；还有表示空值的 `Invalid` 类型。（空的 `reflect.Value` 的 `kind` 即为 `Invalid`。）
```Go
package format

import (
    "reflect"
    "strconv"
)

// Any formats any value as a string.
func Any(value interface{}) string {
    return formatAtom(reflect.ValueOf(value))
}

// formatAtom formats a value without inspecting its internal structure.
func formatAtom(v reflect.Value) string {
    switch v.Kind() {
    case reflect.Invalid:
        return "invalid"
    case reflect.Int, reflect.Int8, reflect.Int16,
        reflect.Int32, reflect.Int64:
        return strconv.FormatInt(v.Int(), 10)
    case reflect.Uint, reflect.Uint8, reflect.Uint16,
        reflect.Uint32, reflect.Uint64, reflect.Uintptr:
        return strconv.FormatUint(v.Uint(), 10)
    // ...floating-point and complex cases omitted for brevity...
    case reflect.Bool:
        return strconv.FormatBool(v.Bool())
    case reflect.String:
        return strconv.Quote(v.String())
    case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
        return v.Type().String() + " 0x" +
            strconv.FormatUint(uint64(v.Pointer()), 16)
    default: // reflect.Array, reflect.Struct, reflect.Interface
        return v.Type().String() + " value"
    }
}
```

## 1.2 示例：递归的值打印器——Value对复杂类型的取值
```Go
func display(path string, v reflect.Value) {
    switch v.Kind() {
    case reflect.Invalid:
        fmt.Printf("%s = invalid\n", path)
    case reflect.Slice, reflect.Array:
        for i := 0; i < v.Len(); i++ {
            display(fmt.Sprintf("%s[%d]", path, i), v.Index(i))
        }
    case reflect.Struct:
        for i := 0; i < v.NumField(); i++ {
            fieldPath := fmt.Sprintf("%s.%s", path, v.Type().Field(i).Name)
            display(fieldPath, v.Field(i))
        }
    case reflect.Map:
        for _, key := range v.MapKeys() {
            display(fmt.Sprintf("%s[%s]", path,
                formatAtom(key)), v.MapIndex(key))
        }
    case reflect.Ptr:
        if v.IsNil() {
            fmt.Printf("%s = nil\n", path)
        } else {
            display(fmt.Sprintf("(*%s)", path), v.Elem())
        }
    case reflect.Interface:
        if v.IsNil() {
            fmt.Printf("%s = nil\n", path)
        } else {
            fmt.Printf("%s.type = %s\n", path, v.Elem().Type())
            display(path+".value", v.Elem())
        }
    default: // basic types, channels, funcs
        fmt.Printf("%s = %s\n", path, formatAtom(v))
    }
}
```
让我们针对不同类型分别讨论。
**Slice和数组：** 两种的处理逻辑是一样的。`Len`方法返回`slice`或数组值中的元素个数，`Index(i)`获得索引`i`对应的元素，返回的也是一个`reflect.Value`；如果索引i超出范围的话将导致`panic`异常，这与数组或`slice`类型内建的`len(a)`和`a[i]`操作类似。`display`针对序列中的每个元素递归调用自身处理，我们通过在递归处理时向path附加“[i]”来表示访问路径。
**结构体：** `NumField`方法报告结构体中成员的数量，`Field(i)`以`reflect.Value`类型返回第i个成员的值。成员列表也包括通过匿名字段提升上来的成员。为了在path添加“.f”来表示成员路径，我们必须获得结构体对应的`reflect.Type`类型信息，然后访问结构体第i个成员的名字。
**Maps:** `MapKeys`方法返回一个`reflect.Value`类型的`slice`，每一个元素对应`map`的一个`key`。和往常一样，遍历`map`时顺序是随机的。`MapIndex(key)`返回`map`中`key`对应的`value`。我们向path添加“[key]”来表示访问路径。
**指针：** `Elem`方法返回指针指向的变量，依然是`reflect.Value`类型。即使指针是`nil`，这个操作也是安全的，在这种情况下指针是`Invalid`类型，但是我们可以用`IsNil`方法来显式地测试一个空指针，这样我们可以打印更合适的信息。我们在path前面添加“\*”，并用括弧包含以避免歧义。
**接口：** 再一次，我们使用`IsNil`方法来测试接口是否是`nil`，如果不是，我们可以调用`v.Elem()`来获取接口对应的动态值，并且打印对应的类型和值。

## 1.3 示例: 编码为S表达式——Value对基本类型的取值
Go语言的标准库支持了包括JSON、XML和ASN.1等多种编码格式。还有另一种依然被广泛使用的格式是S表达式格式，采用Lisp语言的语法。
在本节中，我们将定义一个包用于将任意的Go语言对象编码为S表达式格式，它支持以下结构：
```
42          integer
"hello"     string（带有Go风格的引号）
foo         symbol（未用引号括起来的名字）
(1 2 3)     list  （括号包起来的0个或多个元素）
```

编码是由一个`encode`递归函数完成，如下所示：
```Go
func encode(buf *bytes.Buffer, v reflect.Value) error {
    switch v.Kind() {
    case reflect.Invalid:
        buf.WriteString("nil")

    case reflect.Int, reflect.Int8, reflect.Int16,
        reflect.Int32, reflect.Int64:
        fmt.Fprintf(buf, "%d", v.Int())

    case reflect.Uint, reflect.Uint8, reflect.Uint16,
        reflect.Uint32, reflect.Uint64, reflect.Uintptr:
        fmt.Fprintf(buf, "%d", v.Uint())

    case reflect.String:
        fmt.Fprintf(buf, "%q", v.String())

    case reflect.Ptr:
        return encode(buf, v.Elem())

    case reflect.Array, reflect.Slice: // (value ...)
        buf.WriteByte('(')
        for i := 0; i < v.Len(); i++ {
            if i > 0 {
                buf.WriteByte(' ')
            }
            if err := encode(buf, v.Index(i)); err != nil {
                return err
            }
        }
        buf.WriteByte(')')

    case reflect.Struct: // ((name value) ...)
        buf.WriteByte('(')
        for i := 0; i < v.NumField(); i++ {
            if i > 0 {
                buf.WriteByte(' ')
            }
            fmt.Fprintf(buf, "(%s ", v.Type().Field(i).Name)
            if err := encode(buf, v.Field(i)); err != nil {
                return err
            }
            buf.WriteByte(')')
        }
        buf.WriteByte(')')

    case reflect.Map: // ((key value) ...)
        buf.WriteByte('(')
        for i, key := range v.MapKeys() {
            if i > 0 {
                buf.WriteByte(' ')
            }
            buf.WriteByte('(')
            if err := encode(buf, key); err != nil {
                return err
            }
            buf.WriteByte(' ')
            if err := encode(buf, v.MapIndex(key)); err != nil {
                return err
            }
            buf.WriteByte(')')
        }
        buf.WriteByte(')')

    default: // float, complex, bool, chan, func, interface
        return fmt.Errorf("unsupported type: %s", v.Type())
    }
    return nil
}
```

## 1.4 通过reflect.Value修改值
### 1.4.1 可取地址
Go语言中类似x、x.f[1]和\*p形式的表达式都可以表示变量，但是其它如x + 1和f(2)则不是变量。一个变量就是一个可寻址的内存空间，里面存储了一个值，并且存储的值可以通过内存地址来更新。
对于`reflect.Values`也有类似的区别。有一些`reflect.Values`是可取地址的；其它一些则不可以。考虑以下的声明语句：
```Go
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)
```
所有通过`reflect.ValueOf(x)`返回的`reflect.Value`都是不可取地址的。但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的。我们可以通过调用`reflect.ValueOf(&x).Elem()`，来获取任意变量x对应的可取地址的`Value`。
我们可以通过调用`reflect.Value`的`CanAddr`方法来判断其是否可以被取地址。

每当我们通过指针间接地获取的`reflect.Value`都是可取地址的，即使开始的是一个不可取地址的`Value`。在反射机制中，所有关于是否支持取地址的规则都是类似的。
例如，`slice`的索引表达式`e[i]`将隐式地包含一个指针，它就是可取地址的，即使开始的e表达式不支持也没有关系。以此类推，`reflect.ValueOf(e).Index(i)`对应的值也是可取地址的，即使原始的`reflect.ValueOf(e)`不支持也没有关系。

### 1.4.2 修改变量
要从变量对应的可取地址的`reflect.Value`来访问变量需要三个步骤。第一步是调用`Addr()`方法，它返回一个`Value`，里面保存了指向变量的指针。然后是在`Value`上调用`Interface()`方法，也就是返回一个`interface{}`，里面包含指向变量的指针。最后，如果我们知道变量的类型，我们可以使用类型的断言机制将得到的`interface{}`类型的接口强制转为普通的类型指针。这样我们就可以通过这个普通指针来更新变量了：
```Go
x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)                    // "3"
```

或者，不使用指针，而是通过调用可取地址的`reflect.Value`的`reflect.Value.Set`方法来更新对应的值：
```Go
d.Set(reflect.ValueOf(4))
fmt.Println(x) // "4"
```
`Set`方法将在运行时执行和编译时进行类似的可赋值性约束的检查。以上代码，变量和值都是`int`类型，但是如果变量是`int64`类型，那么程序将抛出一个`panic`异常。同样，对一个不可取地址的`reflect.Value`调用`Set`方法也会导致`panic`异常。

这里有很多用于基本数据类型的`Set`方法：`SetInt`、`SetUint`、`SetString`和`SetFloat`等。
```Go
d := reflect.ValueOf(&x).Elem()
d.SetInt(3)
fmt.Println(x) // "3"
```
从某种程度上说，这些`Set`方法总是尽可能地完成任务。以`SetInt`为例，只要变量是某种类型的有符号整数就可以工作，即使是一些命名的类型、甚至只要底层数据类型是有符号整数就可以，而且如果对于变量类型值太大的话会被自动截断。
但需要谨慎的是：对于一个引用`interface{}`类型的`reflect.Value`调用`SetInt`会导致`panic`异常，即使那个`interface{}`变量对于整数类型也不行。

### 1.4.3 反射读取未导出成员
当我们用`Display`显示`os.Stdout`结构时，我们发现**反射可以越过Go语言的导出规则的限制读取结构体中未导出的成员**，比如在类Unix系统上`os.File`结构体中的`fd int`成员。然而，**利用反射机制并不能修改这些未导出的成员**：
```Go
stdout := reflect.ValueOf(os.Stdout).Elem() // *os.Stdout, an os.File var
fmt.Println(stdout.Type())                  // "os.File"
fd := stdout.FieldByName("fd")
fmt.Println(fd.Int()) // "1"
fd.SetInt(2)          // panic: unexported field
```
另一个相关的方法`CanSet`是用于检查对应的`reflect.Value`是否是可取地址并可被修改的：
```Go
fmt.Println(fd.CanAddr(), fd.CanSet()) // "true false"
```

## 1.5 示例: 解码S表达式
现在让我们为S表达式编码实现一个简易的`Unmarshal`。
词法分析器`lexer`使用了标准库中的text/scanner包将输入流的字节数据解析为一个个类似注释、标识符、字符串面值和数字面值之类的标记。输入扫描器`scanner`的`Scan`方法将提前扫描和返回下一个记号，对于`rune`类型。大多数记号，比如“(”，对应一个单一`rune`可表示的Unicode字符，但是text/scanner也可以用小的负数表示记号标识符、字符串等由多个字符组成的记号。调用`Scan`方法将返回这些记号的类型，接着调用`TokenText`方法将返回记号对应的文本内容。
因为每个解析器可能需要多次使用当前的记号，但是`Scan`会一直向前扫描，所以我们包装了一个`lexer`扫描器辅助类型，用于跟踪最近由`Scan`方法返回的记号。
```Go
type lexer struct {
    scan  scanner.Scanner
    token rune // the current token
}

func (lex *lexer) next()        { lex.token = lex.scan.Scan() }
func (lex *lexer) text() string { return lex.scan.TokenText() }

func (lex *lexer) consume(want rune) {
    if lex.token != want { // NOTE: Not an example of good error handling.
        panic(fmt.Sprintf("got %q, want %q", lex.text(), want))
    }
    lex.next()
}
```

现在让我们转到语法解析器。它主要包含两个功能。第一个是`read`函数，用于读取S表达式的当前标记，然后根据S表达式的当前标记更新可取地址的`reflect.Value`对应的变量v。
```Go
func read(lex *lexer, v reflect.Value) {
    switch lex.token {
    case scanner.Ident:
        // The only valid identifiers are
        // "nil" and struct field names.
        if lex.text() == "nil" {
            v.Set(reflect.Zero(v.Type()))
            lex.next()
            return
        }
    case scanner.String:
        s, _ := strconv.Unquote(lex.text()) // NOTE: ignoring errors
        v.SetString(s)
        lex.next()
        return
    case scanner.Int:
        i, _ := strconv.Atoi(lex.text()) // NOTE: ignoring errors
        v.SetInt(int64(i))
        lex.next()
        return
    case '(':
        lex.next()
        readList(lex, v)
        lex.next() // consume ')'
        return
    }
    panic(fmt.Sprintf("unexpected token %q", lex.text()))
}
```

我们的S表达式使用标识符区分两个不同类型，结构体成员名和`nil`值的指针。`read`函数值处理`nil`类型的标识符。当遇到`scanner.Ident`为“nil”是，使用`reflect.Zero`函数将变量v设置为零值。而其它任何类型的标识符，我们都作为错误处理。后面的`readList`函数将处理结构体的成员名。
一个“(”标记对应一个列表的开始。第二个函数`readList`，将一个列表解码到一个聚合类型中（`map`、结构体、`slice`或数组），具体类型依然于传入待填充变量的类型。每次遇到这种情况，循环继续解析每个元素直到遇到于开始标记匹配的结束标记“)”，`endList`函数用于检测结束标记。
```Go
func readList(lex *lexer, v reflect.Value) {
    switch v.Kind() {
    case reflect.Array: // (item ...)
        for i := 0; !endList(lex); i++ {
            read(lex, v.Index(i))
        }

    case reflect.Slice: // (item ...)
        for !endList(lex) {
            item := reflect.New(v.Type().Elem()).Elem()
            read(lex, item)
            v.Set(reflect.Append(v, item))
        }

    case reflect.Struct: // ((name value) ...)
        for !endList(lex) {
            lex.consume('(')
            if lex.token != scanner.Ident {
                panic(fmt.Sprintf("got token %q, want field name", lex.text()))
            }
            name := lex.text()
            lex.next()
            read(lex, v.FieldByName(name))
            lex.consume(')')
        }

    case reflect.Map: // ((key value) ...)
        v.Set(reflect.MakeMap(v.Type()))
        for !endList(lex) {
            lex.consume('(')
            key := reflect.New(v.Type().Key()).Elem()
            read(lex, key)
            value := reflect.New(v.Type().Elem()).Elem()
            read(lex, value)
            v.SetMapIndex(key, value)
            lex.consume(')')
        }

    default:
        panic(fmt.Sprintf("cannot decode list into %v", v.Type()))
    }
}

func endList(lex *lexer) bool {
    switch lex.token {
    case scanner.EOF:
        panic("end of file")
    case ')':
        return true
    }
    return false
}
```

最后，我们将解析器包装为导出的`Unmarshal`解码函数，隐藏了一些初始化和清理等边缘处理。内部解析器以`panic`的方式抛出错误，但是`Unmarshal`函数通过在`defer`语句调用`recover`函数来捕获内部`panic`，然后返回一个对`panic`对应的错误信息。
```Go
// Unmarshal parses S-expression data and populates the variable
// whose address is in the non-nil pointer out.
func Unmarshal(data []byte, out interface{}) (err error) {
    lex := &lexer{scan: scanner.Scanner{Mode: scanner.GoTokens}}
    lex.scan.Init(bytes.NewReader(data))
    lex.next() // get the first token
    defer func() {
        // NOTE: this is not an example of ideal error handling.
        if x := recover(); x != nil {
            err = fmt.Errorf("error at %s: %v", lex.scan.Position, x)
        }
    }()
    read(lex, reflect.ValueOf(out).Elem())
    return nil
}
```

## 1.6 获取结构体字段标签——Value修改slice
下面的search函数是一个HTTP请求处理函数。它定义了一个匿名结构体类型的变量，用结构体的每个成员表示HTTP请求的参数。其中结构体成员标签指明了对于请求参数的名字，为了减少URL的长度这些参数名通常都是神秘的缩略词。Unpack将请求参数填充到合适的结构体成员中，这样我们可以方便地通过合适的类型类来访问这些参数。
```Go
import "gopl.io/ch12/params"

// search implements the /search URL endpoint.
func search(resp http.ResponseWriter, req *http.Request) {
    var data struct {
        Labels     []string `http:"l"`
        MaxResults int      `http:"max"`
        Exact      bool     `http:"x"`
    }
    data.MaxResults = 10 // set default
    if err := params.Unpack(req, &data); err != nil {
        http.Error(resp, err.Error(), http.StatusBadRequest) // 400
        return
    }

    // ...rest of handler...
    fmt.Fprintf(resp, "Search: %+v\n", data)
}

// Unpack populates the fields of the struct pointed to by ptr
// from the HTTP request parameters in req.
func Unpack(req *http.Request, ptr interface{}) error {
    if err := req.ParseForm(); err != nil {
        return err
    }

    // Build map of fields keyed by effective name.
    fields := make(map[string]reflect.Value)
    v := reflect.ValueOf(ptr).Elem() // the struct variable
    for i := 0; i < v.NumField(); i++ {
        fieldInfo := v.Type().Field(i) // a reflect.StructField
        tag := fieldInfo.Tag           // a reflect.StructTag
        name := tag.Get("http")
        if name == "" {
            name = strings.ToLower(fieldInfo.Name)
        }
        fields[name] = v.Field(i)
    }

    // Update struct field for each parameter in the request.
    for name, values := range req.Form {
        f := fields[name]
        if !f.IsValid() {
            continue // ignore unrecognized HTTP parameters
        }
        for _, value := range values {
            if f.Kind() == reflect.Slice {
                elem := reflect.New(f.Type().Elem()).Elem()
                if err := populate(elem, value); err != nil {
                    return fmt.Errorf("%s: %v", name, err)
                }
                f.Set(reflect.Append(f, elem))
            } else {
                if err := populate(f, value); err != nil {
                    return fmt.Errorf("%s: %v", name, err)
                }
            }
        }
    }
    return nil
}

func populate(v reflect.Value, value string) error {
    switch v.Kind() {
    case reflect.String:
        v.SetString(value)

    case reflect.Int:
        i, err := strconv.ParseInt(value, 10, 64)
        if err != nil {
            return err
        }
        v.SetInt(i)

    case reflect.Bool:
        b, err := strconv.ParseBool(value)
        if err != nil {
            return err
        }
        v.SetBool(b)

    default:
        return fmt.Errorf("unsupported kind %s", v.Type())
    }
    return nil
}
```

## 1.7 显示一个类型的方法集——Value和Type获取方法
我们的最后一个例子是使用`reflect.Type`来打印任意值的类型和枚举它的方法：
```Go
// Print prints the method set of the value x.
func Print(x interface{}) {
    v := reflect.ValueOf(x)
    t := v.Type()
    fmt.Printf("type %s\n", t)

    for i := 0; i < v.NumMethod(); i++ {
        methType := v.Method(i).Type()
        fmt.Printf("func (%s) %s%s\n", t, t.Method(i).Name,
            strings.TrimPrefix(methType.String(), "func"))
    }
}
```
`reflect.Type`和`reflect.Value`都提供了一个`Method`方法。每次`t.Method(i)`调用返回一个`reflect.Method`的实例，对应一个用于描述一个方法的名称和类型的结构体。每次`v.Method(i)`方法调用都返回一个`reflect.Value`以表示对应的函数值。
使用`reflect.Value.Call`方法（我们这里没有演示），将可以调用一个`Func`类型的`Value`，但是这个例子中只用到了它的类型。

# 2 泛型
## 2.1 一切从函数的形参和实参说起
函数的 **形参(parameter)** 只是类似占位符的东西并没有具体的值，只有我们调用函数传入**实参(argument)** 之后才有具体的值。
```Go
func Add(a int, b int) int {  
    // 变量a,b是函数的形参   "a int, b int" 这一串被称为形参列表
    return a + b
}

Add(100,200) // 调用函数时，传入的100和200是实参
```
如果我们将 **形参 实参** 这个概念推广一下，给变量的类型也引入和类似形参实参的概念的话，在这里我们将其称之为 **类型形参(type parameter)** 和 **类型实参(type argument)**，如下：
```Go
// 假设 T 是类型形参，在定义函数时它的类型是不确定的，类似占位符
func Add(a T, b T) T {  
    return a + b
}
```
通过引入 **类型形参** 和 **类型实参** 这两个概念，我们让一个函数获得了处理多种不同类型数据的能力，这种编程方式被称为 **泛型编程**。

> [!info] 要点
> 如果你经常要分别为不同的类型写完全相同逻辑的代码，那么使用泛型将是最合适的选择

## 2.2 Go的泛型
Go1.18也是通过这种方式实现的泛型，还引入了非常多全新的概念：
- 类型形参 (Type parameter)
- 类型实参(Type argument)
- 类型形参列表( Type parameter list)
- 类型约束(Type constraint)
- 实例化(Instantiations)
- 泛型类型(Generic type)
- 泛型接收器(Generic receiver)
- 泛型函数(Generic function)

## 2.3 泛型类型
下面是一个泛型列表类型：
```Go
type StringSlice []string
type Float32Slie []float32
type Float64Slice []float64

type Slice[T int|float32|float64 ] []T
```
- `T` 就是上面介绍过的**类型形参(Type parameter)**，在定义Slice类型的时候 T 代表的具体类型并不确定，类似一个占位符
- `int|float32|float64` 这部分被称为**类型约束(Type constraint)**，中间的 `|` 的意思是告诉编译器，类型形参 T 只可以接收 int 或 float32 或 float64 这三种类型的实参
- 中括号里的 `T int|float32|float64` 这一整串因为定义了所有的类型形参(在这个例子里只有一个类型形参T），所以我们称其为 **类型形参列表(type parameter list)**
- 这里新定义的类型名称叫 `Slice[T]`，类型定义中带 类型形参 的类型，称之为 **泛型类型(Generic type)**。

泛型类型不能直接拿来使用，必须传入**类型实参(Type argument)** 将其确定为具体的类型之后才可使用。而传入类型实参确定具体类型的操作被称为 **实例化(Instantiations)**。
```Go
// 这里传入了类型实参int，泛型类型Slice[T]被实例化为具体的类型 Slice[int]
var a Slice[int] = []int{1, 2, 3}  
fmt.Printf("Type Name: %T",a)  //输出：Type Name: Slice[int]

// 传入类型实参float32, 将泛型类型Slice[T]实例化为具体的类型 Slice[float32]
var b Slice[float32] = []float32{1.0, 2.0, 3.0} 
fmt.Printf("Type Name: %T",b)  //输出：Type Name: Slice[float32]

// ✗ 错误。因为变量a的类型为Slice[int]，b的类型为Slice[float32]，两者类型不同
a = b  

// ✗ 错误。string不在类型约束 int|float32|float64 中，不能用来实例化泛型类型
var c Slice[string] = []string{"Hello", "World"} 

// ✗ 错误。Slice[T]是泛型类型，不可直接使用必须实例化为具体的类型
var x Slice[T] = []int{1, 2, 3} 
```

### 2.3.1 其他的泛型类型
所有类型定义都可使用类型形参，所以下面这种结构体以及接口的定义也可以使用类型形参：
```Go
// 一个泛型类型的结构体。可用 int 或 sring 类型实例化
type MyStruct[T int | string] struct {  
    Name string
    Data T
}

// 一个泛型接口(关于泛型接口在后半部分会详细讲解）
type IPrintData[T int | float32 | string] interface {
    Print(data T)
}

// 一个泛型通道，可用类型实参 int 或 string 实例化
type MyChan[T int | string] chan T
```

### 2.3.2 类型形参的互相套用
类型形参是可以互相套用的，如下:
```Go
type WowStruct[T int | float32, S []T] struct {
    Data     S
    MaxValue T
    MinValue T
}
```

因为 S 的定义是 []T ，所以 T 一定决定了的话 S 的实参就不能随便乱传了，下面这样的代码是错误的：
```Go
// 错误。S的定义是[]T，这里T传入了实参int, 所以S的实参应当为 []int 而不能是 []float32
ws := WowStruct[int, []float32]{
        Data:     []float32{1.0, 2.0, 3.0},
        MaxValue: 3,
        MinValue: 1,
    }
```

### 2.3.3 泛型类型的套娃
泛型和普通的类型一样，可以互相嵌套定义出更加复杂的新类型，如下：
```Go
// 先定义个泛型类型 Slice[T]
type Slice[T int|string|float32|float64] []T

// ✗ 错误。泛型类型Slice[T]的类型约束中不包含uint, uint8
type UintSlice[T uint|uint8] Slice[T]  

// ✓ 正确。基于泛型类型Slice[T]定义了新的泛型类型 FloatSlice[T] 。FloatSlice[T]只接受float32和float64两种类型
type FloatSlice[T float32|float64] Slice[T] 

// ✓ 正确。基于泛型类型Slice[T]定义的新泛型类型 IntAndStringSlice[T]
type IntAndStringSlice[T int|string] Slice[T]  
// ✓ 正确 基于IntAndStringSlice[T]套娃定义出的新泛型类型
type IntSlice[T int] IntAndStringSlice[T] 

// 在map中套一个泛型类型Slice[T]
type WowMap[T int|string] map[string]Slice[T]
// 在map中套Slice[T]的另一种写法
type WowMap2[T Slice[int] | Slice[string]] map[string]T
```

### 2.3.4 几种语法错误
1. 定义泛型类型的时候，**基础类型不能只有类型形参**，如下：
```Go
// 错误，类型形参不能单独使用
type CommonType[T int|string|float32] T
```
2. 当类型约束的一些写法会被编译器误认为是表达式时会报错。如下：
```Go
//✗ 错误。T *int会被编译器误认为是表达式 T乘以int，而不是int指针
type NewType[T *int] []T
// 上面代码再编译器眼中：它认为你要定义一个存放切片的数组，数组长度由 T 乘以 int 计算得到
type NewType [T * int][]T 

//✗ 错误。和上面一样，这里不光*被会认为是乘号，| 还会被认为是按位或操作
type NewType2[T *int|*float64] []T 

//✗ 错误
type NewType2 [T (int)] []T 
```
为了避免这种误解，解决办法就是给类型约束包上 `interface{}` 或加上逗号消除歧义（关于接口具体的用法会在后半篇提及）:
```Go
type NewType[T interface{*int}] []T
type NewType2[T interface{*int|*float64}] []T 

// 如果类型约束中只有一个类型，可以添加个逗号消除歧义
type NewType3[T *int,] []T

//✗ 错误。如果类型约束不止一个类型，加逗号是不行的
type NewType4[T *int|*float32,] []T 
```
因为上面逗号的用法限制比较大，这里推荐统一用 interface{} 解决问题。
3. 匿名结构体不支持泛型，下面的用法是错误的：
```Go
testCase := struct[T int|string] {
        caseName string
        got      T
        want     T
    }[int]{
        caseName: "test OK",
        got:      100,
        want:     100,
    }
```

## 2.4 泛型receiver
定义了新的普通类型之后可以给类型添加方法。那么可以给泛型类型添加方法吗？答案自然是可以的，如下：
```Go
type MySlice[T int | float32] []T

func (s MySlice[T]) Sum() T {
    var sum T
    for _, value := range s {
        sum += value
    }
    return sum
}
```

泛型类型无论如何都需要先用类型实参实例化，所以用法如下：
```Go
var s MySlice[int] = []int{1, 2, 3, 4}
fmt.Println(s.Sum()) // 输出：10

var s2 MySlice[float32] = []float32{1.0, 2.0, 3.0, 4.0}
fmt.Println(s2.Sum()) // 输出：10.0
```
有了泛型之后，我们就能非常简单地创建通用数据结构了。

### 2.4.1 基于泛型的队列
队列是一种先入先出的数据结构，它和现实中排队一样，数据只能从队尾放入、从队首取出，先放入的数据优先被取出来:
```Go
// 这里类型约束使用了空接口，代表的意思是所有类型都可以用来实例化泛型类型 Queue[T] (关于接口在后半部分会详细介绍）
type Queue[T interface{}] struct {
    elements []T
}

// 将数据放入队列尾部
func (q *Queue[T]) Put(value T) {
    q.elements = append(q.elements, value)
}

// 从队列头部取出并从头部删除对应数据
func (q *Queue[T]) Pop() (T, bool) {
    var value T
    if len(q.elements) == 0 {
        return value, true
    }

    value = q.elements[0]
    q.elements = q.elements[1:]
    return value, len(q.elements) == 0
}

// 队列大小
func (q Queue[T]) Size() int {
    return len(q.elements)
}
```

### 2.4.2 动态判断变量的类型
对于 `valut T` 这样通过类型形参定义的变量，我们能不能判断具体类型然后对不同类型做出不同处理呢？答案是不允许的，如下：
```Go
func (q *Queue[T]) Put(value T) {
    value.(int) // 错误。泛型类型定义的变量不能使用类型断言

    // 错误。不允许使用type switch 来判断 value 的具体类型
    switch value.(type) {
    case int:
        // do something
    case string:
        // do something
    default:
        // do something
    }
    
    // ...
}
```
虽然type switch和类型断言不能用，但我们可通过反射机制达到目的：
```Go
func (receiver Queue[T]) Put(value T) {
    // Printf() 可输出变量value的类型(底层就是通过反射实现的)
    fmt.Printf("%T", value) 

    // 通过反射可以动态获得变量value的类型从而分情况处理
    v := reflect.ValueOf(value)

    switch v.Kind() {
    case reflect.Int:
        // do something
    case reflect.String:
        // do something
    }

    // ...
}
```

这看起来达到了我们的目的，可是当你写出上面这样的代码时候就出现了一个问题：
> 你为了避免使用反射而选择了泛型，结果到头来又为了一些功能在在泛型中使用反射

## 2.5 泛型函数
写泛型函数也十分简单。假设我们想要写一个计算两个数之和的函数：
```Go
func Add[T int | float32 | float64](a T, b T) T {
    return a + b
}
```
你会觉得这样每次都要手动指定类型实参太不方便了。所以Go还支持类型实参的自动推导：
```Go
Add(1, 2)  // 1，2是int类型，编译请自动推导出类型实参T是int
Add(1.0, 2.0) // 1.0, 2.0 是浮点，编译请自动推导出类型实参T是float32
```

### 2.5.1 匿名函数不支持泛型
```Go
// 错误，匿名函数不能自己定义类型实参
fnGeneric := func[T int | float32](a, b T) T {
        return a + b
} 

fmt.Println(fnGeneric(1, 2))
```
但是匿名函数可以使用别处定义好的类型实参，如：
```Go
func MyFunc[T int | float32 | float64](a, b T) {

    // 匿名函数可使用已经定义好的类型形参
    fn2 := func(i T, j T) T {
        return i*2 - j*2
    }

    fn2(a, b)
}
```

### 2.5.2 方法不支持泛型
目前Go的方法并不支持泛型，如下：
```Go
type A struct {
}

// 不支持泛型方法
func (receiver A) Add[T int | float32 | float64](a T, b T) T {
    return a + b
}
```
目前唯一的办法就是曲线救国，迂回地通过receiver使用类型形参。

## 2.6 变得复杂的接口
Go支持将类型约束单独拿出来定义到接口中，从而让代码更容易维护：
```Go
type IntUintFloat interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64
}

type Slice[T IntUintFloat] []T
```

不过这样的代码依旧不好维护，而接口和接口、接口和普通类型之间也是可以通过 `|` 进行组合：
```Go
type Int interface {
    int | int8 | int16 | int32 | int64
}

type Uint interface {
    uint | uint8 | uint16 | uint32
}

type Float interface {
    float32 | float64
}

type Slice[T Int | Uint | Float] []T  // 使用 '|' 将多个接口类型组合
```

同时，在接口里也能直接组合其他接口，所以还可以像下面这样：
```Go
type SliceElement interface {
    Int | Uint | Float | string // 组合了三个接口类型并额外增加了一个 string 类型
}

type Slice[T SliceElement] []T 
```

### 2.6.1 指定底层类型
上面定义的 Slie[T] 虽然可以达到目的，但是有一个缺点：
```Go
var s1 Slice[int] // 正确 

type MyInt int
var s2 Slice[MyInt] // ✗ 错误。MyInt类型底层类型是int但并不是int类型，不符合 Slice[T] 的类型约束
```

为了从根本上解决这个问题，Go新增了一个符号 `~` ，在类型约束中使用类似 `~int` 这种写法的话，就代表着不光是 int ，所有以 int 为底层类型的类型也都可用于实例化。
```Go
type Int interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32
}
type Float interface {
    ~float32 | ~float64
}

type Slice[T Int | Uint | Float] []T 

var s Slice[int] // 正确

type MyInt int
var s2 Slice[MyInt]  // MyInt底层类型是int，所以可以用于实例化

type MyMyInt MyInt
var s3 Slice[MyMyInt]  // 正确。MyMyInt 虽然基于 MyInt ，但底层类型也是int，所以也能用于实例化

type MyFloat32 float32  // 正确
var s4 Slice[MyFloat32]
```

使用 `~` 时有一定的限制：
1. ~后面的类型不能为接口
2. ~后面的类型必须为基本类型（包括结构体），或它们的slice

### 2.6.2 从方法集(Method set)到类型集(Type set)
如果你比较敏锐的话，一定会隐约认识到这种写法的改变这也一定意味着Go语言中 `接口(interface)` 这个概念发生了非常大的变化。
是的，在Go1.18之前，Go官方对 `接口(interface)` 的定义是：接口是一个方法集(method set)。

就如下面这个代码一样， `ReadWriter` 接口定义了一个接口(方法集)，这个集合中包含了 `Read()` 和 `Write()` 这两个方法。所有同时定义了这两种方法的类型被视为实现了这一接口。
```Go
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```
但是，我们如果换一个角度来重新思考上面这个接口的话，会发现接口的定义实际上还能这样理解：
我们可以把 `ReaderWriter` 接口看成代表了一个 **类型的集合**，所有实现了 `Read()` `Writer()` 这两个方法的类型都在接口代表的类型集合当中。
通过换个角度看待接口，在我们眼中接口的定义就从 **`方法集(method set)`** 变为了 **`类型集(type set)`**。而Go1.18开始就是依据这一点将接口的定义正式更改为了 **类型集(Type set)**。

#### 2.6.2.1 接口实现(implement)定义的变化
既然接口定义发生了变化，那么从Go1.18开始 `接口实现(implement)` 的定义自然也发生了变化。当满足以下条件时，我们可以说 **类型 T 实现了接口 I ( type T implements interface I)**：
- T 不是接口时：类型 T 是接口 I 代表的类型集中的一个成员 (T is an element of the type set of I)
- T 是接口时： T 接口代表的类型集是 I 代表的类型集的子集(Type set of T is a subset of the type set of I)

#### 2.6.2.2 类型的并集
并集我们已经很熟悉了，之前一直使用的 `|` 符号就是求类型的并集( `union` )

#### 2.6.2.3 类型的交集
接口可以不止书写一行，如果一个接口有多行类型定义，那么取它们之间的 **交集**
```Go
type AllInt interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint32
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

type A interface { // 接口A代表的类型集是 AllInt 和 Uint 的交集
    AllInt
    Uint
}

type B interface { // 接口B代表的类型集是 AllInt 和 ~int 的交集
    AllInt
    ~int
}
```
下面这个例子中
- 接口 A 代表的是 AllInt 与 Uint 的 **交集**，即 `~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64`
- 接口 B 代表的则是 AllInt 和 ~int 的**交集**，即 `~int`

#### 2.6.2.4 空集
当多个类型的交集如下面 `Bad` 这样为空的时候， `Bad` 这个接口代表的类型集为一个**空集**：
```Go
type Bad interface {
    int
    float32 
} // 类型 int 和 float32 没有相交的类型，所以接口 Bad 代表的类型集为空
```

#### 2.6.2.5 空接口和 any
上面说了空集，接下来说一个特殊的类型集——`空接口 interface{}` 。因为，Go1.18开始接口的定义发生了改变，所以 `interface{}` 的定义也发生了一些变更：
> 空接口代表了所有类型的集合

所以，对于Go1.18之后的空接口应该这样理解：
1. 虽然空接口内没有写入任何的类型，但它代表的是所有类型的集合，而非一个 **空集**
2. 类型约束中指定 **空接口** 的意思是指定了一个包含所有类型的类型集，并不是类型约束限定了只能使用 **空接口** 来做类型形参
```Go
// 空接口代表所有类型的集合。写入类型约束意味着所有类型都可拿来做类型实参
type Slice[T interface{}] []T

var s1 Slice[int]    // 正确
var s2 Slice[map[string]string]  // 正确
var s3 Slice[chan int]  // 正确
var s4 Slice[interface{}]  // 正确
```

Go1.18开始提供了一个和空接口 `interface{}` 等价的新关键词 `any` ，用来使代码更简单：
```Go
type Slice[T any] []T // 代码等价于 type Slice[T interface{}] []T
```
实际上 `any` 的定义就位于Go语言的 `builtin.go` 文件中（参考如下）， `any` 实际上就是 `interaface{}` 的别名(alias)，两者完全等价：
```Go
// any is an alias for interface{} and is equivalent to interface{} in all ways.
type any = interface{} 
```

#### 2.6.2.6 comparable(可比较) 和 可排序(ordered)
Go直接内置了一个叫 `comparable` 的接口，它代表了所有可用 `!=` 以及 `==` 对比的类型：
```Go
type MyMap[KEY comparable, VALUE any] map[KEY]VALUE // 正确
```

而可进行大小比较的类型被称为 `Orderd` 。目前Go语言并没有像 `comparable` 这样直接内置对应的关键词，所以想要的话需要自己来定义相关接口，比如我们可以参考Go官方包`golang.org/x/exp/constraints` 如何定义：
```Go
// Ordered 代表所有可比大小排序的类型
type Ordered interface {
    Integer | Float | ~string
}

type Integer interface {
    Signed | Unsigned
}

type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Unsigned interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
    ~float32 | ~float64
}
```

> [!warning] 警告
> 这里虽然可以直接使用官方包 [golang.org/x/exp/constraints](https://link.segmentfault.com/?enc=E96wBhT0mGkrfr42ug%2Buyg%3D%3D.eFrqdutbbBPSGjzfeY5pioBE8z7ZnwmtdWWRS6DvSqPQIXrkEZRAgSpxNDKEY4%2FQ) ，但因为这个包属于实验性质的 x 包，今后可能会发生非常大变动，所以并不推荐直接使用

### 2.6.3 接口两种类型
我们接下来再观察一个例子，这个例子是阐述接口是类型集最好的例子：
```Go
type ReadWriter interface {
    ~string | ~[]rune

    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```
接口类型 ReadWriter 代表了一个类型集合，所有以 string 或 []rune 为底层类型，并且实现了 Read() Write() 这两个方法的类型都在 ReadWriter 代表的类型集当中。

如下面代码中，StringReadWriter 存在于接口 ReadWriter 代表的类型集中，而 BytesReadWriter 因为底层类型是 []byte（既不是string也是不[]rune） ，所以它不属于 ReadWriter 代表的类型集：
```Go
// 类型 StringReadWriter 实现了接口 Readwriter
type StringReadWriter string 

func (s StringReadWriter) Read(p []byte) (n int, err error) {
    // ...
}

func (s StringReadWriter) Write(p []byte) (n int, err error) {
 // ...
}

//  类型BytesReadWriter 没有实现接口 Readwriter
type BytesReadWriter []byte 

func (s BytesReadWriter) Read(p []byte) (n int, err error) {
 ...
}

func (s BytesReadWriter) Write(p []byte) (n int, err error) {
 ...
}
```

你一定会说，啊等等，这接口也变得太复杂了把，那我定义一个 `ReadWriter` 类型的接口变量，然后接口变量赋值的时候不光要考虑到方法的实现，还必须考虑到具体底层类型？心智负担也太大了吧。是的，为了解决这个问题也为了保持Go语言的兼容性，Go1.18开始将接口分为了两种类型
- **基本接口(Basic interface)**
- **一般接口(General interface)**

#### 2.6.3.1 基本接口(Basic interface)
接口定义中如果只有方法的话，那么这种接口被称为**基本接口(Basic interface)**。这种接口就是Go1.18之前的接口，用法也基本和Go1.18之前保持一致。

#### 2.6.3.2 一般接口(General interface)
如果接口内有类型的话，这种接口被称为 **一般接口(General interface)** 。**一般接口类型不能用来定义变量，只能用于泛型的类型约束中**。所以以下的用法是错误的：
```Go
type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

var uintInf Uint // 错误。Uint是一般接口，只能用于类型约束，不得用于变量定义
```

这一限制保证了一般接口的使用被限定在了泛型之中，不会影响到Go1.18之前的代码，同时也极大减少了书写代码时的心智负担。

### 2.6.4 泛型接口
所有类型的定义中都可以使用类型形参，所以接口定义自然也可以使用类型形参，观察下面的例子：
```Go
type DataProcessor2[T any] interface {
    int | ~struct{ Data interface{} }

    Process(data T) (newData T)
    Save(data T) error
}

DataProcessor2[string]

// 实例化后的接口定义可视为
type DataProcessor2[T string] interface {
    int | ~struct{ Data interface{} }

    Process(data string) (newData string)
    Save(data string) error
}
```
`DataProcessor2[string]` 因为带有类型并集所以它是 **一般接口(General interface)**，所以实例化之后的这个接口代表的意思是：
1. 只有实现了 `Process(string) string` 和 `Save(string) error` 这两个方法，并且以 `int` 或 `struct{ Data interface{} }` 为底层类型的类型才算实现了这个接口
2. **一般接口(General interface)** 不能用于变量定义只能用于类型约束，所以接口 `DataProcessor2[string]` 只是定义了一个用于类型约束的类型集

```Go
// XMLProcessor 虽然实现了接口 DataProcessor2[string] 的两个方法，但是因为它的底层类型是 []byte，所以依旧是未实现 DataProcessor2[string]
type XMLProcessor []byte

func (c XMLProcessor) Process(oriData string) (newData string) {

}

func (c XMLProcessor) Save(oriData string) error {

}

// JsonProcessor 实现了接口 DataProcessor2[string] 的两个方法，同时底层类型是 struct{ Data interface{} }。所以实现了接口 DataProcessor2[string]
type JsonProcessor struct {
    Data interface{}
}

func (c JsonProcessor) Process(oriData string) (newData string) {

}

func (c JsonProcessor) Save(oriData string) error {

}

// 错误。DataProcessor2[string]是一般接口不能用于创建变量
var processor DataProcessor2[string]

// 正确，实例化之后的 DataProcessor2[string] 可用于泛型的类型约束
type ProcessorList[T DataProcessor2[string]] []T

// 正确，接口可以并入其他接口
type StringProcessor interface {
    DataProcessor2[string]

    PrintString()
}

// 错误，带方法的一般接口不能作为类型并集的成员(参考6.5 接口定义的种种限制规则
type StringProcessor interface {
    DataProcessor2[string] | DataProcessor2[[]byte]

    PrintString()
}
```

### 2.6.5 接口定义的种种限制规则
剩下还有一些规则因为找不到好的地方介绍，所以在这里统一介绍下：
1. 用 `|` 连接多个类型的时候，类型之间不能有相交的部分(即必须是不交集):
```Go
type MyInt int

// 错误，MyInt的底层类型是int,和 ~int 有相交的部分
type _ interface {
    ~int | MyInt
}
```
但是相交的类型中是接口的话，则不受这一限制：
```Go
type MyInt int

type _ interface {
    ~int | interface{ MyInt }  // 正确
}

type _ interface {
    interface{ ~int } | MyInt // 也正确
}

type _ interface {
    interface{ ~int } | interface{ MyInt }  // 也正确
}
```

2. 类型的并集中不能有类型形参
```Go
type MyInf[T ~int | ~string] interface {
    ~float32 | T  // 错误。T是类型形参
}

type MyInf2[T ~int | ~string] interface {
    T  // 错误
}
```

3. 接口不能直接或间接地并入自己
```Go
type Bad interface {
    Bad // 错误，接口不能直接并入自己
}

type Bad2 interface {
    Bad1
}
type Bad1 interface {
    Bad2 // 错误，接口Bad1通过Bad2间接并入了自己
}

type Bad3 interface {
    ~int | ~string | Bad3 // 错误，通过类型的并集并入了自己
}
```

4. 接口的并集成员个数大于一的时候不能直接或间接并入 `comparable` 接口
```Go
type OK interface {
    comparable // 正确。只有一个类型的时候可以使用 comparable
}

type Bad1 interface {
    []int | comparable // 错误，类型并集不能直接并入 comparable 接口
}

type CmpInf interface {
    comparable
}
type Bad2 interface {
    chan int | CmpInf  // 错误，类型并集通过 CmpInf 间接并入了comparable
}
type Bad3 interface {
    chan int | interface{comparable}  // 理所当然，这样也是不行的
}
```

5. 带方法的接口(无论是基本接口还是一般接口)，都不能写入接口的并集中：
```Go
type _ interface {
    ~int | ~string | error // 错误，error是带方法的接口(一般接口) 不能写入并集中
}

type DataProcessor[T any] interface {
    ~string | ~[]byte

    Process(data T) (newData T)
    Save(data T) error
}

// 错误，实例化之后的 DataProcessor[string] 是带方法的一般接口，不能写入类型并集
type _ interface {
    ~int | ~string | DataProcessor[string] 
}

type Bad[T any] interface {
    ~int | ~string | DataProcessor[T]  // 也不行
}

```

---
# 3 引用
https://segmentfault.com/a/1190000041634906