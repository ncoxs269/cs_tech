2025-04-06 16:19
Status: #idea
Tags: [[Go]]


# 1 数据结构
## 1.1 string
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

### 1.1.1 string和[]byte转换
`Go`语言中提供了标准方式对`string`和`[]byte`进行转换:
```text
func main()  {
 str := "asong"
 by := []byte(str)

 str1 := string(by)
 fmt.Println(str1)
}
```

#### 1.1.1.1 `string`类型转换到`[]byte`类型
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

#### 1.1.1.2 `[]byte`类型转换到`string`类型
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

### 1.1.2 string和[]byte强转换
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

### 1.1.3 两种转换如何取舍
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

## 1.2 slice
### 1.2.1 切片的数据结构
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

### 1.2.2 创建切片
创建切片有两种形式，make 创建切片，空切片。
#### 1.2.2.1 make 和切片字面量
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

#### 1.2.2.2 nil 和空切片
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

### 1.2.3 切片扩容
#### 1.2.3.1 1.18之前的扩容策略
**在Go的1.18版本以前，是按照如下的规则来进行扩容：**
- 首先判断，如果新申请容量（cap，新切片期望最小的容量）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
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

#### 1.2.3.2 1.18及之后的扩容策略
新版本的扩容策略：
1. 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
2. 否则判断，当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍。
3. 否则判断，原slice容量超过256，新slice容量循环 `newcap = oldcap + (oldcap+3*256) / 4`，直到>=cap。
	1. 在1.25~2倍之间寻找一种相对平衡的规则。使用 `768 = 3 × 256` 作为经验值，使扩容过程更平滑。
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

## 1.3 map——1.24之前
### 1.3.1 基本思想
Go对map的设计采用了桶的思想，有一组组桶来装KV对，并且规定一个桶只能装**8个**KV对。
假如我们有很多个KV对，只要桶够多，把它们分散在各个桶中，**那么就能将O(N)的时间复杂度缩小到O(N/bucketsNum)了**，只要给定合适的桶数，时间复杂度就≈O(1)。

#### 1.3.1.1 如何找到桶？
采取的措施是使用哈希映射来解决。我们对hash函数的选取需要有一定的要求，它必须满足以下的性质：
- hash的**可重入性**：相同的key，必定产生相同的hash值
- hash的**离散性**：只要两个key不相同，不论相似度的高低，产生的hash值都会在整个输出域内均匀地离散化
- hash的**单向性**：不可通过hash值反向寻找key

输入是无限的，但是输出的长度却是固定有限的，所以必然会存在两个不同的key通过映射到了同一个hash值的情况上，这种情况称之为**hash冲突**。对于Go对hash冲突采取的策略，将会在下文提及。

#### 1.3.1.2 如何保证桶平均的KV对数目是合理的？
在Go中，引入了**负载因子（Load Factor）**的概念，对于一个map，假如存在`count`个键值对，`2^B`个桶，那么它必须满足以下方程：**「count <=LoadFactor*(2^B)」**，当count的值超过这个界限，就会引发map的扩容机制。`LoadFactor`在Go中，一般取值为6.5。

#### 1.3.1.3 桶结构
一个map会维护一个桶数组，每个桶可以存放八个键值对以及它的hash值，以及一个指向其溢出桶的指针。
Go采用拉链法解决哈希冲突。桶数组中的每一个桶，严格来说应该是一个链表结构，它通过`overflow`指针链接了下一个键值对。若当前桶有8个链表节点了，就开辟一个新的溢出桶，放置在溢出桶里面。

### 1.3.2 数据结构定义
结构定义如下：
```Go
type hmap struct {
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed
 
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
 
	extra *mapextra // optional fields
}
```
- `count`：标识当前map的键值对数量
- `flags`：标识当前map的状态
- `B`：2^B为map目前的总桶数
- `noverflow`：溢出桶数量
- `hash0`：哈希因子
- `buckets`：指向桶数组
- `oldbuckets`：扩容时存储旧的桶数组
- `nevacuate`：待完成数据迁移的桶下标
- `extra`：存储预分配的溢出桶

`mapextra`的定义如下：
```Go
type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap
 
	nextOverflow *bmap
}
```
- `overflow`：指向新的预分配溢出桶数组
- `oldoverflow`：指向旧的预分配溢出桶数组
- `nextoverflow`：指向下一个可被使用的溢出桶

而bmap是一个桶的具体实现，源码如下：
```Go
type bmap struct {
	tophash [bucketCnt]uint8
}
```
虽然在数据定义上，只含有一个`tophash`，但是在内存上，可以通过直接计算得出下一个槽的位置，以及overflow指针的位置。所以为了便于理解，它的实际结构如下：
```Go
type bmap struct {
	tophash [bucketCnt]uint8
	keys [bucketCnt]T
    values [bucketCnt]T
    overflow unsafe.Pointer
}
```

### 1.3.3 主干流程
#### 1.3.3.1 map的创建与初始化
代码如下：
```Go
//makemap为make(map[k]v,hint)实现Go映射创建
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 1. makemap首先根据预分配的容量大小hint进行分配容量，若容量过大，则会置hint为0
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		hint = 0
	}
 
	// 2. 通过new方法，初始化hmap
	if h == nil {
		h = new(hmap)
	}
	// 3. 通过`rand()`生成一个哈希因子
	h.hash0 = uint32(rand())
 
	// 4. 获取哈希表的桶数量的对数B。（注意这里并不是直接计算log_2_hint，是要根据负载因子衡量桶的数量）
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
	
    // 5. 若B==0，那么buckets将会采取懒创建的策略，会在未来要写map的方法mapassign中创建。
    // 若B!=0，初始化哈希表，使用`makeBucketArray`方法构造桶数组。
    // 如果map的容量过大，会提前申请一批溢出桶。
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

/*
返回的两个值为：
- 计算二值的乘积`a*b`
- 乘积是否溢出

如果a|b的二进制表示，没有超过1<<(4*goarch.PtrSize)，那么它们的乘积也不会溢出。在64位操作系统中，goarch.PtrSize的大小为8。否则，则通过a*b>MaxUintptr来判断，MaxUintptr为2^64-1.
*/
func MulUintptr(a, b uintptr) (uintptr, bool) {
	if a|b < 1<<(4*goarch.PtrSize) || a == 0 {
		return a * b, false
	}
	overflow := b > MaxUintptr/a
	return a * b, overflow
}

/*
在这里，`bucketCnt`为8。若count<=8，则直接返回false，只需要将键值对放在一个桶中即可。否则，计算当前的**哈希表的容量*负载因子**，若count的数量＞这个值，将会扩容哈希表，即增大B。
*/
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

##### 1.3.3.1.1 makeBucketArray方法
代码如下：
```Go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    //初始桶数量
	base := bucketShift(b)
    //最终桶数量，初始和base相同
	nbuckets := base
	//溢出桶预分配
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
        //计算分配的总内存大小
		sz := t.Bucket.Size_ * nbuckets
        //将内存大小向上对齐到合适的大小，是内存分配的一个优化。
		up := roundupsize(sz, t.Bucket.PtrBytes == 0)
		if up != sz {
            //调整桶数量，使得内存被充分利用
			nbuckets = up / t.Bucket.Size_
		}
	}
 
	if dirtyalloc == nil {
        //分配nbuckets个桶
		buckets = newarray(t.Bucket, int(nbuckets))
	} else {
		//复用旧的内存
		buckets = dirtyalloc
		size := t.Bucket.Size_ * nbuckets
		if t.Bucket.PtrBytes != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
 
	if base != nbuckets {
		//如果base和nbuckets的数量不同，说明预分配了溢出桶，需要设置溢出桶链表
        //指向第一个可用的预分配溢出桶，计算出溢出桶的起始位置
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
        //最后一个预分配的溢出桶的位置
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
        //将最后一个溢出桶的指针设置为buckets，形成一个环形链表，用于后面的分配判断
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
`makeBucketArray`方法会根据初始的对数B来判断是否需要分配溢出桶。若B>=4，则需要预分配的溢出桶数量为2^(B-4)。确定好桶的总数后，会根据`dirtyalloc`是否为nil来判断是否需要新开辟空间。最后会返回指向桶数组的指针以及指向首个溢出桶位置的指针。
当最后返回到上层的`makemap`方法中，最终创造出的`map`结构如图：
![[Pasted image 20250406175604.png]]

---
# 2 引用
slice原理：https://halfrost.com/go_slice/#toc-0
slice新扩容策略：https://cloud.tencent.com/developer/article/2309846
map底层原理（1.24之前）：https://www.cnblogs.com/MelonTe/p/18753711