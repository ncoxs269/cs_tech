2025-11-27 15:41
Status: #idea
Tags: [[Go]]

# 1 pprof 使用指南
## 1.1 简介
pprof 是 Go 内置的**性能分析工具**，支持对 CPU、内存、协程、阻塞、锁竞争等维度的性能数据进行采集、分析和可视化，是排查 Go 程序性能问题（如协程泄漏、CPU 占用高、内存溢出）的核心工具。

 pprof 基于 Google 的 `pprof` 可视化工具，Go 内置 `net/http/pprof` 包（用于暴露性能数据）和 `runtime/pprof` 包（用于程序内手动采集）。它支持两种工作模式：
- **采样模式**（默认）：对目标指标（如 CPU、内存）进行周期性采样（非精确统计，但开销低，适合生产环境）；
- **精确模式**：通过手动埋点统计（如阻塞时间、锁竞争，开销略高，但数据精确）；

支持的核心指标（对应 `/debug/pprof` 端点）：

|端点|作用|常用场景|
|---|---|---|
|`/goroutine?debug=2`|查看所有协程的详细栈信息（含状态）|协程泄漏、协程阻塞排查|
|`/profile?seconds=N`|采集 N 秒内的 CPU 采样数据|CPU 占用高、耗时函数排查|
|`/heap`|采集内存分配 / 使用数据（堆内存）|内存泄漏、内存占用高排查|
|`/block`|采集 goroutine 阻塞等待数据（如 channel、锁）|阻塞性能瓶颈排查|
|`/mutex`|采集锁竞争数据|锁冲突导致的性能下降排查|
|`/allocs`|采集所有内存分配历史（包括已释放的）|内存分配频率过高排查|
|`/threadcreate`|采集系统线程创建数据|线程泄漏排查|

## 1.2 实操教程：从「配置→采集→分析」全流程
### 1.2.1 第一步：开启 pprof 数据暴露（两种方式）
#### 1.2.1.1 方式 1：HTTP 方式（推荐生产环境，无需修改业务代码）
通过 `net/http/pprof` 包自动注册 `/debug/pprof` 端点，只需在程序中导入包并启动 HTTP 服务：
```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // 自动注册端点，无需额外代码
    "time"
)

// 模拟一个耗时函数（用于后续 CPU 分析）
func busyWork() {
    sum := 0
    for i := 0; i < 100000000; i++ {
        sum += i
    }
}

func main() {
    // 启动 HTTP 服务，暴露 6060 端口（默认端点 /debug/pprof）
    go func() {
        log.Printf("pprof server start at :6060")
        log.Fatal(http.ListenAndServe(":6060", nil))
    }()

    // 模拟业务逻辑：持续执行耗时操作
    for {
        busyWork()
        time.Sleep(100 * time.Millisecond)
    }
}
```

#### 1.2.1.2 方式 2：程序内手动采集（适合无 HTTP 服务的场景，如 CLI 工具）
通过 `runtime/pprof` 包手动创建性能数据文件，后续用 `go tool pprof` 分析：
```go
package main

import (
    "os"
    "runtime/pprof"
    "time"
)

func busyWork() {
    sum := 0
    for i := 0; i < 100000000; i++ {
        sum += i
    }
}

func main() {
    // 1. 采集 CPU 数据（持续 10 秒）
    cpuFile, err := os.Create("cpu.pprof")
    if err != nil {
        log.Fatal("create cpu.pprof failed:", err)
    }
    defer cpuFile.Close()
    pprof.StartCPUProfile(cpuFile)
    defer pprof.StopCPUProfile()

    // 2. 采集内存数据（程序结束时写入）
    memFile, err := os.Create("mem.pprof")
    if err != nil {
        log.Fatal("create mem.pprof failed:", err)
    }
    defer memFile.Close()
    defer pprof.WriteHeapProfile(memFile)

    // 模拟业务逻辑
    for i := 0; i < 10; i++ {
        busyWork()
        time.Sleep(100 * time.Millisecond)
    }
}
```
运行程序后，会在当前目录生成 `cpu.pprof` 和 `mem.pprof` 文件。

### 1.2.2 第二步：核心用法：用 `go tool pprof` 分析数据
`go tool pprof` 是命令行工具，支持两种数据来源：
- 远程 HTTP 端点（`go tool pprof http://localhost:6060/debug/pprof/XXX`）；
- 本地文件（`go tool pprof cpu.pprof`）。

以下以「远程 HTTP 方式」为例，讲解常用场景的分析流程。

#### 1.2.2.1 场景 1：排查协程泄漏
核心是查看协程的栈信息和状态，步骤：
1. **实时查看协程总数和状态**：
	- 在浏览器输入 `http://localhost:6060/debug/pprof/goroutine?debug=1`，会展示总的协程数和每个协程的核心函数调用栈
	- 在浏览器输入 `http://localhost:6060/debug/pprof/goroutine?debug=2`，会展示每个协程的ID和状态（running、sleep等），还有完整调用栈。调用栈最下面还会显示 `created by`，表示它是被哪个协程创建的
2. **交互模式分析协程**：
	```bash
	go tool pprof http://localhost:6060/debug/pprof/goroutine
	```
	交互模式下常用命令：

| 命令         | 作用                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `top`      | 按协程数量排序，显示前 10 个函数（创建协程最多的函数）                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `list 函数名` | 查看该函数的代码，标注创建协程的位置                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `web`      | 生成协程调用链火焰图（需安装 Graphviz），注意不要在 GoLand 的终端中调用，可能不行<br>图片展示的是Goroutine 的函数调用关系 + 数量分布联合图：<br>- 矩形节点代表一个函数<br>- 节点内的数字格式是「`协程数 (占总协程的百分比)`」（总协程数看你图顶部的`Total: 5 goroutines`）<br>- 箭头代表函数调用关系：箭头从「调用者函数」指向「被调用者函数」<br>- 箭头旁的数字代表「通过这条调用路径运行的协程数量」<br>- 节点颜色（浅红）通常是「协程数量较多的函数」<br><br>这个 SVG 图需要和`debug=2`的协程栈信息配合看，效率最高：<br>1. 先看「占比高的节点」：去`debug=2`的栈信息里根据函数名找这些协程的状态<br>2. 追踪「调用路径」：从图的顶部函数往下看，能找到协程的「创建源头」（比如哪个业务入口函数启动了这些协程）—— 这能帮你定位 “泄漏协程是从哪段代码来的”。 |
| `traces`   | 生成协程调用轨迹图（可视化阻塞关系）                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `help`     | 帮助                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `exit`     | 退出                                                                                                                                                                                                                                                                                                                                                                                                                                                    |

#### 1.2.2.2 场景 2：排查 CPU 占用高（耗时函数）
1. **采集 CPU 数据**（默认采集 30 秒，可通过 `seconds=N` 指定时长）：
```bash
# 采集 10 秒内的 CPU 数据，进入交互模式
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=10
```
2. **交互模式分析 CPU**：
	- `top`：按 CPU 耗时占比排序，显示前 N 个函数（`flat` 是函数自身耗时，`cum` 是函数 + 子函数总耗时）；
	- `list busyWork`：查看 `busyWork` 函数的代码，标注每行的 CPU 耗时占比；
	- `web`：生成 CPU 火焰图（横向越长，耗时越高），直观定位耗时函数。

#### 1.2.2.3 场景 3：排查内存泄漏（堆内存占用高）
1. **采集内存数据**：
	```bash
	# 进入堆内存分析交互模式（默认采集当前堆内存使用情况）
	go tool pprof http://localhost:6060/debug/pprof/heap
	```
2. **交互模式分析内存**：
	- `top`：按内存占用排序，显示前 N 个函数。默认展示的内存指标是`inuse_space`，当前使用的内存。可以通过 `sample_index` 切换关键指标：
	    - `inuse_space`：当前堆内存使用量（未释放）——排查「内存占用高 / 内存泄漏」
	    - `inuse_objects`：当前存活的对象数量——排查「对象创建过多」
	    - `alloc_space`：累计分配的堆内存（包括已释放的）——排查「内存分配频率过高」
	    - `alloc_objects`：累计分配的对象数量——排查「频繁创建短生命周期对象」
	    - 也可以在启动 pprof 时，通过 `-sample_index=inuse_objects` 参数直接指定要查看的指标，无需进入交互模式后切换
	- `list 函数名`：查看函数中内存分配的代码行（如 `make`、`new` 操作）；
	- `web`：生成内存分配火焰图，定位内存分配频繁的函数。

#### 1.2.2.4 场景 4：排查阻塞问题（channel / 锁等待）
1. **采集阻塞数据**：
```bash
go tool pprof http://localhost:6060/debug/pprof/block
```
2. **交互模式分析**：
    - `top`：按阻塞时长排序，显示前 N 个函数；
    - `list 函数名`：查看阻塞发生的代码行（如 `<-ch`、`wg.Wait()`）；
    - `traces`：可视化阻塞链，查看哪个协程导致了阻塞。

### 1.2.3 第三步：可视化工具（火焰图 / 调用图）
pprof 的命令行模式不够直观，推荐用「可视化工具」快速定位问题，核心是安装 **Graphviz**（绘图工具）。

在 pprof 交互模式下，输入以下命令自动生成图表（会打开默认浏览器）：
- `web`：生成调用链火焰图（适合 CPU / 内存 / 协程）；
- `svg`：生成 SVG 格式图表（可保存到文件：`svg > cpu.svg`）；
- `png`：生成 PNG 格式图表（`png > mem.png`）。

## 1.3 进阶技巧（生产环境必备）
### 1.3.1 生产环境安全配置
- **限制端口访问**：pprof 端点会暴露程序敏感信息（栈、函数名），生产环境需通过防火墙限制访问 IP（如只允许内网 IP）；
- **添加认证**：给 HTTP 服务添加 Basic Auth 认证，避免未授权访问：
```go
// 示例：给 pprof 服务添加 Basic Auth
func main() {
    mux := http.NewServeMux()
    // 注册 pprof 端点（默认是 http.DefaultServeMux，这里手动注册到自定义 mux）
    mux.Handle("/debug/pprof/", http.HandlerFunc(pprof.Index))
    mux.Handle("/debug/pprof/cmdline", http.HandlerFunc(pprof.Cmdline))
    mux.Handle("/debug/pprof/profile", http.HandlerFunc(pprof.Profile))
    mux.Handle("/debug/pprof/symbol", http.HandlerFunc(pprof.Symbol))
    mux.Handle("/debug/pprof/trace", http.HandlerFunc(pprof.Trace))

    // 添加 Basic Auth 中间件
    authMux := basicAuth(mux, "admin", "123456")

    log.Fatal(http.ListenAndServe(":6060", authMux))
}

// basicAuth 中间件实现
func basicAuth(h http.Handler, username, password string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user, pass, ok := r.BasicAuth()
        if !ok || user != username || pass != password {
            w.Header().Set("WWW-Authenticate", `Basic realm="pprof"`)
            w.WriteHeader(http.StatusUnauthorized)
            return
        }
        h.ServeHTTP(w, r)
    })
}
```

### 1.3.2 远程采集生产环境数据（无直接访问权限）
若生产环境不允许直接访问 6060 端口，可通过 `kubectl port-forward`（K8s 环境）或 `ssh 端口转发` 打通通道：
```bash
# 示例 1：K8s 环境，将 pod 的 6060 端口转发到本地 6060
kubectl port-forward pod/your-pod-name 6060:6060

# 示例 2：SSH 端口转发，将远程服务器的 6060 端口转发到本地 6060
ssh -L 6060:localhost:6060 user@remote-server-ip
```
之后直接访问 `http://localhost:6060/debug/pprof` 即可采集数据。

### 1.3.3 持续性能监控（Prometheus+Grafana）
pprof 适合「问题排查」，若需「长期监控」，可将 pprof 指标导出到 Prometheus，用 Grafana 可视化：
- 使用 `prometheus/client_golang` 库，采集 `runtime_num_goroutine`、`go_gc_duration_seconds` 等指标；
- 导入 Grafana 官方 Go 性能监控面板（ID：10823），可直观查看协程数、CPU、内存、GC 等趋势。

### 1.3.4 工具辅助
- **pprof 可视化 Web UI**：Go 1.19+ 支持 `go tool pprof -http=:8080 数据来源`，直接启动 Web 界面，无需交互命令：
```bash
# 启动 Web UI 分析协程数据（访问 http://localhost:8080 即可）
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine
```
- **go-torch**：生成更美观的火焰图（支持 CPU / 内存 / 协程），github：[https://github.com/uber/go-torch](https://github.com/uber/go-torch)。

# 2 协程泄漏问题排查
协程泄漏的本质是「goroutine 无法进入退出状态」，常见原因是：**阻塞在无退出条件的操作上**（如 channel、sync 原语、IO）或**无限循环无终止逻辑**。排查需按「**确认现象→收集数据→定位阻塞点→分析根因**」的顺序推进。

## 2.1 第一步：确认现象与数据收集
### 2.1.1 确认现象：先验证是否真泄漏
- **关联系统状态**：观察 CPU、内存、GC 情况：
    - ==若协程激增但 CPU 使用率低：大概率是协程阻塞（如 channel、WaitGroup）==；
    - ==若 CPU 和协程同步增长：可能是无限循环（如无退出条件的 for 循环）==。
- **量化协程数量**：通过以下方式获取实时 goroutine 数，确认是否持续增长（而非正常波动）：
	- 代码内埋点：用 `runtime.NumGoroutine()` 打印当前协程数（如日志输出、监控暴露）；
	- 监控面板：通过 Prometheus+Grafana 暴露指标（推荐使用 `prometheus/client_golang` 库，采集 `runtime_num_goroutine` 指标）；
	- 即时查询：通过 pprof 实时获取，执行命令

### 2.1.2 收集关键数据
核心是获取「泄漏协程的栈信息」，明确它们在做什么、阻塞在哪里。
#### 2.1.2.1 工具 1：pprof（首选，最直接）
使用 pprof 实时查看 goroutine 栈，可以先用 debug=2 查看协程状态和阻塞位置，然后用可视化火焰图查看调用链。

#### 2.1.2.2 工具 2：日志与上下文追踪
- 在 goroutine 创建时添加日志（记录 goroutine ID、业务场景）：
```go
// 注意：Go无直接获取goroutine ID的API，需通过runtime.Stack间接获取
func getGoroutineID() uint64 {
    var buf [64]byte
    runtime.Stack(buf[:], false)
    var id uint64
    fmt.Sscanf(string(buf[:]), "goroutine %d ", &id)
    return id
}

// 业务中创建协程时打印
go func() {
    log.Printf("goroutine start: id=%d, task=xxx", getGoroutineID())
    // 业务逻辑...
    log.Printf("goroutine exit: id=%d", getGoroutineID()) // 若未输出，说明泄漏
}()
```
- 若使用分布式追踪（如 Jaeger、Zipkin），可将 goroutine ID 与追踪 ID 绑定，定位泄漏协程对应的业务请求。

#### 2.1.2.3 工具 3：runtime/debug 包
在程序中主动 dump 栈信息（适用于生产环境无法直接执行 pprof 命令的场景）：
```go
// 当协程数超过阈值时，dump所有goroutine栈到文件
if runtime.NumGoroutine() > 10000 { // 警戒线，根据业务调整
    f, err := os.Create("goroutine_dump_" + time.Now().Format("20060102150405") + ".log")
    if err != nil {
        log.Printf("dump goroutine stack failed: %v", err)
        return
    }
    defer f.Close()
    debug.WriteGoroutineStacks(f)
    log.Printf("dump goroutine stack to file, current count: %d", runtime.NumGoroutine())
}
```

## 2.2 第二步：定位根因 —— 常见协程泄漏场景与识别
### 2.2.1 channel使用不当
例如：
- 对nil channel进行读写
- 无缓冲 Channel「只发不收」或「只收不发」
- 带缓冲 Channel「缓冲满后持续发送」或「缓冲空后持续接收」

识别：
pprof 栈信息中出现 `chan send` 或 `chan receive` 阻塞，例如：
```
goroutine 123 [chan send]:
main.taskFunc(...)
    /path/main.go:45 +0x42
created by main.main
    /path/main.go:20 +0x8a
```

解决方案：
- 确保发送和接收成对出现；
- 若不确定接收者是否存在，用 `select+default` 避免阻塞
- 用带缓冲 Channel（需合理设置缓冲大小，避免缓冲满导致发送阻塞）；
- 用 `context.Context` 控制超时 / 取消
- 用「有界通道 + 限流」避免生产者压满缓冲（如使用 `golang.org/x/sync/semaphore` 限流）。

### 2.2.2 sync.WaitGroup 使用不当（Add/Done 不匹配）
`WaitGroup` 用于等待一组协程完成，若 `Add(n)` 后，`Done()` 调用次数少于 `n`，则 `Wait()` 会永久阻塞；若 `Done()` 调用次数多于 `Add(n)`，会导致 panic。

识别：
pprof 栈信息中出现 `sync.WaitGroup.Wait` 阻塞，例如：
```
goroutine 456 [sync.WaitGroup.Wait]:
sync.(*WaitGroup).Wait(0xc0000b4000)
    /usr/local/go/src/sync/waitgroup.go:136 +0x54
main.batchTask(...)
    /path/main.go:89 +0x12a
```

解决方案：
- 严格保证 `Add(n)` 与 `Done()` 次数一致，`Done()` 必须放在 `defer` 中（避免业务逻辑 panic 导致未调用）；
- 避免在循环中动态 `Add`（易出错），建议先计算协程数，一次性 `Add`；
- 若协程可能提前退出，需确保每个协程无论成功失败都调用 `Done()`。

### 2.2.3 互斥锁未释放
可能由于忘记调用 `Unlock()`、临界区 panic 导致 `Unlock()` 未执行。

识别：
pprof 中协程状态为 `blocked`，栈信息显示阻塞在 `sync.Mutex.Lock`，例如：
```
goroutine 123 [blocked]:
sync.(*Mutex).Lock(0xc0000b4000)
    /usr/local/go/src/sync/mutex.go:81 +0x8a
main.processData(...)
    /path/main.go:66 +0x3c
created by main.main
    /path/main.go:22 +0x7d
```

解决方案：
- 强制用 `defer` 释放锁（核心规范）
- 避免在临界区执行可能 panic 的逻辑

### 2.2.4 死锁
死锁是「多个协程互相持有对方需要的资源，形成循环等待」，导致所有参与的协程**永久阻塞**（属于更严重的协程泄漏，且会导致业务逻辑卡死）。常见触发条件：
1. 至少两个协程；
2. 至少两个资源（锁、channel 等）；
3. 协程间互相持有资源并等待对方释放。

识别：
1. **pprof 栈信息**：多个协程阻塞在「资源等待」（如 `Mutex.Lock`、`chan receive`），且等待的资源被其他阻塞协程持有；
2. **程序表现**：CPU 使用率极低（协程都阻塞），业务功能卡死，协程数持续增长（若有新协程不断进入死锁循环）；
3. **工具检测**：使用 `go-deadlock` 库（替换 `sync.Mutex`），运行时自动检测死锁并打印详细日志。

解决方案：
- **统一资源获取顺序**（避免循环等待）：若多个协程需要获取多把锁 / 资源，强制按「固定顺序」获取（如按变量名字典序、资源 ID 升序），打破循环等待
- **给资源等待添加超时**：用 `context.WithTimeout` 或 `select+time.After` 限制资源等待时间，超时后释放已持有资源并退出协程
- **使用工具提前检测死锁**：开发 / 测试环境用 `go-deadlock` 库替换 `sync.Mutex`/`sync.RWMutex`，自动检测死锁并打印调用栈

### 2.2.5 select 语句无 default 分支，且所有 case 均无法触发
例如：
- select 监听未使用的 channel
- select 监听已关闭但无数据的 channel

识别：
协程状态为 `blocked`，栈信息显示阻塞在 `select` 语句，例如：
```
goroutine 456 [blocked]:
main.waitForData(...)
    /path/main.go:55 +0x90
created by main.main
    /path/main.go:25 +0x6a

// 对应代码第55行：select 无 default，所有 case 无法触发
```

解决：
- 根据场景选择是否加 default 分支：
    - 若期望「立即返回，不阻塞」：加 `default` 分支处理「无 case 触发」的情况；
    - 若期望「等待某个 case 触发，但需避免永久阻塞」：用 `context` 或 `time.After` 加超时。
- 确保至少一个 case 能触发，或者用 `context.Done()` 作为退出 case（推荐所有业务协程都加）

### 2.2.6 context 使用不当
`context.Context` 是协程间传递取消信号、超时控制的核心机制。若：
- 协程未接收 `context.Done()` 信号；
- 父 context 未取消 / 超时，子协程无退出条件；
- 未将 context 传递给子协程。
会导致协程永久运行。

识别：
pprof 栈信息中协程处于「运行中」或阻塞在业务逻辑（如循环），无 `context.Done()` 相关逻辑。

解决方案：
- 所有业务协程必须接收 `context.Context`，并通过 `select` 监听 `Done()` 信号；
- 父协程退出时，通过 `context.WithCancel` 主动取消子协程；
- 耗时操作（如 IO、循环）必须设置超时（`context.WithTimeout`）

### 2.2.7 IO 操作无超时（网络、数据库、Redis 等）
Go 的 IO 操作（如 `http.Get`、`sql.Query`、`redis.Do`）默认是**阻塞的**，若对方服务无响应、网络超时，协程会永久阻塞在 IO 等待上。

识别：
pprof 栈信息中阻塞在 IO 相关函数，例如：
```
goroutine 789 [IO wait]:
net.(*netFD).Read(...)
    /usr/local/go/src/net/fd_unix.go:202 +0x2a
net/http.(*conn).readRequest(...)
    /usr/local/go/src/net/http/server.go:1073 +0x12c
```

解决方案：所有 IO 操作必须设置超时：
- HTTP 请求：用 `http.Client{Timeout: 5 * time.Second}`；
- 数据库（如 MySQL）：连接时设置 `timeout`、`readTimeout`、`writeTimeout`；
- Redis（如 redigo）：用 `SetReadTimeout`、`SetWriteTimeout`，或通过 context 控制；
- RPC请求：需要设置超时，且超时时长不要太长，避免高并发下大量协程阻塞

### 2.2.8 Timer 泄漏（time.After 使用不当）
`time.After(d)` 会创建一个定时器，1 次触发后释放，但如果在循环中使用（如 `select` 里的 `case <-time.After(1*time.Second)`），每次循环都会创建新的 Timer，旧 Timer 需等待触发后才会被 GC，若循环频率高，会导致大量 Timer 和协程泄漏。

识别：
pprof 栈信息中出现大量 `time.sleep` 相关协程，例如：
```
goroutine 101 [sleep]:
time.Sleep(...)
    /usr/local/go/src/runtime/time.go:195 +0x105
time.AfterFunc.func1()
    /usr/local/go/src/time/sleep.go:176 +0x36
```

解决方案：
- 循环中使用 `time.NewTimer` 并手动 `Reset`，避免重复创建：
```go
func main() {
    go func() {
        timer := time.NewTimer(1 * time.Second)
        defer timer.Stop()
        for {
            select {
            case <-timer.C:
                log.Println("tick")
                timer.Reset(1 * time.Second) // 重置定时器
            }
        }
    }()
}
```
- 若需超时控制，优先使用 `context.WithTimeout` 而非 `time.After`。

### 2.2.9 无限循环无退出条件
协程内的 `for` 循环缺少退出逻辑（如未监听 `context.Done()`、未判断终止条件），导致协程永久运行。

识别：
pprof 栈信息中协程状态为 `running`，且栈跟踪指向循环逻辑。

解决方案：
循环中必须添加退出条件（如 `context.Done()`、计数器、外部信号）

锁使用不当
互斥锁未释放，其他协程获取锁时阻塞；
死锁，多个协程相互等待对方释放资源
select 语句无 default 分支

## 2.3 第三步：解决泄漏后的验证
修复后需验证协程数量是否恢复正常：
1. 观察监控面板：协程数是否稳定在合理范围（无持续增长）；
2. 重复执行压测：模拟高并发场景，确认协程数不会随请求量无限增加；
3. 再次 dump 栈信息：通过 pprof 确认泄漏的协程已退出，无新增阻塞协程。

## 2.4 第四步：预防措施（长期避免泄漏）
### 2.4.1 编码规范（强制执行）
- 所有协程必须有「明确退出条件」：要么监听 `context.Done()`，要么有业务终止逻辑；
- 所有 IO 操作（HTTP、数据库、Redis、MQ）必须设置超时；
- Channel 使用原则：「发送方负责关闭，接收方负责判断关闭」，避免关闭后发送或接收；
- `sync.WaitGroup` 规范：`Done()` 必须放在 `defer` 中，`Add` 次数与协程数严格一致；
- 禁止在循环中创建无控制的协程（如 `for range data { go func() {}() }`），需用协程池（如 `ants` 库）限制并发数。

### 2.4.2 监控告警
- 暴露 `runtime_num_goroutine` 指标到 Prometheus，设置阈值告警（如超过 1 万触发告警）；
- 监控阻塞协程数：通过 pprof 采集 `goroutine:blocked` 指标，异常增长时告警；
- 监控 IO 超时率：若某类 IO 操作超时率激增，可能伴随协程泄漏。

### 2.4.3 测试与审查
- 单元测试：检测协程泄漏，例如：
```go
func TestTaskNoLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    // 执行被测函数（会创建协程）
    runTask(ctx)
    time.Sleep(2 * time.Second) // 等待协程退出
    after := runtime.NumGoroutine()
    if after > before+1 { // 允许1个协程误差（如pprof服务）
        t.Errorf("goroutine leak: before=%d, after=%d", before, after)
    }
}
```
- 代码审查：重点审查协程创建处（`go func()`），确认每个协程都有退出条件。

### 2.4.4 工具辅助
- 使用 `golangci-lint` 等静态检查工具，检测潜在的协程泄漏（如未使用的 Channel、WaitGroup 不匹配）；
- 生产环境开启 pprof 服务（限制访问权限），便于问题快速排查。

# 3 CPU暴涨问题排查


# 4 内存泄漏问题排查
1. 资源未关闭
	1. 文件、网络连接、数据库连接等未正确关闭
	2. 这些资源通常关联底层系统资源，未关闭，Go 的 GC 无法回收其占用的内存
2. 协程泄露
3. 切片或字符串的截取而引发的内存泄漏：
	1. 一个长的字符串被另一个字符串通过切片的方式截取，这两个字符串共用一个底层数组。
	2. 如果截取的字符串很小，但是原来的字符串很大，只要截取的小字符串还在活跃，则大的字符串将不会被回收，这样会造成临时的内存泄漏；
	3. 同理切片的截取也会存在这样的情况。
4. 全局变量或长生命周期对象的不当使用
	1. 全局变量（如 map、切片）或长生命周期对象（如单例）中存储了不再使用的数据，且未及时清理

# 5 程序响应变慢排查
- 服务器负载过高：检查服务器的CPU、内存使用率。
- 数据库查询效率低下：慢查询日志
- 检查网络：是不是网络带宽不足或不稳定
- 查询数据量过大
- 外部依赖（如第三方服务）响应慢


---
# 6 引用