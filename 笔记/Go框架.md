2025-06-18 16:22
Status: #idea
Tags: [[Go]]

# 1 Hertz
## 1.1 路由原理
Hertz 框架的路由原理主要基于前缀树（Prefix Tree，也称Trie树）来实现高效的路由匹配。当接收到HTTP请求时，Hertz会根据请求的URL路径在树形结构中进行查找，找到最匹配的处理函数。
以下是Hertz路由原理的更详细解释：
1. 前缀树结构:
	- Hertz使用前缀树来存储和管理路由。每个节点代表URL路径的一部分（例如，`/`，`users`，`posts`）。
	- 从根节点开始，每个节点的子节点代表该节点路径的下一个可能的路径段。
	- 通过这种结构，可以快速地查找匹配的路由，因为相同的URL前缀可以共享节点，减少了不必要的遍历。
2. 路由匹配过程:
	- 当一个请求到达Hertz时，框架会提取请求的URL路径。
	- 从前缀树的根节点开始，根据路径段逐层向下查找。
	- 如果当前节点有匹配的子节点，则沿着该子节点继续查找；如果没有匹配的子节点，则表示没有找到匹配的路由。
	- 如果找到一个与URL路径完全匹配的节点，则表示找到了匹配的路由，并执行相应的处理函数。
	- 在匹配过程中，Hertz还会处理一些特殊情况，例如：
	    - **尾部斜杠处理:** Hertz默认会自动处理尾部斜杠，根据路由定义自动重定向。
	    - **参数路由:** 支持类似 `/users/:id` 这样的带参数的路由，在匹配时会提取参数值。﻿
3. 中间件:
	- Hertz支持中间件机制，可以在请求到达路由处理函数之前或之后执行一些通用逻辑，例如权限验证、日志记录等。
	- 中间件可以看作是添加到路由匹配过程中的额外处理步骤。

# 2 Thrift
## 2.1 简介
Thrift 是 Facebook 开源的**跨语言 RPC 框架**，核心目标是解决不同语言服务间的高效通信问题。它通过统一的 IDL（接口定义语言）定义服务和数据结构，生成多语言桩代码，结合灵活的序列化 / 传输 / 服务模型，实现高性能跨语言 RPC。

## 2.2 核心原理
Thrift 采用分层架构（自下而上），每层可灵活替换，这是其跨语言、高性能的核心：

| 层级           | 核心作用               | 关键实现 / 考点                                                       |
| ------------ | ------------------ | --------------------------------------------------------------- |
| IDL 层        | 统一定义数据结构 / 服务接口    | 语法（结构体 / 服务 / 字段 ID）、字段 ID 的作用（减少传输体积、兼容字段扩展）                   |
| Protocol 层   | 序列化 / 反序列化（内存↔字节流） | TBinaryProtocol（二进制，高性能）、TCompactProtocol（压缩）、TJSONProtocol（调试） |
| Transport 层  | 网络传输字节流            | TFramedTransport（帧化传输，解决 TCP 粘包）、TSocket（基础 TCP）                |
| Server Model | 服务端请求处理模型          | TThreadPoolServer（线程池）、THsHaServer（半同步半异步）                      |

## 2.3 核心流程
以 “客户端调用 GetUser” 为例：
1. 客户端调用 Stub 方法→Protocol 序列化参数；
2. Transport（如 TFramedTransport）加长度前缀→TCP 发送；
3. 服务端 Transport 拆帧→Protocol 反序列化参数；
4. 服务端 Skeleton 调用业务逻辑→序列化返回值；
5. 传输回客户端→反序列化得到结果。

## 2.4 关键考点解析
### 2.4.1 Thrift 的核心优势是什么？
1. 跨语言：统一 IDL 支持多语言；
2. 高性能：二进制序列化 + 帧化传输；
3. 灵活：每层可替换（如切换序列化协议）；
4. 一站式：内置传输 / 服务模型，无需重复造轮子。

### 2.4.2 Thrift IDL 中字段 ID 的作用？
1. 替代字段名传输，减少字节体积；
2. 兼容字段扩展：新增字段只要 ID 不重复，老版本不会报错；
3. 序列化时通过 ID 映射字段，不依赖字段顺序。

### 2.4.3 Thrift 如何解决 TCP 粘包？
TCP 是流式协议，粘包是因为请求无边界。Thrift 通过 TFramedTransport 给每个请求加 4 字节长度前缀，服务端先读长度再读数据，精准拆分。

#### 2.4.3.1 TCP粘包 / 拆包到底是什么
首先要明确TCP 的核心特性：**TCP 是「流式协议」**，而非「数据包协议」（比如 UDP）。
可以把 TCP 连接想象成一根 “水管”，数据是水管里的 “水流”—— 水流本身没有天然的 “分段边界”，你往水管里倒 3 杯水（3 个请求），另一端接水时，可能收到：
- 「粘包」：一次性接到 3 杯水混在一起（多个请求的数据粘成一段）；
- 「拆包」：1 杯水被拆成 2 次接（一个请求的数据被拆成多段）
本质是**应用层无法区分 TCP 流中 “一个请求” 的开始和结束位置**。

粘包 / 拆包的核心原因：
1. **发送端**：TCP 有 Nagle 算法（合并小数据包发送，减少网络交互），会把多个小请求合并成一个 TCP 包发送；
2. **接收端**：TCP 有接收缓冲区，数据先存缓冲区，应用层读取不及时的话，缓冲区会堆积多个请求的数据；
3. **网络层面**：数据包在网络中可能被分片 / 重组，导致边界丢失。

举个具体例子：客户端连续调用 2 次`GetUser(1)`和`GetUser(2)`，发送的数据是`[数据1][数据2]`，TCP 传输后，服务端可能收到：
- 粘包：`[数据1+数据2]`（一次读到 2 个请求）；
- 拆包：`[数据1的前半部分]` + `[数据1的后半部分+数据2]`（两次读取，第一次不完整，第二次混了两个请求）；
如果没有边界标识，服务端根本分不清哪段是第一个请求、哪段是第二个。

#### 2.4.3.2 FramedTransport 解决粘包的详细机制
核心思路是：**给每个请求 / 响应加「帧头（长度前缀）」，让应用层能通过 “长度” 识别请求边界**。

##### 2.4.3.2.1 帧的结构（关键）
`TFramedTransport`会把每个请求封装成一个「帧（Frame）」，结构如下：
- 「4 字节长度前缀」：固定占 4 个字节，用**大端序（网络字节序）** 存储，值是「数据体的字节数」；
- 「数据体」：Thrift Protocol 层序列化后的请求数据（比如`GetUser(1)`的二进制数据）。

##### 2.4.3.2.2 发送端（客户端 / 服务端响应时）的处理流程
1. 客户端调用`GetUser(1)`，Thrift 先通过 Protocol 层把参数序列化成二进制数据（比如长度 16 字节）；
2. `TFramedTransport`接收这个 16 字节的数据体，计算长度（16）；
3. 把长度 16 转换成 4 字节的大端序二进制（比如`0x00 0x00 0x00 0x10`）；
4. 把「4 字节长度前缀 + 16 字节数据体」拼接成 20 字节的帧，通过 TCP 发送出去；

##### 2.4.3.2.3 接收端（服务端 / 客户端）的拆帧流程
1. 接收端先从 TCP 缓冲区读取**前 4 字节**（长度前缀）；
2. 把 4 字节的大端序数据转换成十进制数字（比如 16），这个数字就是 “当前请求的数据体长度”；
3. 根据这个长度（16），从 TCP 缓冲区读取接下来的 16 字节，这就是完整的一个请求数据体；
4. 如果 TCP 缓冲区还有剩余数据，重复 Step1-Step3，处理下一个帧；

这样，即使多个帧粘在一起（比如`[帧1][帧2]`），也能通过 “先读长度→再读对应字节” 精准拆分。

### 2.4.4 二进制协议为什么高效
二进制协议比 JSON 高效：无冗余字符（如 {}）、字段 ID 替代字段名、数值用 varint 紧凑编码”。

#### 2.4.4.1 varint 紧凑编码
##### 2.4.4.1.1 简介
常规的数值编码（比如 int32 固定占 4 字节、int64 固定占 8 字节）是「定长编码」—— 不管数值是 1（很小）还是 2147483647（int32 最大值），都占用相同字节数，这对小数值来说是极大的空间浪费。
varint（Variable-Length Integer，变长整数编码）是一种「变长编码方式」：**小数值用更少的字节存储，大数值用更多字节存储**，整体平均占用字节数远低于定长编码，这也是 Thrift 二进制协议高效的核心原因之一。

Thrift 中 varint 主要用于编码整数类型（i8/i16/i32/i64），比如：
- 数值 1 用 varint 编码仅占 1 字节（定长 int32 占 4 字节）；
- 数值 128 用 varint 编码占 2 字节（定长 int32 占 4 字节）；
- 数值 2^28-1 用 varint 编码占 4 字节（和定长 int32 持平）。

##### 2.4.4.1.2 varint 的核心原理
varint 编码的核心规则只有两条，非常简单：
1. **字节的最高位（第 8 位）是「续位标志」**：
	- 1 → 表示当前字节不是最后一个，后面还有字节需要拼接；
	- 0 → 表示当前字节是最后一个，结束拼接。
2. **每个字节的低 7 位是「有效数据位」**：
    - 数值的二进制会被拆分成若干个 7 位的片段；
    - 片段按「低位在前、高位在后」的顺序（小端序）填充到每个字节的低 7 位；
    - 最后一个字节的最高位设为 0，其余字节的最高位设为 1。

##### 2.4.4.1.3 ZigZag 编码
varint 本身只适合编码**非负整数**，但 Thrift 要处理负数。计算机中负数用「补码」表示，比如 int64 的 - 1，补码是`64个1`，如果直接用 varint 编码，会占满 8 字节（因为每字节最高位都是 1，直到最后一个字节），完全失去 varint “紧凑” 的意义。
ZigZag 的核心目的：**把正负整数「zigzag 式」映射到非负整数区间**，让负数也能像小正数一样用 varint 高效编码，同时保留 “区分正负” 的信息。

编码公式：
- 64 位整数（i64）：`ZigZag(n) = (n << 1) ^ (n >> 63)`
- 32 位整数（i32）：`ZigZag(n) = (n << 1) ^ (n >> 31)`
公式拆解：
- `n << 1`：把原数左移 1 位，空出最低位（用来存 “正负标记”）；
- `n >> 63`/`n >> 31`：符号位扩展 —— 正数右移结果是 0，负数右移结果是`-1`（补码中全 1）；
- `^`（异或）：把 “符号位扩展结果” 和 “左移后的数” 异或，最终让编码值的**最低位**成为 “正负标记”：
    - 原数为正 → 最低位 = 0（编码值是偶数）；
    - 原数为负 → 最低位 = 1（编码值是奇数）。

解码公式：`n = (zigzag >> 1) ^ -(zigzag & 1)`
- `zigzag >> 1`：把编码值右移 1 位，还原出原数的 “绝对值部分”；
- `zigzag & 1`：取编码值的最低位（0 = 偶 = 正，1 = 奇 = 负）；
- `-(zigzag & 1)`：最低位为 1 时结果是`-1`（全 1），为 0 时结果是 0；
- `^`（异或）：如果是负数，用`-1`异或 “绝对值部分” 还原补码；如果是正数，用 0 异或，值不变。

|原数 n|ZigZag 编码值（偶 / 奇）|编码过程（64 位）|解码过程|
|---|---|---|---|
|0|0（偶→正）|(0<<1) ^ (0>>63) = 0 ^ 0 = 0|(0>>1) ^ -(0&1) = 0 ^ 0 = 0|
|1|2（偶→正）|(1<<1) ^ (1>>63) = 2 ^ 0 = 2|(2>>1) ^ -(2&1) = 1 ^ 0 = 1|
|-1|1（奇→负）|(-1<<1) ^ (-1>>63) = -2 ^ -1 = 1|(1>>1) ^ -(1&1) = 0 ^ -1 = -1|
|2|4（偶→正）|(2<<1) ^ (2>>63) = 4 ^ 0 = 4|(4>>1) ^ -(4&1) = 2 ^ 0 = 2|
|-2|3（奇→负）|(-2<<1) ^ (-2>>63) = -4 ^ -1 = 3|(3>>1) ^ -(3&1) = 1 ^ -1 = -2|

ZigZag 编码后的数，**最低位是 0（偶数）→ 原数是正数；最低位是 1（奇数）→ 原数是负数**。

### 2.4.5 Thrift 和 Protobuf 的区别？
1. Thrift 是完整 RPC 框架，Protobuf 仅序列化协议（需配合 gRPC）；
2. Thrift IDL 更贴近业务（支持异常 / 服务定义），Protobuf 序列化效率略高

## 2.5 总结
- Thrift 是一个支持多语言的RPC框架，它有几层结构
	- IDL 层：这是一种接口定义语言，使用 IDL 编写接口描述，然后用代码生成器生成不同语言的客户端、服务端
	- 协议层：它定义了请求序列化、反序列化的方式，支持二进制、JSON等格式
	- 传输层：它定义了帧化传输，解决了TCP粘包/拆包问题
	- 服务层：它定义了服务端的请求处理模型，例如线程池
- Thrift关键要点
	- IDL 里结构体的字段都有数字 id
		- 在传输数据时不用传输名字，只需要传输id就可以，减少了数据传输。
		- 兼容性好：可以新增字段，只要id不重复，老版本IDL也不会报错
	- 帧化传输：传输请求响应数据时，都会在开头加上数据长度，这样客户端/服务器先读长度再读数据，解决了TCP粘包/拆包问题
	- 二进制编码协议：
		- 无冗余字符（对比JSON）
		- 通过变长整形+ZigZag编码，减少了整数的字节大小

# 3 Kitex
## 3.1 简介
Kitex 是字节基于 Thrift 开发的**Go 专属 RPC 框架**，并非重构，而是对 Thrift 的 Go 实现做深度优化：
- 完全兼容 Thrift IDL：你写的 Kitex IDL 本质就是 Thrift IDL；
- 复用核心流程：IDL 定义→代码生成→客户端调用→服务端处理，完全继承 Thrift 逻辑；
- 优化性能：Kitex 自研更高效的序列化协议（兼容 Thrift）、协程化服务模型，解决原生 Thrift Go 版性能短板。

## 3.2 核心原理
### 3.2.1 Kitex对Thrift的沾包解决方法做了哪些优化？
Kitex 基于此优化：增加帧校验、超时控制，结合 Go 的 IO 多路复用（epoll）提升高并发拆帧效率，同时适配协程模型。
1. **帧校验增强**：在 4 字节长度前缀基础上，增加了「魔数 / 校验位」，防止恶意数据或网络异常导致的长度解析错误；
2. **超时控制**：接收端读取长度前缀 / 数据体时，增加超时机制（比如 3 秒没凑够数据就断开连接），避免长连接被无效数据阻塞；
3. **协程适配**：结合 Go 的协程模型，拆帧逻辑用非阻塞 IO 实现，一个协程可处理多个连接的拆帧，提升高并发下的效率；
4. **内存复用**：预分配帧缓冲区，避免频繁创建 / 销毁字节数组，减少 GC 开销（原生 Thrift 的 Go 版没有这个优化）。

### 3.2.2 Kitex 对 varint 的优化
Kitex 完全兼容 Thrift 的 varint 编码逻辑，还做了针对性优化：
1. **预计算常见数值的 varint 长度**：比如 1-127 这些高频字段 ID / 小数值，提前计算好编码长度，避免运行时重复计算，减少 CPU 开销；
2. **内存复用**：编码时复用 varint 缓冲区，避免频繁创建字节数组，降低 Go 的 GC 压力；
3. **批量编码优化**：对多个整数（比如批量字段 ID）做连续 varint 编码，减少 IO 调用次数。

### 3.2.3 原生 Thrift 和 Kitex 的服务模型有什么区别？
原生 Thrift 用线程池（TThreadPoolServer），适配 Java 但不贴合 Go 协程生态；Kitex 重构为协程池模型，基于 IO 多路复用处理请求，并发能力远高于原生，还集成了熔断、限流等治理能力。
IO 多路复用参见：[[开发基础知识#3 IO 模型]]

Kitex 作为 Go 专属 RPC 框架，把 “非阻塞 IO+epoll 多路复用 + 协程模型” 深度结合，解决原生 Thrift 的性能瓶颈：
1. **IO 层**：所有 socket 设为非阻塞模式，避免单个连接阻塞协程；
2. **多路复用**：基于 Go 的`net`包（底层封装 epoll），用一个 “主协程” 调用 epoll_wait () 监听所有连接的就绪事件；
3. **协程调度**：epoll 通知就绪 fd 后，主协程把 fd 的处理逻辑交给「协程池」，协程用非阻塞 recv () 拷贝数据，处理 RPC 请求；
4. **极致优化**：
    - 用 epoll 的边缘触发（ET）减少通知次数；
    - 结合 Go 的协程调度（用户态协程映射到内核线程），避免内核线程上下文切换；
    - 对比原生 Thrift 的 “线程池 + BIO”：Kitex 用 “协程池 + epoll”，单台机器可支撑 10 万 + 并发连接，而原生 Thrift 仅能支撑几千。

## 3.3 总结
- Kitex 是字节基于 Thrift 开发的，针对 Go 的高性能RPC框架
- 它兼容 Thrift 的 IDL 和分层架构，此外使用协程池+IO多路复用提升了处理性能，还新增了限流熔断等治理能力。

# 4 Gorm
## 4.1 简明使用教程
Gorm 是 Go 语言主流的 ORM 框架，核心是**将 Go 结构体操作映射为 SQL 语句**，底层基于 Go 标准库 `database/sql` 封装。
参见：https://gorm.io/zh_CN/docs/models.html

### 4.1.1 快速入门
```go
package main

import (
  "gorm.io/driver/sqlite"
  "gorm.io/gorm"
)

type Product struct {
  gorm.Model
  Code  string
  Price uint
}

func main() {
  db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
  if err != nil {
    panic("failed to connect database")
  }

  // Migrate the schema
  db.AutoMigrate(&Product{})

  // Create
  db.Create(&Product{Code: "D42", Price: 100})

  // Read
  var product Product
  db.First(&product, 1) // find product with integer primary key
  db.First(&product, "code = ?", "D42") // find product with code D42

  // Update - update product's price to 200
  db.Model(&product).Update("Price", 200)
  // Update - update multiple fields
  db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // non-zero fields
  db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})

  // Delete - delete product
  db.Delete(&product, 1)
}
```
**Gorm v1.30.0 之后还加入了泛型API**。

### 4.1.2 连接到数据库
#### 4.1.2.1 打开连接
MySQL：
```go
import (
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

func main() {
  dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
  db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

SQLite:
```go
import (
  "gorm.io/driver/sqlite" // 基于 CGO 的 Sqlite 驱动
  // "github.com/glebarez/sqlite" // 纯 Go 实现的 SQLite 驱动, 详情参考：https://github.com/glebarez/sqlite
  "gorm.io/gorm"
)

// github.com/mattn/go-sqlite3
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{})
```
您也可以使用 `file::memory:?cache=shared` 替代文件路径。 这会告诉 SQLite 在系统内存中使用一个临时数据库。

#### 4.1.2.2 连接池设置
GORM 使用 [database/sql](https://pkg.go.dev/database/sql) 来维护连接池
```go
sqlDB, err := db.DB()

// SetMaxIdleConns 设置空闲连接池中连接的最大数量。
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns 设置打开数据库连接的最大数量。
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime 设置了可以重新使用连接的最大时间。
sqlDB.SetConnMaxLifetime(time.Hour)
```

### 4.1.3 模型定义
#### 4.1.3.1 基础
模型是使用普通结构体定义的。 这些结构体可以包含具有基本Go类型、指针或这些类型的别名，甚至是自定义类型（只需要实现 `database/sql` 包中的Scanner和Valuer接口）。
- 具体数字类型如 `uint`、`string`和 `uint8` 直接使用。
- 指向 `*string` 和 `*time.Time` 类型的指针表示可空字段。
- 来自 `database/sql` 包的 `sql.NullString` 和 `sql.NullTime` 用于具有更多控制的可空字段。
- `CreatedAt` 和 `UpdatedAt` 是特殊字段，当记录被创建或更新时，GORM 会自动向内填充当前时间。
- 非导出字段会被忽略。
- **嵌入结构体**：对于匿名字段，GORM 会将其字段包含在父结构体中。对于正常的结构体字段，你也可以通过标签 `embedded` 将其嵌入 `gorm:"embedded"`。并且，您可以使用标签 `embeddedPrefix` 来为 db 中的字段名添加前缀，例如：`gorm:"embedded;embeddedPrefix:author_"`。

#### 4.1.3.2 约定
1. **主键**：GORM 使用一个名为`ID` 的字段作为每个模型的默认主键。
2. **表名**：默认情况下，GORM 将结构体名称转换为 `snake_case` 并为表名加上复数形式。
3. **列名**：GORM 自动将结构体字段名称转换为 `snake_case` 作为数据库中的列名。
4. **时间戳字段**：GORM使用字段 `CreatedAt` 和 `UpdatedAt` 来自动跟踪记录的创建和更新时间。
5. **软删除字段**：`DeletedAt` 用于软删除（将记录标记为已删除，而实际上并未从数据库中删除）。
6. GORM提供了一个预定义的结构体，名为`gorm.Model`，其中包含常用字段：
	```go
// gorm.Model 的定义
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
	```

#### 4.1.3.3 字段标签
| 标签名                    | 说明                                                                                                                                                                                                                                       |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| column                 | 指定 db 列名                                                                                                                                                                                                                                 |
| type                   | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| serializer             | 指定将数据序列化或反序列化到数据库中的序列化器, 例如: `serializer:json/gob/unixtime`                                                                                                                                                                              |
| size                   | 定义列数据类型的大小或长度，例如 `size: 256`                                                                                                                                                                                                             |
| primaryKey             | 将列定义为主键                                                                                                                                                                                                                                  |
| unique                 | 将列定义为唯一键                                                                                                                                                                                                                                 |
| default                | 定义列的默认值                                                                                                                                                                                                                                  |
| precision              | 指定列的精度                                                                                                                                                                                                                                   |
| scale                  | 指定列大小                                                                                                                                                                                                                                    |
| not null               | 指定列为 NOT NULL                                                                                                                                                                                                                            |
| autoIncrement          | 指定列为自动增长                                                                                                                                                                                                                                 |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                                                                                                                                                                                                                         |
| embedded               | 嵌套字段                                                                                                                                                                                                                                     |
| embeddedPrefix         | 嵌入字段的列名前缀                                                                                                                                                                                                                                |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano`                                                                                                                                                |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli`                                                                                                                                            |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://gorm.io/zh_CN/docs/indexes.html) 获取详情                                                                                                                                                        |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                                                                                                                                                                                                                   |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://gorm.io/zh_CN/docs/constraints.html) 获取详情                                                                                                                                                     |
| <-                     | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限                                                                                                                                                                 |
| ->                     | 设置字段读的权限，`->:false` 无读权限                                                                                                                                                                                                                 |
| -                      | 忽略该字段，`-` 表示无读写，`-:migration` 表示无迁移权限，`-:all` 表示无读写迁移权限                                                                                                                                                                                  |
| comment                | 迁移时为字段添加注释                                                                                                                                                                                                                               |

#### 4.1.3.4 其他
[高级选项](https://gorm.io/zh_CN/docs/models.html#%E9%AB%98%E7%BA%A7%E9%80%89%E9%A1%B9)

## 4.2 Gorm 核心底层原理
核心原理可拆解为 8 个关键模块：

### 4.2.1 核心架构分层
- **抽象层**：定义 `DB`/`Statement`/`Session` 等核心接口，解耦具体数据库实现；
- **核心层**：处理 SQL 生成、参数绑定、结果映射、事务管理、钩子回调等核心逻辑；
- **驱动层**：适配 MySQL/PostgreSQL/SQLite 等数据库，封装底层驱动，处理不同数据库的方言差异（如自增语法、分页语法）；
- **插件层**：支持读写分离、缓存、审计等扩展（如 `dbresolver` 读写分离插件）。

### 4.2.2 SQL 生成原理（核心中的核心）
Gorm 把结构体操作转 SQL 的核心流程：
1. **结构体解析**：通过 `reflect` 反射解析结构体的 `gorm` 标签（如 `gorm:"column:name;primaryKey"`），映射表名、字段名、约束（主键 / 自增 / 非空等）；
2. **条件构建**：`Where`/`Order`/`Limit` 等方法将条件转化为「条件节点」，最终拼接为 `WHERE`/`ORDER BY` 等子句；
3. **SQL 拼接**：根据操作类型（Create/Update/Query/Delete），结合结构体映射信息 + 条件节点，拼接标准 SQL，同时适配数据库方言；
4. **参数绑定**：使用「预处理语句（PrepareStatement）」，将参数绑定到 SQL 的 `?` 占位符（避免 SQL 注入，提升性能）。

### 4.2.3 ORM 映射原理
- **表 / 字段映射**：默认规则（结构体名蛇形→表名，如 `UserInfo`→`user_infos`；字段名蛇形→列名，如 `UserName`→`user_name`），可通过 `gorm` 标签覆盖；
- **类型映射**：内置 Go 类型→数据库类型映射表（如 `int`→`INT`、`time.Time`→`DATETIME`），支持自定义类型（实现 `Scanner/Valuer` 接口）；
- **结果映射**：查询后通过反射将 `database/sql.Rows` 中的行数据赋值到结构体字段，支持嵌套结构体、关联映射。

### 4.2.4 事务处理原理
- **手动事务**：`db.Begin()` 获取 `Tx` 实例（绑定单个数据库连接），所有操作在该连接执行；`Commit()` 提交 /`Rollback()` 回滚，未提交的 `Tx` 不会归还连接到池；
- **自动事务**：`db.Transaction(func(tx *gorm.DB) error)` 封装 `Begin/Commit/Rollback`，函数返回 `error` 则回滚，否则提交；嵌套事务通过「保存点（SavePoint）」实现；
- **隔离级别**：通过 `Set("gorm:tx_options", "ISOLATION LEVEL READ COMMITTED")` 设置，底层执行 `SET TRANSACTION` 语句。

### 4.2.5 钩子函数（Hooks）原理
钩子是生命周期回调，基于「反射 + 方法名匹配」实现：
- 生命周期阶段：`BeforeCreate/AfterCreate`（创建）、`BeforeUpdate/AfterUpdate`（更新）、`BeforeDelete/AfterDelete`（删除）、`BeforeFind/AfterFind`（查询）；
- 执行逻辑：执行对应 SQL 前后，Gorm 反射检查结构体是否实现钩子方法，若实现则调用；**批量操作（如 CreateInBatches）默认不触发钩子**，需手动处理。

### 4.2.6 预加载（Preload）原理
解决 N+1 查询问题的核心，流程：
1. 查询主模型（如 `User`），获取主模型 ID 列表；
2. 根据外键批量查询关联模型（如 `User.Articles`，外键 `user_id`）；
3. 反射将关联模型映射到主模型的关联字段；

补充：`Joins` 预加载通过 `JOIN` 语句一次性查询，适合一对一关联；`Preload` 适合一对多 / 多对多。

### 4.2.7 连接池原理
Gorm 复用 `database/sql` 的连接池机制：
- `SetMaxOpenConns`：最大打开连接数（同时使用的连接数）；
- `SetMaxIdleConns`：最大空闲连接数（池子里保持的空闲连接）；
- `SetConnMaxLifetime`：连接最大存活时间（避免长期占用连接）；
- `DB` 实例并发安全，每次操作从池获取连接，用完归还；`Tx` 实例绑定单个连接，未提交 / 回滚则不归还。

## 4.3 Gorm 高频面试题（附标准答案，可直接背）
### 4.3.1 基础类
1. Gorm 中常用的结构体标签有哪些？作用是什么？
	- `gorm:"column:xxx"`：指定字段映射到数据库的列名（覆盖默认蛇形命名）；
	- `gorm:"primaryKey"`：标记字段为主键（复合主键可多个字段标记）；
	- `gorm:"autoIncrement"`：标记字段为自增（仅支持数值类型）；
	- `gorm:"default:xxx"`：设置字段默认值（注意：创建时未传值才会用默认值）；
	- `gorm:"not null"`：标记字段非空；
	- `gorm:"size:255"`：设置字段长度（如 VARCHAR (255)）；
	- `gorm:"softDelete:flag"`：标记字段为软删除标识（默认 `deleted_at`）；
	- `gorm:"foreignKey:xxx"`：指定关联的外键；
	- `gorm:"many2many:xxx"`：指定多对多关联的中间表名。
2. Gorm 如何实现批量创建 / 更新？底层原理是什么？
	- 批量创建：`db.CreateInBatches(users, 100)`（第二个参数是批次大小），底层拼接 `INSERT INTO ... VALUES (...), (...), ...` 批量 SQL，减少网络往返；
	- 批量更新：`db.Model(&User{}).Where("age > ?", 18).Updates(map[string]interface{}{"status": 1})`（批量更新同字段），或 `db.UpdateInBatches(users, 100)`（批量更新不同字段）；
	- 注意：批量操作默认不触发钩子函数，需手动遍历调用钩子。
3. Gorm 的软删除原理是什么？如何禁用软删除？
	- 原理：软删除不是真正删除数据，而是给 `deleted_at` 字段赋值（默认 `NULL` 表示未删除）；Gorm 查询 / 更新 / 删除时，会自动拼接 `WHERE deleted_at IS NULL` 条件，过滤已软删除数据；
	- 禁用软删除的方式：
	    1. 结构体不嵌入 `gorm.Model`（`gorm.Model` 包含 `deleted_at`）；
	    2. 查询时使用 `Unscoped()`：`db.Unscoped().Find(&users)`（查询所有数据，包括软删除）；
	    3. 删除时使用 `Unscoped()`：`db.Unscoped().Delete(&user)`（物理删除）。
4. Preload 和 Joins 的区别？适用场景？
	- 核心区别：
	    1. `Preload`：分两次查询（先查主模型，再批量查关联模型），底层是「IN 语句」，适合**一对多 / 多对多**关联；
	    2. `Joins`：通过 `JOIN` 语句一次性查询主模型 + 关联模型，适合**一对一 / 一对多（少量数据）** 关联；
	- 场景：
	    - 查用户 + 用户的多篇文章 → 用 `Preload("Articles")`；
	    - 查用户 + 用户的基本信息（一对一）→ 用 `Joins("Profile")`；
	- 注意：`Joins` 不支持嵌套预加载，`Preload` 支持（如 `Preload("Articles.Comments")`）。

### 4.3.2 进阶类
1. Gorm 如何实现自定义类型映射？（如 JSON 类型）需实现 Go 标准库的 `sql.Scanner` 和 `driver.Valuer` 接口：
	```go
type JSONMap map[string]interface{}

// Valuer：将自定义类型转化为数据库可接收的类型
func (j JSONMap) Value() (driver.Value, error) {
  return json.Marshal(j)
}

// Scanner：将数据库返回的值转化为自定义类型
func (j *JSONMap) Scan(value interface{}) error {
  return json.Unmarshal(value.([]byte), j)
}

// 结构体使用
type User struct {
  ID    uint     `gorm:"primaryKey"`
  Info  JSONMap  `gorm:"type:json"` // 指定数据库类型为 json
}
	```
2. Gorm 如何避免 SQL 注入？底层原理是什么？
	- 核心方式：Gorm 所有参数化查询都使用「预处理语句 + 参数绑定」，而非字符串拼接；
	- 原理：将 SQL 语句和参数分离，SQL 语句先预编译，参数通过 `?` 占位符绑定，数据库会将参数作为纯数据处理，而非 SQL 语句的一部分，从而避免注入；
	- 注意：若手动拼接 SQL（如 `Raw("SELECT * FROM users WHERE name = '"+name+"'")`），仍有注入风险，需使用 `Raw` 的参数绑定：`db.Raw("SELECT * FROM users WHERE name = ?", name)`。
3. Gorm 的 DB 实例和 Tx 实例有什么区别？
	- `DB` 实例：
	    1. 并发安全，可全局复用；
	    2. 每次操作从连接池获取连接，用完归还；
	    3. 无事务上下文，操作是独立的；
	- `Tx` 实例：
	    1. 绑定单个数据库连接，所有操作都在该连接执行；
	    2. 非并发安全，需避免多 goroutine 使用；
	    3. 有事务上下文，需手动 `Commit()`/`Rollback()`，否则连接不会归还到池。
4. 嵌套事务的实现：Gorm 嵌套事务基于数据库的 SavePoint（保存点）实现，内层事务 `Begin()` 会创建 SavePoint，`Rollback()` 只会回滚到该保存点，不会影响外层事务，`Commit()` 则无操作，最终由外层事务统一提交

### 4.3.3 性能优化类
1. Gorm 如何优化 N+1 查询问题？N+1 问题是指「查 1 个主模型列表（1 次查询），再循环查每个主模型的关联模型（N 次查询）」，优化方式：
	1. **Preload**：批量查询关联模型（1 次主查询 + 1 次关联查询），适合一对多 / 多对多；
	2. **Joins**：通过 `JOIN` 一次性查询主模型 + 关联模型，适合一对一；
	3. **自定义 SQL**：手动拼接 `IN` 语句查询关联模型，减少查询次数。
2. Gorm 开启 PrepareStmt 的作用？底层原理？
	- 作用：预编译 SQL 语句并缓存，后续执行相同 SQL 时复用预编译语句，减少数据库解析 SQL 的开销，提升性能；
	- 原理：开启 `db.Config{PrepareStmt: true}` 后，Gorm 会为每个 SQL 语句创建 `database/sql.Stmt`（预编译语句）并缓存，执行时直接绑定参数执行，无需重复解析 SQL；
	- 配置示例：
	```go
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		PrepareStmt: true, // 开启预编译缓存
	})
	```
3. Gorm 慢查询如何排查？
	1. **开启 Gorm 日志**：配置日志级别为 `Info`，打印所有 SQL 及执行时间：
	```go
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	```
	2. **自定义日志拦截**：通过 Gorm 的 `Callback` 或插件，记录执行时间超过阈值（如 500ms）的 SQL；
	3. **数据库层面**：开启 MySQL 慢查询日志（`slow_query_log = 1`，`long_query_time = 0.5`），定位慢 SQL；

---
# 5 引用