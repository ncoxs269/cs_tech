2025-05-11 16:54
Status: #idea
Tags: [[Go]] [[并发]]


# 1 context
## 1.1 简介
`context`包是在`go1.7`版本中引入到标准库中的。
`context`的作用就是在不同的`goroutine`之间同步数据、取消信号以及处理请求的截止日期。
`context`包定义了上下文类型，可以使用`background`、`TODO`创建一个上下文，在函数调用链之间传播`context`，也可以使用`WithDeadline`、`WithTimeout`、`WithCancel` 或 `WithValue` 创建的修改副本替换它。

## 1.2 使用方法
### 1.2.1 创建
`context`包主要提供了两种方式创建`context`:
- `context.Backgroud()`
- `context.TODO()`

这两个函数其实只是互为别名，没有差别，官方给的定义是：
- `context.Background` 是上下文的默认值，所有其他的上下文都应该从它衍生（Derived）出来。
- `context.TODO` 应该只在不确定应该使用哪种上下文时使用。

上面的两种方式是创建根`context`，不具备任何功能，具体实践还是要依靠`context`包提供的`With`系列函数来进行派生：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```
这四个函数都要基于父`Context`衍生，通过这些函数，就创建了一颗Context树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个，画个图表示一下：
![[image-129.png]]

### 1.2.2 `WithValue`携带数据
我们日常在业务开发中都希望能有一个`trace_id`能串联所有的日志。
在`python`中我们可以用`gevent.local`来传递，在`java`中我们可以用`ThreadLocal`来传递，在`Go`语言中我们就可以使用`Context`来传递，通过使用`WithValue`来创建一个携带`trace_id`的`context`，然后不断透传下去，打印日志时输出即可。

在使用`withVaule`时要注意以下事项：
- **不建议使用`context`值传递关键参数**，关键参数应该显示的声明出来，不应该隐式处理，`context`中最好是携带签名、`trace_id`这类值。
- 因为携带`value`也是`key`、`value`的形式，为了避免`context`因多个包同时使用`context`而带来冲突，`key`建议采用内置类型。
- `context`传递的数据中`key`、`value`都是`interface`类型，这种类型编译期无法确定类型，所以不是很安全，所以在类型断言时别忘了保证程序的健壮性。

在获取键值对是，我们先从当前`context`中查找，没有找到会在从父`context`中查找该键对应的值直到在某个父`context`中返回 `nil` 或者查找到对应的值。

### 1.2.3 超时控制
`withTimeout`和`withDeadline`作用是一样的，就是传递的时间参数不同而已，他们都会通过传入的时间来自动取消`Context`。
这里要注意的是他们都会返回一个`cancelFunc`方法，通过调用这个方法可以达到提前进行取消。不过在使用的过程还是建议在自动取消后也调用`cancelFunc`去停止定时减少不必要的资源浪费。

现在我们就举个例子来试用一下超时控制，现在我们就模拟一个请求写两个例子：
1. 达到超时时间终止接下来的执行：
```Go
func main()  {
    HttpHandler()
}

func NewContextWithTimeout() (context.Context,context.CancelFunc) {
    return context.WithTimeout(context.Background(), 3 * time.Second)
}

func HttpHandler()  {
    ctx, cancel := NewContextWithTimeout()
    defer cancel()
    deal(ctx)
}

func deal(ctx context.Context)  {
    for i:=0; i< 10; i++ {
        time.Sleep(1*time.Second)
        select {
        case <- ctx.Done():
            fmt.Println(ctx.Err())
            return
        default:
            fmt.Printf("deal time is %d\n", i)
        }
    }
}

/* 输出：
deal time is 0
deal time is 1
context deadline exceeded
*/
```
2. 没有达到超时时间终止接下来的执行：
```go
func main()  {
    HttpHandler1()
}

func NewContextWithTimeout1() (context.Context,context.CancelFunc) {
    return context.WithTimeout(context.Background(), 3 * time.Second)
}

func HttpHandler1()  {
    ctx, cancel := NewContextWithTimeout1()
    defer cancel()
    deal1(ctx, cancel)
}

func deal1(ctx context.Context, cancel context.CancelFunc)  {
    for i:=0; i< 10; i++ {
        time.Sleep(1*time.Second)
        select {
        case <- ctx.Done():
            fmt.Println(ctx.Err())
            return
        default:
            fmt.Printf("deal time is %d\n", i)
            cancel()
        }
    }
}

/* 输出：
deal time is 0
deal time is 1
context deadline exceeded
*/
```

这里大家要记的一个坑，如果我们想单独开一个goroutine去处理其他的事情并且不会随着请求结束后而被取消的话，那么传递的`context`要基于`context.Background`或者`context.TODO`重新衍生一个传递，否决就会和预期不符合了。

### 1.2.4 `withCancel`取消
我们往往为了完成一个复杂的需求会开多个`gouroutine`去做一些事情，我们可以使用`withCancel`来衍生一个`context`传递到不同的`goroutine`中，当我想让这些`goroutine`停止运行，就可以调用`cancel`来进行取消。
例如：
```go
func main()  {
    ctx,cancel := context.WithCancel(context.Background())
    go Speak(ctx)
    time.Sleep(10*time.Second)
    cancel()
    time.Sleep(1*time.Second)
}

func Speak(ctx context.Context)  {
    for range time.Tick(time.Second){
        select {
        case <- ctx.Done():
            fmt.Println("我要闭嘴了")
            return
        default:
            fmt.Println("balabalabalabala")
        }
    }
}

/* 输出：
balabalabalabala
....省略
balabalabalabala
我要闭嘴了
*/
```

## 1.3 实现原理
### 1.3.1 接口
Context其实就是一个接口，定义了四个方法：
```go
type Context interface {
 Deadline() (deadline time.Time, ok bool)
 Done() <-chan struct{}
 Err() error
 Value(key interface{}) interface{}
}
```
- `Deadlne`方法：返回绑定当前`context`的任务被取消的截止时间；如果没有设定期限，将返回`ok == false`。
- `Done`方法：当绑定当前`context`的任务被取消时，将返回一个关闭的`channel`；如果当前`context`不会被取消，将返回`nil`。
- `Err`方法：如果`Done`返回的`channel`没有关闭，将返回`nil`;如果`Done`返回的`channel`已经关闭，将返回非空的值表示任务结束的原因。如果是`context`被取消，`Err`将返回`Canceled`；如果是`context`超时，`Err`将返回`DeadlineExceeded`。
- `Value`方法：获取设置的`key`对应的值

这个接口主要被三个类继承实现，分别是`emptyCtx`、`ValueCtx`、`cancelCtx`，采用匿名接口的写法，这样可以对任意实现了该接口的类型进行重写。

### 1.3.2 创建根`Context`
其在我们调用`context.Background`、`context.TODO`时创建的对象就是`empty`：
```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

#### 1.3.2.1 `emptyCtx`类
`emptyCtx`主要是给我们创建根`Context`时使用的，其实现方法也是一个空结构，实际源代码长这样：
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}

func (e *emptyCtx) String() string {
    switch e {
    case background:
        return "context.Background"
    case todo:
        return "context.TODO"
    }
    return "unknown empty Context"
}
```

### 1.3.3 `WithValue`的实现
`withValue`内部主要就是调用`valueCtx`类：
```Go
func WithValue(parent Context, key, val interface{}) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

`valueCtx`目的就是为`Context`携带键值对，因为它采用匿名接口的继承实现方式，他会继承父`Context`，也就相当于嵌入`Context`当中了
```Go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) String() string {
    return contextName(c.Context) + ".WithValue(type " +
        reflectlite.TypeOf(c.key).String() +
        ", val " + stringify(c.val) + ")"
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

### 1.3.4 `WithCancel`的实现
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    // 调用`propagateCancel`构建父子`context`之间的关联关系，这样当父`context`被取消时，子`context`也会被取消
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```

#### 1.3.4.1 `cancelCtx`类
`cancelCtx`继承了`Context`，也实现了接口`canceler`:
```go
type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields
    done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}
```
- `Context`：父context
- `mu`：就是一个互斥锁，保证并发安全的，所以`context`是并发安全的
- `done`：用来做`context`的取消通知信号，之前的版本使用的是`chan struct{}`类型，现在用`atomic.Value`做锁优化
- `children`：`key`是接口类型`canceler`，目的就是存储实现当前`canceler`接口的子节点，当根节点发生取消时，遍历子节点发送取消信号
- `error`：当`context`取消时存储取消信息

#### 1.3.4.2 `propagateCancel`方法
```go
func propagateCancel(parent Context, child canceler) {
  // 如果返回nil，说明当前父`context`从来不会被取消，是一个空节点，直接返回即可。
    done := parent.Done()
    if done == nil {
        return // parent is never canceled
    }

  // 提前判断一个父context是否被取消，如果取消了也不需要构建关联了，
  // 把当前子节点取消掉并返回
    select {
    case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
    default:
    }

  // 这里目的就是找到可以“挂”、“取消”的context
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
    // 找到了可以“挂”、“取消”的context，但是已经被取消了，那么这个子节点也不需要
    // 继续挂靠了，取消即可
        if p.err != nil {
            child.cancel(false, p.err)
        } else {
      // 将当前节点挂到父节点的childrn map中，外面调用cancel时可以层层取消
            if p.children == nil {
        // 这里因为childer节点也会变成父节点，所以需要初始化map结构
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
    // 没有找到可“挂”，“取消”的父节点挂载，那么就开一个goroutine
        atomic.AddInt32(&goroutines, +1)
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context
		default:
			return nil, false
		}
	}
}
```
我们可以自己定制`context`，把`context`塞进一个结构时，就会导致找不到可取消的父节点，只能重新起一个协程做监听。

#### 1.3.4.3 `cancel`方法
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
  // 取消时传入的error信息不能为nil, context定义了默认error:var Canceled = errors.New("context canceled")
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
  // 已经有错误信息了，说明当前节点已经被取消过了
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
  
    c.err = err
  // 用来关闭channel，通知其他协程
    d, _ := c.done.Load().(chan struct{})
    if d == nil {
        c.done.Store(closedchan)
    } else {
        close(d)
    }
  // 当前节点向下取消，遍历它的所有子节点，然后取消
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
  // 节点置空
    c.children = nil
    c.mu.Unlock()
  // 把当前节点从父节点中移除，只有在外部父节点调用时才会传true
  // 其他都是传false，内部调用都会因为c.children = nil被剔除出去
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```
**通过源码我们可以知道`cancel`方法可以被重复调用，是幂等的**。

#### 1.3.4.4 Done 方法
```go
func (c *cancelCtx) Done() <-chan struct{} {  
    d := c.done.Load()  
    if d != nil {  
       return d.(chan struct{})  
    }  
    c.mu.Lock()  
    defer c.mu.Unlock()  
    d = c.done.Load()  
    if d == nil {  
       d = make(chan struct{})  
       c.done.Store(d)  
    }  
    return d.(chan struct{})  
}
```

### 1.3.5 `withDeadline`、`WithTimeout`的实现
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
  // 不能为空`context`创建衍生context
    if parent == nil {
        panic("cannot create context from nil parent")
    }
  
  // 当父context的结束时间早于要设置的时间，则不需要再去单独处理子节点的定时器了
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
  // 创建一个timerCtx对象
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
  // 将当前节点挂到父节点上
    propagateCancel(parent, c)
  
  // 获取过期时间
    dur := time.Until(d)
  // 当前时间已经过期了则直接取消
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
  // 如果没被取消，则直接添加一个定时器，定时去取消
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

`withDeadline`相较于`withCancel`方法也就多了一个定时器去定时调用`cancel`方法，这个`cancel`方法在`timerCtx`类中进行了重写，我们先来看一下`timerCtx`类，他是基于`cancelCtx`的，多了两个字段：
```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}
```
`timerCtx`实现的`cancel`方法，内部也是调用了`cancelCtx`的`cancel`方法取消：
```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
  // 调用cancelCtx的cancel方法取消掉子节点context
    c.cancelCtx.cancel(false, err)
  // 从父context移除放到了这里来做
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
  // 停掉定时器，释放资源
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

## 1.4 缺点
- `context`可以携带值，但是没有任何限制，类型和大小都没有限制。还有一个问题就是通过`context`携带值不如显式传值舒服，可读性变差了。
- `context`取消和自动取消的错误返回不够友好，无法自定义错误，出现难以排查的问题时不好排查。
- - 创建衍生节点实际是创建一个个链表节点，其时间复杂度为O(n)，节点多了会导致效率变低。

# 2 chan
## 2.1 channel 简介
golang 的 channel 就是一个环形队列的实现。

它有以下用法和底层实现：
1. 创建一个 channel ，这个对应了实际函数是 `makechan` 。位于 `runtime/chan.go` 文件里（下同）。
2. chan 入队，对应函数实现 `chansend` 。
3. chan 出队，分为单个值和返回ok两种方式，对应函数分别是 `chanrecv1` 和 `chanrecv2` 。
4. 在 select 中入队，对应函数实现为 `selectnbsend` 。
5. 在 select 中出队，对应函数实现为 `selectnbrecv`、`selectnbrecv2`。
6. for-range，对应使用函数 `chanrecv2`。

## 2.2 源码解析
### 2.2.1 makechan
```go
func makechan(t *chantype, size int) *hchan {
}
```
其中 t 参数是指明元素类型：
```go
type chantype struct {
 typ  _type
 elem *_type
 dir  uintptr
}
```
`makechan`做了两件事：
1. 参数校验
2. 初始化 **hchan** 结构

hchan 分配内存简单的分为三种情况：
```go
switch {
// no buffer 的场景，这种 channel 可以看成 pipe；
case mem == 0:
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    c.buf = c.raceaddr()
// channel 元素不含指针的场景，那么是分配出一个大内存块；
case elem.ptrdata == 0:
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
// 默认场景，hchan 结构体和 buffer 内存块单独分配；
default:
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
}
```
除了 hchan 结构体本身的内存分配，该结构体初始化的关键在于四个字段：
```go
// channel 的元素 buffer 数组地址；
c.buf = mallocgc(mem, elem, true)
// channel 元素大小，如果是 int，那么就是 8 字节；
c.elemsize = uint16(elem.size)
// 元素类型，这样就知道 channel 里面每个元素究竟是啥了；
c.elemtype = elem
// 元素 buffer 数组的大小，比如 make(chan int, 2)，那么这里赋值的就是 2；
c.dataqsiz = uint(size)
```

### 2.2.2 hchan 结构
```go
type hchan struct {
 qcount   uint           // queue 里面有效用户元素，这个字段是在元素出对，入队改变的；
 dataqsiz uint           // 初始化的时候赋值，之后不再改变，指明数组 buffer 的大小；
 buf      unsafe.Pointer // 指明 buffer 数组的地址，初始化赋值，之后不会再改变；
 elemsize uint16  // 指明元素的大小，和 dataqsiz 配合使用就能知道 buffer 内存块的大小了；
 closed   uint32
 elemtype *_type // 元素类型，初始化赋值；
 sendx    uint   // send index
 recvx    uint   // receive index
 recvq    waitq  // 等待 recv 响应的对象列表，抽象成 waiters
 sendq    waitq  // 等待 sedn 响应的对象列表，抽象成 waiters

 // 互斥资源的保护锁，官方特意说明，在持有本互斥锁的时候，绝对不要修改 Goroutine 的状态，不能很有可能在栈扩缩容的时候，出现死锁
 lock mutex
}
```

channel 常常会因为两种情况阻塞，1）投递的时候没有空间了，2）取出的时候还未有元素。这就就涉及到 goroutine 阻塞和 goroutine 唤醒，这个功能就跟 `recvq`，`sendq` 这两个字段有关。
waitq 类型其实就是一个双向列表的实现，和 linux 里面的 LIST 实现非常相像。
```go
type waitq struct {
 first *sudog
 last  *sudog
}
```

### 2.2.3 chansend
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // channel 的所有操作，都在互斥锁下；
    lock(&c.lock)
    // 如果投递的目标是已经关闭的 channel，那么直接 panic；
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    // 场景一：性能最好的场景，我投递的元素刚好有人在等着（那我直接给他就完了）;
    // 调用的是 send 函数，这个函数后面详细阐述，其实非常简单，递增 sendx, recvx 的索引，然后直接把元素给到等他的人，并且唤醒他；
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    // 场景二：ringbuffer 还有空间，那么把元素放好，递增索引，就可以返回了；
    if c.qcount < c.dataqsiz {
        // 复制，赋值好元素；
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep)
        // 递增索引
        c.sendx++
        // 回环空间
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        // 递增元素个数
        c.qcount++
        unlock(&c.lock)
        return true
    }
    // 判断是否需要阻塞？如果是非阻塞的，那么就直接解锁返回了，如果是阻塞的场景，那么就会走到下面的逻辑哦；
    // chan <- 和 <-chan 的场景，都是 true，但是会有其他场景这里是 false，可以提前想下？
    if !block {
        unlock(&c.lock)
        return false
    }
    // 代码走到这里，说明都是因为条件不满足，要阻塞当前 goroutine，所以做的事情本质上就是保留好通知路径，等待条件满足，会在这个地方唤醒；
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 把 goroutine 相关的线索结构入队，等待条件满足的唤醒；
    c.sendq.enqueue(mysg)
    // goroutine 切走，让出 cpu 执行权限；
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

    // 到这就是某些人唤醒该 goroutine 了。
    // 下面就是唤醒之后的逻辑了；
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    // 做一些资源的释放和环境的清理。
    gp.waiting = nil
    gp.activeStackChans = false
    if gp.param == nil {
        // 做一些校验
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```
总结来说：
1. 场景一：如果有人（ goroutine ）等着取 channel 的元素，这种场景最快，直接把元素给他就完了，然后把它唤醒，hchan 本身递增下 ringbuffer 索引；
2. 场景二：如果 ringbuffer 还有空间，那么就把元素存着，这种也是场景的流程，存和取走的是异步流程，可以把 channel 理解成消息队列，生产者和消费者解耦；
3. 场景三：ringbuffer 没空间，这个时候就要是否需要 block 了，一般来讲，`c <- x` 编译出的代码都是 `block = true` ，那么什么时候 chansend 的 block 参数会是 false 呢？答案是：select 的时候；

chansend 返回值标明元素是否 push 入队成功，成功的话，返回值为 true，否则 false 。

之前说的`selectnbasend` 只是一个代理：
```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    // 调用的就是 chansend 函数，block 参数为 false；
 return chansend(c, elem, false, getcallerpc())
}
```

### 2.2.4 chanrecv
```go
<- c
// 对应着
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

v, ok :=  <- c
// 对应着
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)       
    return
}
```
上面两种调用 block 都等于 true（同样的，只有 select 的时候，block 才会是 false ）。
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
 // 特殊场景：非阻塞模式，并且没有元素的场景直接就可以返回了，这个分支是快速分支，下面的代码都是在锁内的；
 if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
  c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
  atomic.Load(&c.closed) == 0 {
  return
 }

 // 以下所有的逻辑都在锁内；
 lock(&c.lock)

 if c.closed != 0 && c.qcount == 0 {
  if raceenabled {
   raceacquire(c.raceaddr())
  }
  unlock(&c.lock)
  if ep != nil {
   typedmemclr(c.elemtype, ep)
  }
  return true, false
 }

 // 场景：如果发现有个人（sender）正在等着别人接收，那么刚刚好，直接把它的元素给到我们这里就好了；
 if sg := c.sendq.dequeue(); sg != nil {
  recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
  return true, true
 }

 // 场景：ringbuffer 还有空间存元素，那么下面就可以把元素放到 ringbuffer 放好，递增索引，就可以返回了；
 if c.qcount > 0 {
  // 存元素
  qp := chanbuf(c, c.recvx)
  if ep != nil {
   typedmemmove(c.elemtype, ep, qp)
  }
  typedmemclr(c.elemtype, qp)
  // 递增索引
  c.recvx++
  if c.recvx == c.dataqsiz {
   c.recvx = 0
  }
  c.qcount--
  unlock(&c.lock)
  return true, true
 }

 // 代码到这说明 ringbuffer 空间是不够的，后面学会要做两个事情，是否需要阻塞？
 // 如果 block 为 false ，那么直接就退出了，返回对应的返回值；
 if !block {
  unlock(&c.lock)
  return false, false
 }

 // 到这就说明要阻塞等待了，下面唯一要做的就是给阻塞做准备（准备好唤醒的条件）
 gp := getg()
 mysg := acquireSudog()
 mysg.releasetime = 0
 mysg.elem = ep
 mysg.waitlink = nil
 gp.waiting = mysg
 mysg.g = gp
 mysg.isSelect = false
 mysg.c = c
 gp.param = nil
 // goroutine 作为一个 waiter 入队列，等待条件满足之后，从这个队列里取出来唤醒；
 c.recvq.enqueue(mysg)
 // goroutine 切走，交出 cpu 执行权限
 goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

 // 这里是被唤醒的开始的地方；
 if mysg != gp.waiting {
  throw("G waiting list is corrupted")
 }
 // 下面做一些资源的清理
 gp.waiting = nil
 closed := gp.param == nil
 gp.param = nil
 mysg.c = nil
 releaseSudog(mysg)
 return true, !closed
}
```
chanrecv 函数的返回值有两个值，selected，received。其中 selected 一般作为 select 结合的函数返回值，指明是否要进入 select-case 的代码分支，received 表明是否从队列中成功获取到元素，有几种情况：
1. 如果是非阻塞模式（ block=false ），并且没有任何可用元素，返回 （selected=false，received=false），这样就不会进到 select 的 case 分支；
2. 如果是阻塞模式（ block=true ），如果 chan 已经 closed 了，那么返回的是 （selected=true，received=false），说明需要进到 select 的分支，但是是没有取到元素的；
3. 如果是阻塞模式，chan 还是正常状态，那么返回（selected=true，recived=true），说明正常取到了元素；

# 3 锁机制
## 3.1 简介
在 Golang 里有专门的方法来实现锁，就是 sync 包，这个包有两个很重要的锁类型。一个叫 Mutex， 利用它可以实现互斥锁。一个叫 RWMutex，利用它可以实现读写锁。
- `sync.Mutex` 的锁只有一种锁：`Lock()`，它是互斥锁，同一时间只能有一个锁。
- `sync.RWMutex` 叫读写锁，它有两种锁： `RLock()` 和 `Lock()`：

## 3.2 Mutex-互斥锁
### 3.2.1 结构
Mutex 的实现主要借助了 **CAS 指令 + 自旋 + 信号量**：
```go
type Mutex struct {
    state int32
    sema  uint32
}
```
在默认情况下，互斥锁的所有状态位都是 0，state 中的不同位分别表示了不同的状态：
- 1位表示是否被锁定
- 1位表示是否有协程已经被唤醒
- 1位表示是否处于饥饿状态
- 剩下29位表示阻塞的协程数
![[image-128.png]]

`sema`表示用于阻塞和唤醒`goroutine`的信号量。
```go
// 获取信号量
runtime_SemacquireMutex(&m.sema, queueLifo, 1)

// 释放信号量
runtime_Semrelease(&m.sema, false, 1)
```

### 3.2.2 乐观到悲观
**go中的sync.Mutex制定了锁的升级过程，这个过程就是从乐观转换为悲观**：首先保持乐观，用自旋+CAS的策略竞争锁，当到达一定条件之后，判断为过于激烈，转为阻塞+唤醒模式。
1. 当自旋达到4次之后还没有结果之后；  
2. CPU单核或者gmp模型中仅有1个P调度器（这个时候自旋，其他的goroutine根本没机会释放锁，自旋纯属空转）；  
3. 当前P的执行队列中仍有待执行的G（避免因为自旋影响到GMP的调度效率）。

### 3.2.3 正常模式和饥饿模式
**正常模式**：正常模式下waiter都是先入先出，在队列中等待的waiter被唤醒后不会直接获取锁，因为要和新来的goroutine 进行竞争。新来的goroutine相对于被唤醒的waiter是具有优势的，新的goroutine 正在cpu上运行，被唤醒的waiter还要进行调度才能进入状态，所以在并发的情况下waiter大概率抢不过新来的goroutine。这个时候waiter会被放到队列的头部，如果等待的时间超过了1ms，这个时候Mutex就会进入饥饿模式。

**饥饿模式**：当Mutex进入饥饿模式之后，锁的所有权会从解锁的goroutine移交给队列头部的goroutine，这几个时候新来的goroutine会直接放入队列的尾部，这样很好的解决了老的goroutine一直抢不到锁的场景。

对于两种模式，正常模式下的性能是最好的，goroutine可以连续多次获取锁。饥饿模式解决了取锁公平的问题，但是性能会下降，其实是性能和公平的一个平衡模式。
锁状态包含：饥饿锁定、饥饿未锁定、正常锁定、正常未锁定。

### 3.2.4 Lock函数
```go
// 加锁
// 如果锁已经被使用，调用goroutine阻塞，直到锁可用
func (m *Mutex) Lock() {
    // 快速路径：没有竞争直接获取到锁，修改状态位为加锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        // 开启-race之后会进行判断，正常情况可忽略
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    // 慢路径（以便快速路径可以内联）
    m.lockSlow()
}
```
这里的代码注释提到了内联：
> [!info] 内联
> 简单的说方法内联就是将被调用方函数代码“复制”到调用方函数中，减少函数调用开销。在2018年之前的go版本中，所有的逻辑都在Lock函数中，并没有拆出来，2018年之后Go开发者将slow path拆出来。
> 
> 当lock方法被频繁调用的时候，有两种情况：如果直接获得锁走的是fast path，这个时候内联就只有fast path 的代码，这样会减少方法调用的堆栈空间和时间的消耗；如果处于自旋，锁竞争的情况下，走的是slow path，这个时候才会把lock slow 的方法内联进来。

#### 3.2.4.1 lockSlow 函数
```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64 //记录请求锁的初始时间
    starving := false //饥饿标记
    awoke := false //唤醒标记
    iter := 0 //自旋次数
    old := m.state  //当前锁的状态
    for {
        //锁处于正常模式还没有释放的时候，尝试自旋
        //如果锁变为饥饿状态或者已经解锁了，或者不符合自旋条件了就结束自旋
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {        
	        // 如果等待队列有协程、锁没有设置唤醒状态，就努力CAS设置唤醒状态 
	        // 这么做的好处是，当锁解锁的时候，不会去唤醒已经阻塞的协程，保证自己更大概率获取到锁
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 
            && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            //更新锁的状态
            old = m.state
            continue
        }

        new := old
        // 此处说明锁变为饥饿状态或者已经解锁了，或者不符合自旋条件了（仍为锁定状态）
        // 如果是正常状态，尝试加锁。饥饿状态下要出让竞争权利，肯定不能加锁的
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }

        // 如果锁还是被占用的或者锁是饥饿状态，只能将自己放到等待队列上
        // 到了这个阶段，遇到这些状态，协程只能躺平。饥饿状态要出让竞争权利
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }

        // 如果自身已经到饥饿状态了，而且锁是被占用情况下，将锁改为饥饿状态
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }

        // 如果该协程设置锁的唤醒状态，需要将唤醒状态进行重置。
        // 因为该协程要么获得了锁、要么进入休眠，都和唤醒状态无关了
        if awoke {
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
                
		// old -> new 
		// (0,1)正常且已锁定 -> (+1,1?,1) 等待加一，状态待定，加锁 -> 加到等待队列 
		// (0,0)正常且未锁定 -> (+0,0 ,1) 等待不变，正常状态，加锁 -> 加锁成功 
		// (1,1)饥饿且已锁定 -> (+1,1?,1) 等待加一，饥饿待定，加锁 -> 加到等待队列 
		// (1,0)饥饿且未锁定 -> (+1,1 ,0) 等待加一，饥饿状态，不加锁 -> 加到等待队列
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
	        // CAS成功
	        // 如果锁为未锁定且正常状态，表明占有锁成功，加锁操作完毕
            if old&(mutexLocked|mutexStarving) == 0 {
                break 
            }
            // 判断是不是第一次加入队列
            // 如果之前就在队列里面等待了，加入到队头
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }

			// 走到此处，说明协程没有获取到锁，调用runtime_SemacquireMutex，将该协程挂起 
			// waitStartTime能够判断该协程是新来的还是被唤醒的 
			// 如果是新来的，则加入队列尾部，等待唤醒，queueLifo=false 
			// 如果是从等待队列中唤醒的，则加入队列头部，queueLifo=true 
			// 如果后面该协程被唤醒，就从该位置继续往下执行
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			// 此刻说明该协程被唤醒了，锁的所有权已经传递给了当前协程
			// 判断该协程是否长时间没有获取到锁，如果是的话，就是饥饿的协程
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 协程被挂起的时间有点长，需要重新获取一下当前锁的状态
            old = m.state
            // 表示当前是饥饿状态的情况。按照设定，饥饿状态下，被唤醒的协程直接获得锁。
            if old&mutexStarving != 0 {
	            // 饥饿状态下，我被唤醒，结果发现锁没释放、唤醒值是1、等待列表没有协程了（不把我算作协程了）,抛出异常
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // delta表示锁的状态变化量。mutexLocked 是锁的占用标志位，1<<mutexWaiterShift 表示等待队列中协程数量的变化
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 如果唤醒等待队列的协程不饥饿、或者这个协程是等待队列中最后一个协程，就改为正常状态
                if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式               
                    delta -= mutexStarving
                }
                // 将锁状态设置为等待数量减一，同时设置为锁定。加锁成功
                atomic.AddInt32(&m.state, delta)
                break
            }
            // 本协程千真万确就是被系统唤醒的协程
            awoke = true
            // 自旋次数重置为0
            iter = 0
        } else {
	        //如果CAS失败，则重新开始
            old = m.state
        }
    }
    // -race开启检测冲突，可以忽略
    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

#### 3.2.4.2 总结
1. **先尝试走快速路径**，使用CAS尝试获取锁
2. 没抢到，**再尝试走慢速路径**
	1. **先尝试自旋**：如果锁是**正常模式、被锁定且满足自旋条件**，就先尝试自旋。否则如果锁变成饥饿模式、或解锁了、或不满足自旋条件了，就接着后面第2步走
		1. 如果等待队列有协程、锁没有设置唤醒状态，就努力CAS设置唤醒状态，保证自己大概率抢到锁
		2. 然后跳过此次循环，再自旋检查锁状态
	2. **接下来计算一个新的锁状态**
		1. 如果锁不是饥饿模式，就尝试抢锁（设置锁状态位）
		2. 如果锁是被占有or饥饿模式，就将等待协程数+1
		3. 如果协程自己是饥饿状态了并且锁被占用，就尝试将锁设为饥饿模式
		4. 如果该协程设置了锁的唤醒状态，需要将唤醒状态进行重置。因为接下来协程要么抢到锁，要么休眠，都和唤醒状态无关了
		5. **总结：**
			1. **锁是正常且已锁定，则协程加到等待队列**
			2. **锁是正常未锁定，则加锁成功**
			3. **锁是饥饿未锁定，则加到等待队列**
			4. **锁是饥饿已锁定，则加到等待队列**
	3. **将上面计算的锁状态使用CAS设置到锁的state中**，设置成功则
		1. **如果是抢锁成功，则break循环**
		2. **否则把协程加到等待队列中并挂起**。如果不是第一次加到队列中，则加到队列头部。调用信号量加锁函数，把协程挂起
		3. **挂起的协程被唤醒了，锁的所有权已经传递给了当前协程**。重新计算锁状态
			1. 如果阻塞时间有点长，则协程处于饥饿
			2. 如果唤醒的协程不饥饿、或者这个协程是等待队列中最后一个协程，就改为正常状态
		4. **最后将锁状态设置为等待数量减一，同时设置为锁定。加锁成功**
	4. **CAS失败，则再次进入循环**

### 3.2.5 Unlock
```go
// 如果在解锁的时候，锁是没有被锁定的，则报运行时错误。
func (m *Mutex) Unlock() {
   if race.Enabled { //默认是false
      _ = m.state
      race.Release(unsafe.Pointer(m))
   }

   // 不是饥饿状态，没有等待的协程、没有唤醒，直接解锁完毕
   new := atomic.AddInt32(&m.state, -mutexLocked)
   // 说明可能为饥饿状态、有等待协程、有唤醒的协程，事情没处理完，还得继续处理
   if new != 0 {
      m.unlockSlow(new)
   }
}

func (m *Mutex) unlockSlow(new int32) {
   if (new+mutexLocked)&mutexLocked == 0 {
      throw("sync: unlock of unlocked mutex")
   }
   //如果是正常模式
   if new&mutexStarving == 0 {
      old := new
      for {
         // 如果等待列表里没有协程了，或者已经有唤醒的协程了（三个标志位任意一个不唯一），就无需浪费精力唤醒其它协程了
         if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
            return
         }
         // 等待协程数量减1，并将锁的唤醒位置为1
         new = (old - 1<<mutexWaiterShift) | mutexWoken
         if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema, false, 1)
            return
         }
         old = m.state
      }
   } else {//如果是饥饿模式
      // 饥饿模式下，直接将锁的所有权交给等待队列中的第一个
      // 注意：锁的locked位没有被设置，唤醒的协程后面会进行设置
      // 尽管没有设置locked位，但是在饥饿模式下，新来的协程也是无法获取到锁的。
      runtime_Semrelease(&m.sema, true, 1)
   }
}
```
总结：
1. 使用原子操作解锁，如果只有一个协程，直接解锁成功返回。
2. 如果还有协程阻塞，则需要进入慢速路径：
	1. 如果处于正常模式
		1. 如果阻塞列表中没有协程、或者已经有活跃的协程（三个标志位任意一个不唯一），就交给其他协程处理即可，自己直接结束
		2. 否则等待协程数-1，锁的唤醒标志设为1，再结束
	2. 否则处于饥饿模式，调用信号量唤醒函数，唤醒等待队列第一个协程

## 3.3 RWMutex-读写锁
读写锁就是一种能保证：
- 并发读操作之间不互斥；
- 并发写操作之间互斥；
- 并发读操作和写操作互斥；

### 3.3.1 使用Mutex实现写写互斥
在看源码之前我们不妨先思考一下，如果自己实现，需要怎么设计这个数据结构来满足上面那三个要求。
首先，为了满足第二点和第三点要求，肯定需要一个互斥锁：
```go
type RWMutex struct{
    w Mutex // held if there are pending writers
    ...
}
```
这个互斥锁是在写操作时使用的：
```go
func (rw *RWMutex) Lock(){
    ...
    rw.w.Lock()
    ...
}

func (rw *RWMutex) Unlock(){
    ...
    rw.w.Unlock()
    ...
}
```

### 3.3.2 记录读goroutine数，实现写读互斥
而读操作之间是不互斥的，因此读操作的RLock()过程并不获取这个互斥锁。但读写之间是互斥的，那么RLock()如果不获取互斥锁又怎么能阻塞住写操作呢？go语言的实现是这样的：
```go
type RWMutex struct{
    w           Mutex // held if there are pending writers
    // 通过一个int32变量记录当前正在读的goroutine数：
    readerCount int32 // number of pending readers
    ...
}
```
每次调用Rlock方法时将readerCount加1，对应地，每次调用RUnlock方法时将readerCount减1：
```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 如果readerCount小于0则通过同步原语阻塞住，否则将readerCount加1后即返回
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}

func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
    // 如果readerCount减1后小于0，则调用rUnlockSlow方法，将这个方法剥离出来是为了RUnlock可以内联，这样能进一步提升读操作时的取锁性能
        rw.rUnlockSlow(r)
    }
}
```

#### 3.3.2.1 写锁加锁时操作读goroutine数
既然每次RLock时都会将readerCount增加，那判断它是否小于0有什么意义呢？这就需要和写操作的取锁过程Lock()参看：
```go
func (rw *RWMutex) Lock() {
    // 通过rw.w.Lock阻塞其它写操作
    rw.w.Lock()
    // 将readerCount减去一个最大数（2的30次方，RWMutex能支持的最大同时读操作数），这样readerCount将变成一个小于0的很小的数，
    // 后续再调RLock方法时将会因为readerCount<0而阻塞住，这样也就阻塞住了新来的读请求
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 等待之前的读操作完成
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```
**总结一下写锁Lock的流程：**
1. **阻塞新来的写操作；**
2. **阻塞新来的读操作；**
3. **等待之前的读操作完成；**

那之前未完成的读操作怎么处理呢？很简单，只要跟踪写操作Lock之前未完成的reader数就行了，这里通过一个int32变量readerWait来做这件事情：
```go
type RWMutex struct{
    w           Mutex // held if there are pending writers
    readerCount int32 // number of pending readers
    readerWait  int32 // number of departing readers
    ...
}
```
每次写操作Lock时会将当前readerCount数量记在readerWait里。

#### 3.3.2.2 读锁解锁的慢速路径
当写操作Lock后readerCount会小于0，这时reader unlock时会执行rUnlockSlow方法，现在可以来看它的实现过程了：
```go
func (rw *RWMutex) rUnlockSlow(r int32) {
    if r+1 == 0 || r+1 == -rwmutexMaxReaders {
        throw("sync: RUnlock of unlocked RWMutex")
    }
    // 每个reader完成读操作后将readerWait减小1
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 当readerWait为0时代表writer等待的所有reader都已经完成了，可以唤醒writer了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

### 3.3.3 总结
将上面这些过程梳理一下：
1. **写锁之间的阻塞，是通过一个阻塞锁实现的；而写锁、读锁的阻塞，是通过读锁协程数实现的**
2. 如果没有writer请求进来，则每个reader开始后只是将readerCount增1，完成后将readerCount减1，整个过程不阻塞，这样就做到“并发读操作之间不互斥”；
3. 当有writer请求进来时首先通过互斥锁阻塞住新来的writer，做到“并发写操作之间互斥”；
	1. 然后将readerCount改成一个很小的值，从而阻塞住新来的reader；
	2. 记录writer进来之前未完成的reader数量，等待它们都完成后再唤醒自己
	3. 这样就做到了“并发读操作和写操作互斥”；
4. writer结束后将readerCount置回原来的值，保证新的reader不会被阻塞，然后唤醒之前等待的reader，再将互斥锁释放，使后续writer不会被阻塞。

---
# 4 引用
[context原理](https://segmentfault.com/a/1190000040917752)
[锁原理](https://juejin.cn/post/6986473832202633229#heading-18)
[万字Mutex原理](https://blog.csdn.net/theaipower/article/details/146487121)
[读写锁原理](https://segmentfault.com/a/1190000039712353)