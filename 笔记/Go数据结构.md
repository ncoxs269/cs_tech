2025-04-06 16:19
Status: #idea
Tags: [[Go底层原理]]


# 1 string
`string`类型本质也是一个结构体，定义如下：
```text
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

我们看看他在实例化时调用的方法：
```text
//go:nosplit
func gostringnocopy(str *byte) string {
 ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
 s := *(*string)(unsafe.Pointer(&ss))
 return s
}

s := "abc"
```
入参是一个`byte`类型的指针，从这我们可以看出`string`类型底层是一个`byte`类型的数组。

## 1.1 string和[]byte转换
`Go`语言中提供了标准方式对`string`和`[]byte`进行转换:
```text
func main()  {
 str := "asong"
 by := []byte(str)

 str1 := string(by)
 fmt.Println(str1)
}
```

### 1.1.1 `string`类型转换到`[]byte`类型
我们对上面的代码执行如下指令`go tool compile -N -l -S ./string_to_byte/string.go`，可以看到调用的是`runtime.stringtoslicebyte`：
```text
// runtime/string.go go 1.15.7
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
 var b []byte
 if buf != nil && len(s) <= len(buf) {
  *buf = tmpBuf{}
  b = buf[:len(s)]
 } else {
  b = rawbyteslice(len(s))
 }
 copy(b, s)
 return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
 cap := roundupsize(uintptr(size))
 p := mallocgc(cap, nil, false)
 if cap != uintptr(size) {
  memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
 }

 *(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
 return
}
```
这里分了两种状况，通过字符串长度来决定是否需要重新分配一块内存。也就是说预先定义了一个长度为`32`的数组，字符串的长度超过了这个数组的长度，就说明`[]byte`不够用了，需要重新分配一块内存了。这也算是一种优化吧，`32`是阈值，只有超过`32`才会进行内存分配。
最后我们会通过调用`copy`方法实现string到[]byte的拷贝，具体实现在`src/runtime/slice.go`中的`slicestringcopy`方法，这里就不贴这段代码了，这段代码的核心思路就是：**将string的底层数组从头部复制n个到[]byte对应的底层数组中去**。

### 1.1.2 `[]byte`类型转换到`string`类型
`[]byte`类型转换到`string`类型本质调用的就是`runtime.slicebytetostring`：
```text
// 以下无关的代码片段
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
 if n == 0 {
  return ""
 }
 if n == 1 {
  p := unsafe.Pointer(&staticuint64s[*ptr])
  if sys.BigEndian {
   p = add(p, 7)
  }
  stringStructOf(&str).str = p
  stringStructOf(&str).len = 1
  return
 }

 var p unsafe.Pointer
 if buf != nil && n <= len(buf) {
  p = unsafe.Pointer(buf)
 } else {
  p = mallocgc(uintptr(n), nil, false)
 }
 stringStructOf(&str).str = p
 stringStructOf(&str).len = n
 memmove(p, unsafe.Pointer(ptr), uintptr(n))
 return
}
```
这段代码我们可以看出会根据`[]byte`的长度来决定是否重新分配内存，最后通过`memove`可以拷贝数组到字符串。

## 1.2 string和[]byte强转换
标准的转换方法都会发生内存拷贝，所以为了减少内存拷贝和内存申请我们可以使用**强转换**的方式对两者进行转换。在标准库中有对这两种方法实现：
```text
// runtime/string.go
func slicebytetostringtmp(ptr *byte, n int) (str string) {
 stringStructOf(&str).str = unsafe.Pointer(ptr)
 stringStructOf(&str).len = n
 return
}

func stringtoslicebytetmp(s string) []byte {
    str := (*stringStruct)(unsafe.Pointer(&s))
    ret := slice{array: unsafe.Pointer(str.str), len: str.len, cap: str.len}
    return *(*[]byte)(unsafe.Pointer(&ret))
}
```
主要使用的就是`unsafe.Pointer`进行指针替换，为什么这样可以呢？因为`string`和`slice`的结构字段是相似的：
```text
type stringStruct struct {
    str unsafe.Pointer
    len int
}
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
唯一不同的就是`cap`字段，`array`和`str`是一致的，`len`是一致的，所以他们的内存布局上是对齐的，这样我们就可以直接通过`unsafe.Pointer`进行指针替换。

## 1.3 两种转换如何取舍
当然是推荐大家使用标准转换方式了，毕竟标准转换方式是更安全的！但是如果你是在高性能场景下使用，是可以考虑使用强转换的方式的，但是要注意强转换的使用方式，他不是安全的，这里举个例子：
```text
func stringtoslicebytetmp(s string) []byte {
 str := (*reflect.StringHeader)(unsafe.Pointer(&s))
 ret := reflect.SliceHeader{Data: str.Data, Len: str.Len, Cap: str.Len}
 return *(*[]byte)(unsafe.Pointer(&ret))
}

func main()  {
 str := "hello"
 by := stringtoslicebytetmp(str)
 by[0] = 'H'
}
```

运行结果：
```text
unexpected fault address 0x109d65f
fatal error: fault
[signal SIGBUS: bus error code=0x2 addr=0x109d65f pc=0x107eabc]
```
程序直接发生严重错误了，即使使用`defer`+`recover`也无法捕获。`string`类型是不能改变的，也就是底层数据是不能更改的，这里因为我们使用的是强转换的方式，那么`by`指向了`str`的底层数组，现在对这个数组中的元素进行更改，就会出现这个问题，导致整个程序`down`掉！

# 2 slice
## 2.1 切片的数据结构
切片本身并不是动态数组或者数组指针。它内部实现的数据结构通过指针引用底层数组，设定相关属性将数据读写操作限定在指定的区域内。**切片本身是一个只读对象，其工作机制类似数组指针的一种封装**。
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

如果想从 slice 中得到一块内存地址，可以这样做：
```go
s := make([]byte, 200)
ptr := unsafe.Pointer(&s[0])
```
如果反过来呢？从 Go 的内存地址中构造一个 slice：
```go
var ptr unsafe.Pointer
var s1 = struct {
    addr uintptr
    len int
    cap int
}{ptr, length, length}
s := *(*[]byte)(unsafe.Pointer(&s1))
```

当然还有更加直接的方法，在 Go 的反射中就存在一个与之对应的数据结构 SliceHeader，我们可以用它来构造一个 slice:
```go
var o []byte
sliceHeader := (*reflect.SliceHeader)((unsafe.Pointer(&o)))
sliceHeader.Cap = length
sliceHeader.Len = length
sliceHeader.Data = uintptr(ptr)
```

## 2.2 创建切片
创建切片有两种形式，make 创建切片，空切片。
### 2.2.1 make 和切片字面量
```go
func makeslice(et *_type, len, cap int) slice {
	// 根据切片的数据类型，获取切片的最大容量
	maxElements := maxSliceCap(et.size)
    // 比较切片的长度，长度值域应该在[0,maxElements]之间
	if len < 0 || uintptr(len) > maxElements {
		panic(errorString("makeslice: len out of range"))
	}
    // 比较切片的容量，容量值域应该在[len,maxElements]之间
	if cap < len || uintptr(cap) > maxElements {
		panic(errorString("makeslice: cap out of range"))
	}
    // 根据切片的容量申请内存
	p := mallocgc(et.size*uintptr(cap), et, true)
    // 返回申请好内存的切片的首地址
	return slice{p, len, cap}
}
```
还有一个 int64 的版本：
```go
func makeslice64(et *_type, len64, cap64 int64) slice {
	len := int(len64)
	if int64(len) != len64 {
		panic(errorString("makeslice: len out of range"))
	}

	cap := int(cap64)
	if int64(cap) != cap64 {
		panic(errorString("makeslice: cap out of range"))
	}

	return makeslice(et, len, cap)
}
```

### 2.2.2 nil 和空切片
nil 切片和空切片也是常用的。
```go
var slice []int
```
![[Pasted image 20250406162937.png]]

空切片一般会用来表示一个空的集合。比如数据库查询，一条结果也没有查到，那么就可以返回一个空切片。
```go
silce := make( []int , 0 )
slice := []int{ }
```
![[Pasted image 20250406163018.png]]

## 2.3 切片扩容
### 2.3.1 1.18之前的扩容策略
**在Go的1.18版本以前，是按照如下的规则来进行扩容：**
- 首先判断，如果新申请容量（cap，新插入元素导致所需的容量）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
- 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap）
- 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的 1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
- 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）

扩容代码实现：
```go
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc(unsafe.Pointer(&et))
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if et.size == 0 {
		// 如果新要扩容的容量比原来的容量还要小，这代表要缩容了，那么可以直接报panic了。
		if cap < old.cap {
			panic(errorString("growslice: cap out of range"))
		}

		// 如果当前切片的大小为0，还调用了扩容方法，那么就新生成一个新的容量的切片返回。
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

    // 这里就是扩容的策略
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	// 计算新的切片的容量，长度。
	var lenmem, newlenmem, capmem uintptr
	const ptrSize = unsafe.Sizeof((*byte)(nil))
	switch et.size {
	case 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		newcap = int(capmem)
	case ptrSize:
		lenmem = uintptr(old.len) * ptrSize
		newlenmem = uintptr(cap) * ptrSize
		capmem = roundupsize(uintptr(newcap) * ptrSize)
		newcap = int(capmem / ptrSize)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem = roundupsize(uintptr(newcap) * et.size)
		newcap = int(capmem / et.size)
	}

	// 判断非法的值，保证容量是在增加，并且容量不超过最大容量
	if cap < old.cap || uintptr(newcap) > maxSliceCap(et.size) {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.kind&kindNoPointers != 0 {
		// 在老的切片后面继续扩充容量
		p = mallocgc(capmem, nil, false)
		// 将 lenmem 这个多个 bytes 从 old.array地址 拷贝到 p 的地址处
		memmove(p, old.array, lenmem)
		// 先将 P 地址加上新的容量得到新切片容量的地址，然后将新切片容量地址后面的 capmem-newlenmem 个 bytes 这块内存初始化。为之后继续 append() 操作腾出空间。
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// 重新申请新的数组给新切片
		// 重新申请 capmen 这个大的内存地址，并且初始化为0值
		p = mallocgc(capmem, et, true)
		if !writeBarrier.enabled {
			// 如果还不能打开写锁，那么只能把 lenmem 大小的 bytes 字节从 old.array 拷贝到 p 的地址处
			memmove(p, old.array, lenmem)
		} else {
			// 循环拷贝老的切片的值
			for i := uintptr(0); i < lenmem; i += et.size {
				typedmemmove(et, add(p, i), add(old.array, i))
			}
		}
	}
	// 返回最终新切片，容量更新为最新扩容之后的容量
	return slice{p, old.len, newcap}
}
```

### 2.3.2 1.18及之后的扩容策略
新版本的扩容策略：
1. 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
2. 否则判断，当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍。
3. 否则判断，原slice容量超过256，新slice容量循环 `newcap = oldcap + (oldcap+3*256) / 4`，直到>=cap。
	1. **通过减小阈值并固定增加一个常数，使得优化后的扩容的系数在阈值前后不再会出现从2到1.25的突变**
4. 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）

```Go
// type切片期望的类型，old旧切片，cap新切片期望最小的容量
func growslice(et *_type, old slice, cap int) slice {
  newcap := old.cap// 老切片容量
  doublecap := newcap + newcap// 老切片容量的两倍
  if cap > doublecap {// 期望最小的容量 > 老切片的两倍(新切片的容量 = 2 * 老切片的容量)
    newcap = cap
  } else {
    const threshold = 256
    if old.cap < threshold {
      newcap = doublecap
    } else {
      for 0 < newcap && newcap < cap {
        // 在2倍增长以及1.25倍之间寻找一种相对平衡的规则
        newcap += (newcap + 3*threshold) / 4
      }
      if newcap <= 0 {
        newcap = cap
      }
    }
  }
}
```

## 2.4 总结
slice底层是一个结构体，包含三个字段：底层数组的指针引用、元素个数和容量。
slice的扩容策略：
1. 如果新申请容量(cap)大于2倍的旧容量(oldcap)，那么最终容量就是新申请容量
2. 否则当旧容量小于256的时候，最终容量为原来的2倍。
3. 否则当旧容量大于256，新容量(newcap)循环 `newcap = oldcap + (oldcap+3*256) / 4`，直到`>=cap`。
	1. 这里通过引入常数 3\*256，让扩容系数从2倍（256时）平滑过渡到1.25倍
4. 如果最终容量计算值溢出，则最终容量就是新申请容量

# 3 map——1.24之前
## 3.1 简介
Go map 底层实现方式是 Hash 表。Go map 的数据被置入一个由桶组成的数组中，每个桶最多可以存放 8 个键值对（更好的局部性）。key的哈希值低位用于在数组中定位到桶，而高8位则用于在桶中区分键值对。
每个桶最多存 8 个键值对，超了则会链接到额外的溢出桶。所以 Go map 基本数据结构是**hash数组+桶内的键值对数组+溢出的桶链表**。
哈希表容量超过阈值需要扩容时，会分配一个新的桶数组，新数组的大小一般是旧数组的 2 倍。迁移数据时不会全量拷贝，因为耗时太大，Go 会在每次读写 Map 时以桶为单位做渐进式搬迁。

### 3.1.1 核心结构
map 主要有两个核心结构，基础结构和桶结构：
- hmap：map 的基础结构。 
- bmap：存放 key-value 的桶结构。

hmap定义：
```go
type hmap struct {
	count     int // 元素的个数
	flags     uint8 // 状态标记位。如是否被多线程读写、迭代器在使用新桶、迭代器在使用旧桶等
	B         uint8 // 桶指数，表示 hash 数组中桶数量为 2^B（不包括溢出桶）。最大可存储元素数量为 loadFactor * 2^B
	noverflow uint16 // 溢出桶的数量的近似值。详见函数 incrnoverflow()
	hash0     uint32 // hash种子

	buckets    unsafe.Pointer // 指向2^B个桶组成的数组的指针。可能是 nil 如果 count 为 0
	oldbuckets unsafe.Pointer // 指向长度为新桶数组一半的旧桶数组，仅在增长时为非零
	nevacuate  uintptr // 进度计数器，表示扩容后搬迁的进度（小于该数值的桶已迁移）

	extra *mapextra // 溢出桶
}

type mapextra struct {
	overflow    *[]*bmap // 保存溢出桶链表
	oldoverflow *[]*bmap // 保存旧溢出桶链表

	nextOverflow *bmap // 下一个空闲溢出桶地址
}
```

bmap 定义：
```go
type bmap struct {
	tophash [bucketCnt]uint8 // 存储桶内 8 个 key 的 hash 值的高字节。tophash[0] < minTopHash 表示桶处于扩容迁移状态
}
```
特别注意：实际分配内存时会申请一个更大的内存空间 A，A 的前 8 字节为 bmap，后面依次跟 8 个key、8 个 value、1 个溢出指针。把所有的 key 排在一起和所有的 value 排列在一起，而不是交替排列，这样的好处是在某些情况下可以省略掉内存对齐占用的空间。map 的桶结构实际指的是内存空间 A。

另外，map.go 里很多函数的第 1 个入参是下面这个结构，从成员来看很明显，此结构标示了键值对和桶的类型和大小等必要信息。有了这个结构的信息，map.go 的代码就可以与键值对的具体数据类型解耦。所以 map.go 用内存偏移量和 unsafe.Pointer 指针来直接对内存进行存取，而无需关心 key 或 value 的具体类型：
```go
type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
}
```
### 3.1.2 数据结构图
创建 map 时，会初始化一个 hmap 结构体，同时分配一个足够大的内存空间 A。
其中 A 的前段用于 hash 数组，A 的后段预留给溢出的桶。于是 `hmap.buckets` 指向 hash 数组，即 A 的首地址；`hmap.extra.nextOverflow` 初始时指向内存 A 中的后段，即 hash 数组结尾的下一个桶，也即第 1 个预留的溢出桶。
所以当 hash 冲突需要使用到新的溢出桶时，会优先使用上述预留的溢出桶。`hmap.extra.nextOverflow` 依次往后偏移直到用完所有的溢出桶，才有可能会申请新的溢出桶空间。
![[image-151.png]]
当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但是，我们看 bmap 其实有一个 overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想。这时会把 overflow 移动到 extra 字段来，避免了 gc 扫描。

## 3.2 实现机制
### 3.2.1 增加或修改
步骤：
1. 参数合法性检测与 hash 值计算
2. 定位 key 在 hash 表中的位置
	1. 用 key 的 hash 值的低位定位 hash 数组的下标偏移量，用 hash 值的高 8 位用于在桶内定位键值对
	2. 怎么定位桶位置？首先用 B 算出掩码，例如B=3、掩码111；然后用掩码&hash获得低位；桶位置偏移量=低位值\*桶大小
3. 进一步定位 key 可以插入的桶及桶中的位置
	1. 两轮循环，外层循环遍历 hash 桶及其指向的溢出链表，内层循环则在桶内遍历。如果在链表上的桶内找到了 key，则直接更新 key；否则进入第 4 步插入新 key。
	2. 为什么不直接用key比较？因为hash高位对比快，能快速对比出不一样的情况；hash一样，则再用key对比
4. 插入新 key
	1. 插入新 key 首先需要判断 map 是否需要扩容
	2. 如果不需要扩容且找到了 key 的插入位置，则直接插入；否则说明链表上的桶都满了，这时链接一个新的溢出桶进来。
	3. 注意：当 key 或 value 的大小超过一定值时，桶只存储 key 或 value 的指针。

### 3.2.2 删除
由于定位 key 位置的方式是查找 tophash，所以删除操作对 tophash 的处理是关键：
1. map 首先将对应位置的 `tophash[i]` 置为 `emptyOne=1`，表示该位置已被删除；
2. 如果 `tophash[i]` 不是整个链表的最后一个有效位置（即最后一个元素），则只置 `emptyOne` 标志。该位置被删除但未释放，后续插入操作不能使用此位置
3. 如果 `tophash[i]` 是链表最后一个有效位置了，则把链表最后面的所有标志为 `emptyOne` 的位置，都置为 `emptyRest=0`。置为 `emptyRest` 的位置可以在后续的插入操作中被使用。

事实上，Go 数据一旦被插入到桶的确切位置，map 是不会再移动该数据在桶中的位置了。这种删除方式，以少量空间避免了被删除的数据再次插入时出现数据移动的情况。

### 3.2.3 扩容
触发扩容有两个条件：
1. 当前不处在 growing 状态
2. 元素过多or溢出桶多
	1. 元素个数大于hash桶数量(2^B)\*6.5。注意这里的 hash 桶指的是 hash 数组中的桶，不包括溢出的桶；
	2. 或者，溢出桶数量noverflow>=32768(1<<15) || noverflow>=hash桶数量

Go map 有两种扩容类型：
1. 一种是真扩容，扩到 hash 桶数量为原来的两倍，针对元素数量过多的情况；
2. 一种是假扩容，hash 桶数量不变，只是把元素搬迁到新的 map，针对溢出桶过多的情况。此时元素没那么多，很多桶没装满，但是因为不断插入、删除，导致emptyOne过多、使得溢出桶过多。假扩容会消除emptyOne，减少溢出桶
	1. 但是假扩容解决不了key 哈希都一样、落到同一个 bucket 里的情况

搬迁的工作是增量搬迁的，在每次插入和删除操作时执行一次搬迁工作：
1. 每执行一次插入或删除，都会调用 growWork() 函数搬迁 0~2 个 hash 桶
2. 搬迁是以 hash 桶为单位的，包含对应的 hash 桶和这个桶的溢出链表；
3. 被 delete 掉的元素（emptyone 标志）会被舍弃不进行搬迁。

minTopHash：当一个 cell 的 tophash 值小于 minTopHash 时，标志这个 cell 的迁移状态。因为这个状态值是放在 tophash 数组里，为了和正常的哈希值区分开，会给 key 计算出来的哈希值加一个增量：minTopHash。这样就能区分正常的 top hash 值和表示状态的哈希值。

---
# 4 引用
slice原理：https://halfrost.com/go_slice/#toc-0
slice新扩容策略：https://cloud.tencent.com/developer/article/2309846
map实现原理：https://cloud.tencent.com/developer/article/1746966
map底层原理（1.24之前）：https://www.cnblogs.com/MelonTe/p/18753711