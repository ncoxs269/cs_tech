2025-04-27 09:38
Status: #idea
Tags: [[MySQL]]


# 1 日志系统：一条SQL更新语句是如何执行的
我们还是从一个表的一条更新语句说起，下面是这个表的创建语句，这个表有一个主键 `ID` 和一个整型字段 `c`：
```sql
mysql> create table T(ID int primary key, c int);
```
如果要将 `ID=2` 这一行的值加 1，SQL 语句就会这么写：
```sql
mysql> update T set c=c+1 where ID=2;
```
首先，可以确定的说，查询语句的那一套流程，更新语句也是同样会走一遍。

与查询流程不一样的是，更新流程还涉及两个重要的**日志模块**，它们正是我们今天要讨论的主角：**redo log（重做日志）和 binlog（归档日志）**。

## 1.1 redo log
### 1.1.1 基础概念
如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。MySQL 里使用 **WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘**。
“先写日志” 也是先写磁盘，只是因为 WAL 只需要写到日志最后，写日志是顺序写盘，速度很快。
**redo log 是物理日志**，记录的是“在某个数据页上做了什么修改”。

具体来说，就是以下步骤：
1. 当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。
2. 同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

### 1.1.2 结构
InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示：
![[image-28.png]]
**write pos** 是当前记录的位置，一边写一边后移。**checkpoint** 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。
write pos 和 checkpoint 之间的部分可以用来记录新的操作。如果 write pos 追上 checkpoint，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

### 1.1.3 刷脏
所谓**刷脏就是由于内存页和磁盘数据不一致导致了该内存页是“脏页”，将内存页数据刷到磁盘的操作称为“刷脏”**。为了搞清楚为什么“刷脏”会导致慢查，我们先分析下 redo log 再哪些场景会刷到磁盘。
1. redo log 写满了，此时 MySQL 会停止所有更新操作，把脏页刷到磁盘。
2. 系统内存不足，需要将脏页淘汰，此时会把脏页刷到磁盘。
3. 系统空闲时，MySQL定期将脏页刷到磁盘。
在场景 1 和 2 都会导致慢查的产生。

Innodb 存储引擎的刷脏策略是怎么样的呢？通常而言会有两种策略：**全量（sharp checkpoint）和部分（fuzzy checkpoint）**。全量刷脏发生在关闭数据库时，部分刷脏发生在运行时。部分刷脏又分为定期刷脏、最近最少使用刷脏、异步/同步刷脏、脏页过多刷脏。

在崩溃恢复场景中，InnoDB 如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后**让 redo log 更新内存内容**。更新完成后，内存页变成脏页，就回到了第一种情况的状态。

### 1.1.4 change buffer
参见 [[MySQL-索引#3.4 change buffer 和 redo log]]

## 1.2 binlog
**redo log 是 InnoDB 引擎特有的日志**，而 **Server 层也有自己的日志，称为 binlog（归档日志）**。
最开始 MySQL 里并没有 InnoDB 引擎。MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的。既然**只依靠 binlog 是没有 crash-safe 能力的**，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。

这两种日志有以下三点不同：
1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. **redo log 是物理日志**，记录的是“在某个数据页上做了什么修改”；**binlog 是逻辑日志**，记录的是这个语句的原始逻辑，比如“给 `ID=2` 这一行的 `c` 字段加 1 ”。
    [binlog 有三种模式](https://www.cnblogs.com/baizhanshi/p/10512399.html)：statement 格式的话是记 sql 语句；row 格式会记录行的内容，记两条，更新前和更新后都有；mixed 格式是其他两种格式的混合，MySQL 会根据执行的每一条具体 SQL 语句来区分对待记录的日志形式，也就是在 statement 和 row 之间选择一种。
3. **redo log 是循环写的**，空间固定会用完；**binlog 是可以追加写入的**。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

之所以 binlog 没有 crash-safe 能力，因为 binlog 是追加写，crash 时不能判定 binlog 中哪些内容是已经入库，哪些还没被写入。而 **redo log 是循环写，从 check point 到 write pos 间的内容都是未入库的**。
假如现在重新设计 MySQL，只用一个 binlog 是否可以实现 cash_safe 能力呢？答案是可以的，只不过 binlog 中也要加入 checkpoint，数据库故障重启后，binlog checkpoint 之后的 sql 都重放一遍。但是这样做让 binlog 耦合的功能太多。

binglog的一个用处L：当你需要扩容的时候，也就是需要再多搭建一些备库来增加系统的读能力的时候，现在常见的做法也是用全量备份加上应用 binlog 来实现的。

## 1.3 更新语句执行过程
我们再来看执行器和 InnoDB 引擎在执行这个简单的 `update` 语句时的内部流程。
1. 执行器先找引擎取 `ID=2` 这一行。`ID` 是主键，引擎直接用树搜索找到这一行。如果 `ID=2` 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
    需要注意并不会更新某条记录就把某条记录查询到内存中对其做修改就行，而是**将对应记录所在页都加载到内存中**。MySQL 和磁盘交互基本单位是页，默认是 16KB。使用 `show variables like 'innodb_page_size'` 命令，可以查询页的大小。
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 **prepare 状态**。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成**提交（commit）状态**，更新完成。

## 1.4 两阶段提交
为什么必须有“两阶段提交”呢？这是为了让两份日志之间的逻辑一致。这里不妨用反证法来进行解释。
由于 redo log 和 binlog 是两个独立的逻辑，如果不用两阶段提交，要么就是先写完 redo log 再写 binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。
假设当前 `ID=2` 的行，字段 `c` 的值是 0，再假设执行 `update` 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash，会出现什么情况呢？
1. **先写 redo log 后写 binlog**。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 `c` 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 `c` 的值就是 0，与原库的值不同。
2. **先写 binlog 后写 redo log**。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 `c` 的值是 0。但是 binlog 里面已经记录了“把 `c` 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。
两阶段提交就是为了给所有人一个机会，当每个人都说“我 ok”的时候，再一起提交。

### 1.4.1 崩溃恢复
我们先来看一下崩溃恢复时的判断规则。
1. 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
2. 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
    1. 如果是，则提交事务；
    2. 否则，回滚事务。

之所以还要跑断 binlog 是否存在，是因为这时候 binlog 已经写入了，之后就会被从库（或者用这个 binlog 恢复出来的库）使用。

#### 1.4.1.1 binlog完整性的校验
一个事务的 binlog 是有完整格式的：
- statement 格式的 binlog，最后会有 COMMIT；
- row 格式的 binlog，最后会有一个 XID event。
另外，在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 checksum 的结果来发现。所以，MySQL 还是有办法验证事务 binlog 的完整性的。

### 1.4.2 redo log 和 binlog 的关联
它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：
- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

## 1.5 日志写入机制
### 1.5.1 binlog写入机制
binlog 的写入逻辑比较简单：事务执行过程中，先把日志写到 **binlog cache**，事务提交的时候，再把 binlog cache 写到 binlog 文件中。
一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 binlog cache 的保存问题。
系统给 binlog cache 分配了一片内存，每个线程一个，参数 `binlog_cache_size` 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。状态如图 1 所示：
![[image-80.png]]
可以看到，每个线程有自己 binlog cache，但是共用同一份 binlog 文件。
- 图中的 write，指的就是指把日志写入到文件系统的 page cache（它是OS关于磁盘IO的缓存，位于内核中），并没有把数据持久化到磁盘，所以速度比较快。
- 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。

`write` 和 `fsync` 的时机，是由参数 `sync_binlog` 控制的：
1. `sync_binlog=0` 的时候，表示每次提交事务都只 `write`，不 `fsync`；
2. `sync_binlog=1` 的时候，表示每次提交事务都会执行 `fsync`；
3. `sync_binlog=N(N>1)` 的时候，表示每次提交事务都 `write`，但累积 N 个事务后才 `fsync`。
在出现 IO 瓶颈的场景里，将 `sync_binlog` 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值。
因为 page cache 是系统机制，只要主机不挂，进程重启是不会丢失数据的。但是如果主机发生异常重启，会丢失最近 N 个事务的 `binlog` 日志。

### 1.5.2 redo log 的写入机制
#### 1.5.2.1 redo log buffer
在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：
```sql
begin;
insert into t1 ...
insert into t2 ...
commit;
```
插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。
所以，**redo log buffer 就是一块内存，用来先存 redo 日志的**。真正把日志写到 redo log 文件（文件名是 ib_logfile+ 数字），是在执行 commit 语句的时候做的（后面会介绍其他场景的写入）。

binlog 是每个线程都有一个 binlog cache，而 **redo log 是多个线程共用一个 redo log buffer**。

#### 1.5.2.2 为什么redo log buffer 是全局共用的
binlog 是一种逻辑性的日志，记录的是一个事务完整的语句，是一个原子性操作。当用来做主从同步，如果分散写，可能造成事务不完整，分多次执行，从而导致不可预知的问题。而 redo log 属于物理性的日志，记录的是物理地址的变动，因此，分散写也不会改变最终的结果。

#### 1.5.2.3 redo log buffer 怎么持久化到磁盘？
binlog要事务提交才会写入磁盘。那么 redo log buffer 中的部分日志会不会没有提交就被持久化到磁盘呢？答案是确实会有。这个问题，要从 redo log 可能存在的三种状态说起。这三种状态，对应的就是图 2 中的三个颜色块。
![[image-81.png]]
这三种状态分别是：
1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中，就是图中的红色部分；
2. 写到磁盘 (write)，但是没有持久化（fsync)，物理上是在文件系统的 page cache 里面，也就是图中的黄色部分；
3. 持久化到磁盘，对应的是 hard disk，也就是图中的绿色部分。

为了控制 redo log 提交时的写入策略，InnoDB 提供了 `innodb_flush_log_at_trx_commit` 参数，它有三种可能取值：
1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中。只写内存，不管是主机掉电还是MySQL异常重启，都有丢数据的风险，风险高，但是写入快 ;
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘。直接写到磁盘，没有丢数据的风险，但是写入慢；
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。写入文件系统的page cache，主机掉电后会丢数据，但是MySQL异常重启不会丢数据，风险较低，写入比较快。

没有提交事务的 redo log，以下三种场景也会被写入到磁盘中：
1. **InnoDB 有一个后台线程，每隔 1 秒**，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。
2. **redo log buffer 占用的空间即将达到 `innodb_log_buffer_size` 一半的时候，后台线程会主动写盘**。注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache。
3. **并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘**。假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交，如果 `innodb_flush_log_at_trx_commit` 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘。

**如果把 `innodb_flush_log_at_trx_commit` 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次**，每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了。
通常我们说 **MySQL 的“双 1”配置**，指的就是 `sync_binlog` 和 `innodb_flush_log_at_trx_commit` 都设置成 1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。

### 1.5.3 组提交机制
你可能有一个疑问，这意味着我从 MySQL 看到的 TPS 是每秒两万的话，每秒就会写四万次磁盘。但是，我用工具测试出来，磁盘能力也就两万左右，怎么能实现两万的 TPS？解释这个问题，就要用到**组提交（group commit）** 机制了。

#### 1.5.3.1 redo log 组提交机制
我需要先和你介绍**日志逻辑序列号（log sequence number，LSN）** 的概念。LSN 是单调递增的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length。
LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log。

如图 3 所示，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。
![[image-82.png]]
从图中可以看到：
1. trx1 是第一个到达的，会被选为这组的 leader；
2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160；
3. trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘；
4. 这时候 trx2 和 trx3 就可以直接返回了。

一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。
但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 fsync 越晚调用，组员可能越多，节约 IOPS 的效果就越好。

#### 1.5.3.2 binlog 组提交机制
为了让一次 fsync 带的组员更多，MySQL 有一个很有趣的优化：拖时间。在介绍两阶段提交的时候，我曾经给你画了一个图，现在我把它截过来。
![[image-83.png]]
我把“写 binlog”当成一个动作。但实际上，写 binlog 是分成两步的：
1. 先把 binlog 从 binlog cache 中写到 page cache；
2. 调用 fsync 持久化。

MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了步骤 1 之后。也就是说，上面的图变成了这样：
![[image-84.png]]
这么一来，binlog 也可以组提交了。在执行第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了 cache，也是一起持久化的，这样也可以减少 IOPS 的消耗。

不过通常情况下第 3 步执行得会很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果那么好。
如果你想提升 binlog 组提交的效果，可以通过设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 来实现。
1. `binlog_group_commit_sync_delay` 参数，表示延迟多少微秒后才调用 fsync;
2. `binlog_group_commit_sync_no_delay_count` 参数，表示累积多少次以后才调用 fsync。
这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。所以，当 `binlog_group_commit_sync_delay` 设置为 0 的时候，`binlog_group_commit_sync_no_delay_count` 也无效了。
这个参数和 sync_binlog 的区别：
- sync_binlog = N：每个事务write后就响应客户端了。刷盘是N次事务后刷盘。N次事务之间宕机，数据丢失。
- binlog_group_commit_sync_no_delay_count=N： 必须等到N个cache后才能提交。换言之，会增加响应客户端的时间。但是一旦响应了，那么数据就一定持久化了。宕机的话，数据是不会丢失的。

之前有同学在评论区问到，WAL 机制是减少磁盘写，可是每次提交事务都要写 redo log 和 binlog，这磁盘读写次数也没变少呀？现在你就能理解了，WAL 机制主要得益于两个方面：
1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。

### 1.5.4 IO 瓶颈的参数优化
如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？针对这个问题，可以考虑以下三种方法：
1. 设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 参数，减少 binlog 的写盘次数。
2. 将 `sync_binlog` 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。
3. 将 `innodb_flush_log_at_trx_commit` 设置为 2。这样做的风险是，主机掉电的时候会丢数据。

# 2 主备机制
## 2.1 基本原理
### 2.1.1 主备切换流程
如图 1 所示就是基本的主备切换流程。
![[image-85.png]]

在状态 1 中，虽然节点 B 没有被直接访问，但是我依然建议你把节点 B（也就是备库）设置成只读（`readonly`）模式。这样做，有以下几个考虑：
1. 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
2. 防止切换逻辑有 bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用 `readonly` 状态，来判断节点的角色。

### 2.1.2 内部流程
你可能会问，我把备库设置成只读了，还怎么跟主库保持同步更新呢？这个问题，你不用担心。因为 `readonly` 设置对超级 (super) 权限用户是无效的，而用于同步更新的线程，就拥有超级权限。
接下来，我们再看看节点 A 到 B 这条线的内部流程是什么样的。图 2 中画出的就是一个 `update` 语句在节点 A 执行，然后同步到节点 B 的完整流程图。
![[image-86.png]]

备库 B 跟主库 A 之间维持了一个长连接。一个事务日志同步的完整过程是这样的：
1. 在备库 B 上通过 `change master` 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 1. 在备库 B 上执行 `start slave` 命令，这时候备库会启动两个线程，就是图中的 `io_thread` 和 `sql_thread`。其中 `io_thread` 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B。
4. 备库 B 拿到 binlog 后，写到本地文件，称为**中转日志（relay log）**。
5. `sql_thread` 读取中转日志，解析出日志里的命令，并执行。
后来由于多线程复制方案的引入，`sql_thread` 演化成为了多个线程，跟我们今天要介绍的原理没有直接关系，暂且不展开。

分析完了这个长连接的逻辑，我们再来看一个问题：binlog 里面到底是什么内容，为什么备库拿过去可以直接执行。

## 2.2 binlog 的三种格式对比
之前提到过 binlog 有两种格式，一种是 statement，一种是 row。可能你在其他资料上还会看到有第三种格式，叫作 mixed，其实它就是前两种格式的混合。为了便于描述 binlog 的这三种格式间的区别，我创建了一个表，并初始化几行数据。
```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```

如果要在表中删除一行数据的话，我们来看看这个 `delete` 语句的 binlog 是怎么记录的。注意，下面这个语句包含注释，如果你用 MySQL 客户端来做这个实验的话，要记得加 -c 参数，否则客户端会自动去掉注释：
```sql
mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
```

### 2.2.1 statement 模式
当 `binlog_format=statement` 时，binlog 里面记录的就是 SQL 语句的原文。你可以用 `show binlog events in 'master.000001';` 命令看 binlog 中的内容。
注意，首先要通过 `show variables like 'log_%';` 查看 `log_bin` 参数是否为 `ON`。否则，执行上面的命令是没有任何东西的。 同时，下面四个命令都可以查看 binlog 文件：
- `mysql> show binlog events;`：只查看第一个 binlog 文件的内容
- `mysql> show binlog events in 'mysql-bin.000001';`：查看指定 binlog 文件的内容
- `mysql> show binary logs;`：获取 binlog 文件列表
- `mysql> show master status;`：查看当前正在写入的 binlog 文件

![[image-87.png]]
- 第一行 `SET @@SESSION.GTID_NEXT='ANONYMOUS'` 你可以先忽略，后面文章我们会在介绍主备切换的时候再提到；
- 第二行是一个 `BEGIN`，跟第四行的 `commit` 对应，表示中间是一个事务；
- 第三行就是真实执行的语句了。可以看到，在真实执行的 `delete` 命令之前，还有一个 `use 'test'` 命令。这条命令不是我们主动执行的，而是 MySQL 根据当前要操作的表所在的数据库，自行添加的。这样做可以保证日志传到备库去执行的时候，不论当前的工作线程在哪个库里，都能够正确地更新到 `test` 库的表 `t`。
    `use 'test'` 命令之后的 `delete` 语句，就是我们输入的 SQL 原文了。可以看到，binlog “忠实”地记录了 SQL 命令，甚至连注释也一并记录了。
- 最后一行是一个 `COMMIT`。你可以看到里面写着 `xid=61`。XID 是用来联系 binlog 和 redo log 的。

### 2.2.2 row 模式
为了说明 statement 和 row 格式的区别，我们来看一下这条 `delete` 命令的执行效果图：
![[image-88.png]]
可以看到，运行这条 `delete` 命令产生了一个 warning，原因是当前 binlog 设置的是 statement 格式，并且语句中有 `limit`，所以这个命令可能是 unsafe 的。为什么这么说呢？这是因为 `delete` 带 `limit`，很可能会出现主备数据不一致的情况。比如上面这个例子：
1. 如果 `delete` 语句使用的是索引 `a`，那么会根据索引 `a` 找到第一个满足条件的行，也就是说删除的是 `a=4` 这一行；
2. 但如果使用的是索引 `t_modified`，那么删除的就是 `t_modified='2018-11-09'` 也就是 `a=5` 这一行。

如果我把 binlog 的格式改为 `binlog_format='row'`， 是不是就没有这个问题了呢？我们先来看看这时候 binog 中的内容吧。
![[image-89.png]]
row 格式的 binlog 里没有了 SQL 语句的原文，而是替换成了两个 event：`Table_map` 和 `Delete_rows`。
1. `Table_map event`，用于说明接下来要操作的表是 `test` 库的表 `t`;
2. `Delete_rows event`，用于定义删除的行为。

我们通过图 5 是看不到详细信息的，还需要借助 mysqlbinlog 工具，用下面这个命令解析和查看 binlog 中的内容。因为图 5 中的信息显示，这个事务的 binlog 是从 8900 这个位置开始的，所以可以用 `start-position` 参数来指定从这个位置的日志开始解析：
```bash
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```
![[image-90.png]]
从这个图中，我们可以看到以下几个信息：
- `server id 1`，表示这个事务是在 `server_id=1` 的这个库上执行的。
- 每个 event 都有 CRC32 的值，这是因为我把参数 `binlog_checksum` 设置成了 CRC32。
- `Table_map event` 跟在图 5 中看到的相同，显示了接下来要打开的表，map 到数字 226。现在我们这条 SQL 语句只操作了一张表，如果要操作多张表呢？每个表都有一个对应的 `Table_map event`、都会 map 到一个单独的数字，用于区分对不同表的操作。
- 我们在 mysqlbinlog 的命令中，使用了 -vv 参数是为了把内容都解析出来，所以从结果里面可以看到各个字段的值（比如，`@1=4`、 `@2=4` 这些值）。
- `binlog_row_image` 的默认配置是 `FULL`，因此 `Delete_event` 里面，包含了删掉的行的所有字段的值。如果把 `binlog_row_image` 设置为 `MINIMAL`，则只会记录必要的信息，在这个例子里，就是只会记录 `id=4` 这个信息。
- 最后的 Xid event，用于表示事务被正确地提交了。
你可以看到，当 `binlog_format` 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 `id`，这样 binlog 传到备库去的时候，就肯定会删除 `id=4` 的行，不会有主备删除不同行的问题。

### 2.2.3 为什么会有 mixed 格式的 binlog？
#### 2.2.3.1 mixed 优点
推论过程是这样的：
1. 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
2. 但 row 格式的缺点是，很占空间。比如你用一个 `delete` 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度。
3. 所以，MySQL 就取了个折中方案，也就是有了 mixed 格式的 binlog。mixed 格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。

比如我们这个例子，设置为 mixed 后，就会记录为 row 格式；而如果执行的语句去掉 `limit 1`，就会记录为 statement 格式。

#### 2.2.3.2 row格式的优点：恢复数据
现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。这么做的理由有很多，我来给你举一个可以直接看出来的好处：**恢复数据**。接下来，我们就分别从 `delete`、`insert` 和 `update` 这三种 SQL 语句的角度，来看看数据恢复的问题。
即使我执行的是 `delete` 语句，row 格式的 binlog 也会把被删掉的行的整行信息保存起来。所以，如果你在执行完一条 `delete` 语句以后，发现删错数据了，可以直接把 binlog 中记录的 `delete` 语句转成 `insert`，把被错删的数据插入回去就可以恢复了（但是这个也得基于 `binlog_row_image = 'FULL'` 才可以）。
如果执行的是 `update` 语句的话，binlog 里面会记录修改前整行的数据和修改后的整行数据。所以，如果你误执行了 `update` 语句的话，只需要把这个 event 前后的两行信息对调一下，再去数据库里面执行，就能恢复这个更新操作了。
其实，由 `delete`、`insert` 或者 `update` 语句导致的数据操作错误，需要恢复到操作之前状态的情况，也时有发生。MariaDB 的 [Flashback](https://mariadb.com/kb/en/library/flashback/) 工具就是基于上面介绍的原理来回滚数据的。

#### 2.2.3.3 执行上下文
虽然 mixed 格式的 binlog 现在已经用得不多了，但这里我还是要再借用一下 mixed 格式来说明一个问题，来看一下这条 SQL 语句：
```sql
mysql> insert into t values(10,10, now());
```
如果我们把 binlog 格式设置为 mixed，你觉得 MySQL 会把它记录为 row 格式还是 statement 格式呢？先不要着急说结果，我们一起来看一下这条语句执行的效果。
![[image-91.png]]
可以看到，MySQL 用的居然是 statement 格式。你一定会奇怪，如果这个 binlog 过了 1 分钟才传给备库的话，那主备的数据不就不一致了吗？接下来，我们再用 mysqlbinlog 工具来看看：
![[image-92.png]]
从图中的结果可以看到，原来 binlog 在记录 event 的时候，多记了一条命令：`SET TIMESTAMP=1546103491`。它用 `SET TIMESTAMP` 命令约定了接下来的 `now()` 函数的返回时间。因此，不论这个 binlog 是 1 分钟之后被备库执行，还是 3 天后用来恢复这个库的备份，这个 `insert` 语句插入的行，值都是固定的。也就是说，通过这条 `SET TIMESTAMP` 命令，MySQL 就确保了主备数据的一致性。

我之前看过有人在重放 binlog 数据的时候，是这么做的：用 mysqlbinlog 解析出日志，然后把里面的 `statement` 语句直接拷贝出来执行。你现在知道了，这个方法是有风险的。因为有些语句的执行结果是依赖于上下文命令的，直接执行的结果很可能是错误的。
所以，用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。类似下面的命令：
```sql
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```
这个命令的意思是，将 `master.000001` 文件里面从第 2738 字节到第 2973 字节中间这段内容解析出来，放到 MySQL 去执行。

## 2.3 双 M 结构的循环复制问题
实际生产上使用比较多的是双 M 结构，也就是图 9 所示的主备切换流程。
![[image-93.png]]
双 M 结构和 M-S 结构，其实区别只是多了一条线，即：**节点 A 和 B 之间总是互为主备关系。这样在切换的时候就不用再修改主备关系**。互为主备那就是刚开始的时候双方都要指定一下，然后后面就主推过去。

双 M 结构还有一个问题需要解决。业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（我建议你把参数 `log_slave_updates` 设置为 `on`，表示备库执行 relay log 后生成 binlog）。
如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。这个要怎么解决呢？
MySQL 在 binlog 中记录了这个命令第一次执行时所在实例的 server id。因此，我们可以用下面的逻辑，来解决两个节点间的循环复制的问题：
1. 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog；
3. 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

## 2.4 主备延迟
### 2.4.1 基础概念
与数据同步有关的时间点主要包括以下三个：
1. 主库 A 执行完成一个事务，写入 binlog，我们把这个时刻记为 T1;
2. 之后传给备库 B，我们把备库 B 接收完这个 binlog 的时刻记为 T2;
3. 备库 B 执行完成这个事务，我们把这个时刻记为 T3。

所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也就是 T3-T1。你可以在备库上执行 `show slave status` 命令，它的返回结果里面会显示 `seconds_behind_master`，用于表示当前备库延迟了多少秒。
`seconds_behind_master` 的计算方法是这样的：
1. 每个事务的 binlog 里面都有一个时间字段，用于记录主库上写入的时间；
2. 备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到 `seconds_behind_master`。

你可能会问，如果主备库机器的系统时间设置不一致，会不会导致主备延迟的值不准？其实不会的。因为，备库连接到主库的时候，会通过执行 `SELECT UNIX_TIMESTAMP()` 函数来获得当前主库的系统时间。如果这时候发现主库的系统时间与自己不一致，备库在执行 `seconds_behind_master` 计算的时候会自动扣掉这个差值。

需要说明的是，在网络正常的时候，日志从主库传给备库所需的时间是很短的，即 T2-T1 的值是非常小的。也就是说，**网络正常情况下，主备延迟的主要来源是备库接收完 binlog 和执行完这个事务之间的时间差**。
所以说，主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产 binlog 的速度要慢。

### 2.4.2 主备延迟的来源
1. 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。
2. 由于主库直接影响业务，大家使用起来会比较克制，反而忽视了备库的压力控制。结果就是，备库上的查询耗费了大量的 CPU 资源，影响了同步速度，造成主备延迟。
	1. 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
	2. 通过 binlog 输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力。
3. 大事务：因为主库上必须等事务执行完成才会写入 binlog，再传给备库。如果一个主库上的语句执行 10 分钟，那这个事务很可能就会导致从库延迟 10 分钟。
4. 造成主备延迟还有一个大方向的原因，就是备库的并行复制能力。后面会讲

## 2.5 主备切换策略
### 2.5.1 数据可靠性优先策略
从状态 1 到状态 2 切换的详细过程是这样的：
1. 判断备库 B 现在的 `seconds_behind_master`，如果小于某个值（比如 5 秒）继续下一步，否则持续重试这一步；
2. 把主库 A 改成只读状态，即把 `readonly` 设置为 `true`；
3. 判断备库 B 的 `seconds_behind_master` 的值，直到这个值变成 0 为止；
4. 把备库 B 改成可读写状态，也就是把 `readonly` `设置为` false；
5. 把业务请求切到备库 B。
这个切换流程，一般是由专门的 HA 系统来完成的，我们暂时称之为可靠性优先流程。

可以看到，这个切换流程中是有不可用时间的。因为在步骤 2 之后，主库 A 和备库 B 都处于 `readonly` 状态，也就是说这时系统处于不可写状态，直到步骤 5 完成后才能恢复。
在这个不可用状态中，比较耗费时间的是步骤 3，可能需要耗费好几秒的时间。这也是为什么需要在步骤 1 先做判断，确保 `seconds_behind_master` 的值足够小。

### 2.5.2 可用性优先策略
如果我强行把步骤 4、5 调整到最开始执行，也就是说不等主备数据同步，直接把连接切到备库 B，并且让备库 B 可以读写，那么系统几乎就没有不可用时间了。我们把这个切换流程，暂时称作可用性优先流程。这个切换流程的代价，就是可能出现数据不一致的情况。

接下来，我就和你分享一个可用性优先流程产生数据不一致的例子。假设有一个表 `t`：
```sql
mysql> CREATE TABLE `t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `c` int(11) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into t(c) values(1),(2),(3);
```
这个表定义了一个自增主键 `id`，初始化数据后，主库和备库上都是 3 行数据。接下来，业务人员要继续在表 `t` 上执行两条插入语句的命令，依次是：
```sql
insert into t(c) values(4);
insert into t(c) values(5);
```

假设，现在主库上其他的数据表有大量的更新，导致主备延迟达到 5 秒。在插入一条 `c=4` 的语句后，发起了主备切换。图 3 是可用性优先策略，且 `binlog_format=mixed` 时的切换流程和数据结果：
![[image-94.png]]
最后的结果就是，主库 A 和备库 B 上出现了两行不一致的数据。可以看到，这个数据不一致，是由可用性优先流程导致的。

那么，如果我还是用可用性优先策略，但设置 `binlog_format=row`，情况又会怎样呢？
因为 row 格式在记录 binlog 的时候，会记录新插入的行的所有字段值，所以最后只会有一行不一致。而且，两边的主备同步的应用线程会报错 duplicate key error 并停止。也就是说，这种情况下，备库 B 的 (5,4) 和主库 A 的 (5,5) 这两行数据，都不会被对方执行。
![[image-95.png]]

从上面的分析中，你可以看到一些结论：
1. 使用 row 格式的 binlog 时，数据不一致的问题更容易被发现。而使用 mixed 或者 statement 格式的 binlog 时，数据很可能悄悄地就不一致了。如果你过了很久才发现数据不一致的问题，很可能这时的数据不一致已经不可查，或者连带造成了更多的数据逻辑不一致。
2. 主备切换的可用性优先策略会导致数据不一致。因此，大多数情况下，我都建议你使用可靠性优先策略。毕竟对数据服务来说的话，数据的可靠性一般还是要优于可用性的。

## 2.6 主备切换具体实现
下图 A、A' 是主备，其他是从库：
备库和从库的概念是不同的，虽然二者都是只读的，但是从库对外提供服务，而备库只是为主库提供备份。
![[image-101.png]]

### 2.6.1 基于位点的主备切换
#### 2.6.1.1 change master
当我们把节点 B 设置成节点 A’的从库的时候，需要执行一条 `change master` 命令：
```sql
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
MASTER_LOG_FILE=$master_log_name 
MASTER_LOG_POS=$master_log_pos  
```
这条命令有这么 6 个参数：
- `MASTER_HOST`、`MASTER_PORT`、`MASTER_USER` 和 `MASTER_PASSWORD` 四个参数，分别代表了主库 A’的 IP、端口、用户名和密码。
- 最后两个参数 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 表示，要从主库的 master_log_name 文件的 master_log_pos 这个位置的日志继续同步。而这个位置就是我们所说的同步位点，也就是主库对应的文件名和日志偏移量。

#### 2.6.1.2 找同步位点
原来节点 B 是 A 的从库，本地记录的也是 A 的位点。但是相同的日志，A 的位点和 A’的位点是不同的。因此，**从库 B 要切换的时候，就需要先经过“找同步位点”这个逻辑**。这个位点很难精确取到，只能取一个大概位置。为什么这么说呢？
考虑到切换过程中不能丢数据，所以我们找位点的时候，总是要找一个“稍微往前”的（如果太靠后可能会丢了记录），然后再通过判断跳过那些在从库 B 上已经执行过的事务。

一种取同步位点的方法是这样的：
1. 等待新主库 A’把中转日志（relay log）全部同步完成；
2. 在 A’上执行 `show master status` 命令，得到当前 A’上最新的 File 和 Position；
3. 取原主库 A 故障的时刻 T；
4. 用 mysqlbinlog 工具解析 A’的 File，得到 T 时刻的位点。
```sql
mysqlbinlog File --stop-datetime=T --start-datetime=T
```
![[image-102.png]]
图中，end_log_pos 后面的值“123”，表示的就是 A’这个实例，在 T 时刻写入新的 binlog 的位置。然后，我们就可以把 123 这个值作为 `$master_log_pos` ，用在节点 B 的 `change master` 命令里。

#### 2.6.1.3 跳过错误
你可以设想有这么一种情况，假设在 T 这个时刻，主库 A 已经执行完成了一个 `insert` 语句插入了一行数据 R，并且已经将 binlog 传给了 A’和 B，然后在传完的瞬间主库 A 的主机就掉电了。那么，这时候系统的状态是这样的：
1. 在从库 B 上，由于同步了 binlog， R 这一行已经存在；
2. 在新主库 A’上， R 这一行也已经存在，日志是写在 123 这个位置之后的；
3. 我们在从库 B 上执行 `change master` 命令，指向 A’的 File 文件的 123 位置，就会把插入 R 这一行数据的 binlog 又同步到从库 B 去执行。

这时候，从库 B 的同步线程就会报告 Duplicate entry ‘id_of_R’ for key ‘PRIMARY’ 错误，提示出现了主键冲突，然后停止同步。所以，通常情况下，我们在切换任务的时候，要先主动跳过这些错误，有两种常用的方法：
1. 一种做法是，主动跳过一个事务。跳过命令的写法是：
```sql
set global sql_slave_skip_counter=1;
start slave;
```
因为切换过程中，可能会不止重复执行一个事务，所以我们需要在从库 B 刚开始接到新主库 A’时，持续观察，每次碰到这些错误就停下来，执行一次跳过命令，直到不再出现停下来的情况，以此来跳过可能涉及的所有事务。
2. 另外一种方式是，通过设置 `slave_skip_errors` 参数，直接设置跳过指定的错误。在执行主备切换时，有这么两类错误，是经常会遇到的：
- 1062 错误是插入数据时唯一键冲突；
- 1032 错误是删除数据时找不到行。

我们很清楚在主备切换过程中，直接跳过 1032 和 1062 这两类错误是无损的，所以才可以这么设置 `slave_skip_errors` 参数。等到主备间的同步关系建立完成，并稳定执行一段时间之后，我们还需要把这个参数设置为空，以免之后真的出现了主从数据不一致，也跳过了。

### 2.6.2 GTID
通过 `sql_slave_skip_counter` 跳过事务和通过 `slave_skip_errors` 忽略错误的方法，虽然都最终可以建立从库 B 和新主库 A’的主备关系，但这两种操作都很复杂，而且容易出错。所以，MySQL 5.6 版本引入了 GTID，彻底解决了这个困难。

#### 2.6.2.1 格式
GTID 的全称是 Global Transaction Identifier，也就是全局事务 ID，是一个事务在提交的时候生成的，是这个事务的唯一标识。它由两部分组成，格式是：GTID=server_uuid:gno
- `server_uuid` 是一个实例第一次启动时自动生成的，是一个全局唯一的值；
- `gno` 是一个整数，初始值是 1，每次**提交事务的时候分配给这个事务**，并加 1。

这里我需要和你说明一下，在 MySQL 的官方文档里，GTID 格式是这么定义的：GTID=source_id:transaction_id。
这里的 `source_id` 就是 `server_uuid`；而后面的这个 `transaction_id`，我觉得容易造成误导，所以我改成了 `gno`。因为，在 MySQL 里面我们说 `transaction_id` 就是指事务 id，事务 id 是在事务执行过程中分配的，如果这个事务回滚了，事务 id 也会递增，而 `gno` 是在事务提交的时候才会分配。
从效果上看，GTID 往往是连续的，因此我们用 `gno` 来表示更容易理解。

GTID 模式的启动也很简单，我们只需要在启动一个 MySQL 实例的时候，加上参数 `gtid_mode=on` 和 `enforce_gtid_consistency=on` 就可以了。

#### 2.6.2.2 生成方式
在 GTID 模式下，每个事务都会跟一个 GTID 一一对应。这个 GTID 有两种生成方式，而使用哪种方式取决于 `session` 变量 `gtid_next` 的值。
1. 如果 `gtid_next=automatic`，代表使用默认值。这时，MySQL 就会把 `server_uuid:gno` 分配给这个事务。
    - 记录 binlog 的时候，先记录一行 `SET @@SESSION.GTID_NEXT='server_uuid:gno';`
    - 把这个 GTID 加入本实例的 GTID 集合。
2. 如果 `gtid_next` 是一个指定的 GTID 的值，比如通过 `set gtid_next='current_gtid'` 指定为 `current_gtid`，那么就有两种可能：
	- 如果 `current_gtid` 已经存在于实例的 GTID 集合中，接下来执行的这个事务会直接被系统忽略；
	- 如果 `current_gtid` 没有存在于实例的 GTID 集合中，就将这个 `current_gtid` 分配给接下来要执行的事务，也就是说系统不需要给这个事务生成新的 GTID，因此 `gno` 也不用加 1。
		- 注意，一个 `current_gtid` 只能给一个事务使用。这个事务提交后，如果要执行下一个事务，就要执行 `set` 命令，把 `gtid_next` 设置成另外一个 `gtid` 或者 `automatic`。

这样，每个 MySQL 实例都维护了一个 GTID 集合，用来对应“这个实例执行过的所有事务”。

#### 2.6.2.3 例子
这样看上去不太容易理解，接下来我就用一个简单的例子，来和你说明 GTID 的基本用法。我们在实例 X 中创建一个表 `t`。
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into t values(1,1);
```
![[image-103.png]]

假设，现在这个实例 X 是另外一个实例 Y 的从库，并且此时在实例 Y 上执行了下面这条插入语句：
```sql
insert into t values(1,1);
```
并且，这条语句在实例 Y 上的 GTID 是 “aaaaaaaa-cccc-dddd-eeee-ffffffffffff:10”。
那么，实例 X 作为 Y 的从库，就要同步这个事务过来执行，显然会出现主键冲突，导致实例 X 的同步线程停止。这时，我们应该怎么处理呢？处理方法就是，你可以执行下面的这个语句序列：
```sql
set gtid_next='aaaaaaaa-cccc-dddd-eeee-ffffffffffff:10';
begin;
commit;
set gtid_next=automatic;
start slave;
```
其中，前三条语句的作用，是通过提交一个空事务，把这个 GTID 加到实例 X 的 GTID 集合中。

### 2.6.3 基于 GTID 的主备切换
在 GTID 模式下，备库 B 要设置为新主库 A’的从库的语法如下：
```sql
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
master_auto_position=1 
```
其中，`master_auto_position=1` 就表示这个主备关系使用的是 GTID 协议。可以看到，前面让我们头疼不已的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 参数，已经不需要指定了。

我们把现在这个时刻，实例 A’的 GTID 集合记为 set_a，实例 B 的 GTID 集合记为 set_b。接下来，我们就看看现在的主备切换逻辑。我们在实例 B 上执行 `start slave` 命令，取 binlog 的逻辑是这样的：
1. 实例 B 指定主库 A’，基于主备协议建立连接。
2. 实例 B 把 set_b 发给主库 A’。
3. 实例 A’算出 set_a 与 set_b 的差集，也就是所有存在于 set_a，但是不存在于 set_b 的 GTID 的集合，判断 A’本地是否包含了这个差集需要的所有 binlog 事务。
    - 如果不包含，表示 A’已经把实例 B 需要的 binlog 给删掉了，直接返回错误；
    - 如果确认全部包含，A’从自己的 binlog 文件里面，找出第一个不在 set_b 的事务，发给 B；
4. 之后就从这个事务开始，往后读文件，按顺序取 binlog 发给 B 去执行。

这个逻辑里面包含了一个设计思想：在基于 GTID 的主备关系里，系统认为只要建立主备关系，就必须保证主库发给备库的日志是完整的。因此，如果实例 B 需要的日志已经不存在，A’就拒绝把日志发给 B。
这跟基于位点的主备协议不同。基于位点的协议，是由备库决定的，备库指定哪个位点，主库就发哪个位点，不做日志的完整性判断。
严谨地说，主备切换不是不需要找位点了，而是找位点这个工作，在实例 A’内部就已经自动完成了。但由于这个工作是自动的，所以对 HA 系统的开发人员来说，非常友好。

### 2.6.4 GTID 和在线 DDL
[[TODO]]

## 2.7 备库并行复制
在官方的 5.6 版本之前，MySQL 只支持单线程复制，由此在主库并发高、TPS 高时就会出现严重的主备延迟问题。从单线程复制到最新版本的多线程复制，中间的演化经历了好几个版本。接下来，我就跟你说说 MySQL 多线程复制的演进过程。

### 2.7.1 多线程复制机制
其实说到底，所有的多线程复制机制，都是要把只有一个线程的 sql_thread，拆成多个线程，也就是都符合下面的这个模型：
![[image-96.png]]
coordinator 就是原来的 sql_thread, 不过现在它不再直接更新数据了，只负责读取中转日志和分发事务。真正更新日志的，变成了 worker 线程。而 work 线程的个数，就是由参数 `slave_parallel_workers` 决定的。
根据我的经验，把这个值设置为 8~16 之间最好（32 核物理机的情况），毕竟备库还有可能要提供读查询，不能把 CPU 都吃光了。

coordinator 在分发的时候，需要满足以下这两个基本要求：
1. 不能造成更新覆盖。这就要求更新同一行的两个事务，必须被分发到同一个 worker 中且保证顺序和主库一致。
2. 同一个事务不能被拆开，必须放到同一个 worker 中。

### 2.7.2 MySQL 5.5 版本的并行复制策略
官方 MySQL 5.5 版本是不支持并行复制的。但是，在 2012 年的时候，我自己服务的业务出现了严重的主备延迟，原因就是备库只有单线程复制。然后，我就先后写了两个版本的并行策略。
这里，我给你介绍一下这两个版本的并行策略，即按表分发策略和按行分发策略，以帮助你理解 MySQL 官方版本并行复制策略的迭代。

#### 2.7.2.1 按表分发策略
按表分发事务的基本思路是，如果两个事务更新不同的表，它们就可以并行。当然，如果有跨表的事务，还是要把两张表放在一起考虑的。如图 3 所示，就是按表分发的规则。
![[image-97.png]]
可以看到，每个 worker 线程对应一个 hash 表，用于保存当前正在这个 worker 的“执行队列”里的事务所涉及的表。hash 表的 key 是“库名. 表名”，value 是一个数字，表示队列中有多少个事务修改这个表。

假设在图中的情况下，coordinator 从中转日志中读入一个新事务 T，这个事务修改的行涉及到表 `t1` 和 `t3`。现在我们用事务 T 的分配流程，来看一下分配规则。
1. 由于事务 T 中涉及修改表 `t1`，而 worker_1 队列中有事务在修改表 `t1`，事务 T 和队列中的某个事务要修改同一个表的数据，这种情况我们说事务 T 和 worker_1 是冲突的。
2. 按照这个逻辑，顺序判断事务 T 和每个 worker 队列的冲突关系，会发现事务 T 跟 worker_2 也冲突。
3. **事务 T 跟多于一个 worker 冲突，coordinator 线程就进入等待**。
4. 每个 worker 继续执行，同时修改 hash_table。假设 hash_table_2 里面涉及到修改表 `t3` 的事务先执行完成，就会从 hash_table_2 中把 `db1.t3` 这一项去掉。
5. 这样 coordinator 会发现跟事务 T 冲突的 worker 只有 worker_1 了，因此就把它分配给 worker_1。
6. coordinator 继续读下一个中转日志，继续分配事务。

也就是说，每个事务在分发的时候，跟所有 worker 的冲突关系包括以下三种情况：
1. 如果跟所有 worker 都不冲突，coordinator 线程就会把这个事务分配给最空闲的 woker;
2. 如果跟多于一个 worker 冲突，coordinator 线程就进入等待状态，直到和这个事务存在冲突关系的 worker 只剩下 1 个；
3. 如果只跟一个 worker 冲突，coordinator 线程就会把这个事务分配给这个存在冲突关系的 worker。

#### 2.7.2.2 按行分发策略
**这个模式要求 binlog 格式必须是 row**。这时候，我们判断一个事务 T 和 worker 是否冲突，用的就规则就不是“修改同一个表”，而是“修改同一行”。

按行复制和按表复制的数据结构差不多，也是为每个 worker，分配一个 hash 表。只是要实现按行分发，这时候的 key，就必须是“库名 + 表名 + 唯一键的值”。
但是，这个“唯一键”只有主键 `id` 还是不够的，我们还需要考虑下面这种场景，表 `t1` 中除了主键，还有唯一索引 `a`：
```sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB;

insert into t1 values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5);
```
假设，接下来我们要在主库执行这两个事务：
![[image-98.png]]
如果它们被分到不同的 worker，就有可能 session B 的语句先执行。这时候 `id=1` 的行的 a 的值还是 1，就会报唯一键冲突。

在上面这个例子中，我要在表 `t1` 上执行 `update t1 set a=1 where id=2` 语句，在 binlog 里面记录了整行的数据修改前各个字段的值，和修改后各个字段的值。因此，coordinator 在解析这个语句的 binlog 的时候，这个事务的 hash 表就有三个项:
1. `key=hash_func(db1+t1+“PRIMARY”+2), value=2;` 这里 `value=2` 是因为修改前后的行 `id` 值不变，出现了两次。
2. `key=hash_func(db1+t1+“a”+2), value=1`，表示会影响到这个表 `a=2` 的行。
3. `key=hash_func(db1+t1+“a”+1), value=1`，表示会影响到这个表 `a=1` 的行。
可见，**相比于按表并行分发策略，按行并行策略在决定线程分发的时候，需要消耗更多的计算资源**。

这个方案有一些约束条件：
1. 要能够从 binlog 里面解析出表名、主键值和唯一索引的值。也就是说，主库的 binlog 格式必须是 row；
2. 表必须有主键；
3. 不能有外键。表上如果有外键，级联更新的行不会记录在 binlog 中，这样冲突检测就不准确。

如果是要操作很多行的大事务的话，按行分发的策略有两个问题：
1. 耗费内存。比如一个语句要删除 100 万行数据，这时候 hash 表就要记录 100 万个项。
2. 耗费 CPU。解析 binlog，然后计算 hash 值，对于大事务，这个成本还是很高的。

所以，我在实现这个策略的时候会设置一个阈值，单个事务如果超过设置的行数阈值（比如，如果单个事务更新的行数超过 10 万行），就暂时退化为单线程模式，退化过程的逻辑大概是这样的：
1. coordinator 暂时先 hold 住这个事务；
2. 等待所有 worker 都执行完成，变成空队列；
3. coordinator 直接执行这个事务；
4. 恢复并行模式。

### 2.7.3 MySQL 5.6 版本的并行复制策略
官方 MySQL5.6 版本，支持了并行复制，只是支持的粒度是按库并行。理解了上面介绍的按表分发策略和按行分发策略，你就理解了，用于决定分发策略的 hash 表里，key 就是数据库名。
如果你的主库上的表都放在同一个 DB 里面，这个策略就没有效果了；或者如果不同 DB 的热点不同，比如一个是业务逻辑库，一个是系统配置库，那也起不到并行的效果。

### 2.7.4 MariaDB 的并行复制策略
之前介绍了 redo log 组提交 (group commit) 优化， 而 MariaDB 的并行复制策略利用的就是这个特性：
1. 能够在同一组里提交的事务，一定不会修改同一行；因为事务是在需要的时候加锁，但是必须是事务提交了之后才会释放锁，如果两个事务修改的是同一行，那么这两个事务需要获取相同的行锁，由于两个事务都没有提交，由于锁的互斥性，是不可能让两个事务同时拥有的，故能够在同一个组提交的事务，一定不会修改同一行。
2. 主库上可以并行执行的事务，备库上也一定是可以并行执行的。

在实现上，MariaDB 是这么做的：
1. 在一组里面一起提交的事务，有一个相同的 `commit_id`，下一组就是 `commit_id+1`；
2. `commit_id` 直接写到 binlog 里面；
3. 传到备库应用的时候，相同 `commit_id` 的事务分发到多个 worker 执行；
4. 这一组全部执行完成后，coordinator 再去取下一批。

MariaDB 的这个策略，目标是“模拟主库的并行模式”。
但是，这个策略有一个问题，它并没有实现“真正的模拟主库并发度”这个目标。在主库上，一组事务在 commit 的时候，下一组事务是同时处于“执行中”状态的。
假设了三组事务在主库的执行情况，你可以看到在 trx1、trx2 和 trx3 提交的时候，trx4、trx5 和 trx6 是在执行的。这样，在第一组事务提交完成的时候，下一组事务很快就会进入 commit 状态。
![[image-99.png]]
而按照 MariaDB 的并行复制策略，备库上的执行效果如图 6 所示。
![[image-100.png]]
可以看到，在备库上执行的时候，要等第一组事务完全执行完成后，第二组事务才能开始执行，这样系统的吞吐量就不够。另外，这个方案很容易被大事务拖后腿。假设 trx2 是一个超大事务，那么在备库应用的时候，trx1 和 trx3 执行完成后，就只能等 trx2 完全执行完成，下一组才能开始执行。这段时间，只有一个 worker 线程在工作，是对资源的浪费。

### 2.7.5 MySQL 5.7 的并行复制策略
在 MariaDB 并行复制实现之后，官方的 MySQL5.7 版本也提供了类似的功能，由参数 `slave-parallel-type` 来控制并行复制策略：
1. 配置为 `DATABASE`，表示使用 MySQL 5.6 版本的按库并行策略；
2. 配置为 `LOGICAL_CLOCK`，表示的就是类似 MariaDB 的策略。不过，MySQL 5.7 这个策略，针对并行度做了优化。这个优化的思路也很有趣儿。

MySQL 5.7 并行复制策略的思想是：
1. 同时处于 prepare 状态的事务，在备库执行时是可以并行的；
2. 处于 prepare 状态的事务，与处于 commit 状态的事务之间，在备库执行时也是可以并行的。

之前讲 binlog 的组提交的时候，介绍过两个参数 。在 MySQL 5.7 的并行复制策略里，它们可以用来制造更多的“同时处于 prepare 阶段的事务”。这样就增加了备库复制的并行度。

### 2.7.6 MySQL 5.7.22 的并行复制策略
MySQL 增加了一个新的并行复制策略，基于 WRITESET 的并行复制。相应地，新增了一个参数 `binlog-transaction-dependency-tracking`，用来控制是否启用这个新策略。这个参数的可选值有以下三种。
1. COMMIT_ORDER，表示的就是前面介绍的，根据同时进入 prepare 和 commit 来判断是否可以并行的策略。
2. WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的 hash 值，组成集合 writeset。如果两个事务没有操作相同的行，也就是说它们的 writeset 没有交集，就可以并行。
3. WRITESET_SESSION，是在 WRITESET 的基础上多了一个约束，即在主库上同一个线程先后执行的两个事务，在备库执行的时候，要保证相同的先后顺序。

当然为了唯一标识，这个 hash 值是通过“库名 + 表名 + 索引名 + 值”计算出来的。如果一个表上除了有主键索引外，还有其他唯一索引，那么对于每个唯一索引，`insert` 语句对应的 writeset 就要多增加一个 hash 值。
MySQL 官方的这个实现还是有很大的优势：
1. writeset 是在主库生成后直接写入到 binlog 里面的，这样在备库执行的时候，不需要解析 binlog 内容（event 里的行数据），节省了很多计算量；
2. 不需要把整个事务的 binlog 都扫一遍才能决定分发到哪个 worker，更省内存；
3. 由于备库的分发策略不依赖于 binlog 内容，所以 binlog 是 statement 格式也是可以的。
当然，对于“表上没主键”和“外键约束”的场景，WRITESET 策略也是没法并行的，也会暂时退化为单线程模型。

官方 MySQL5.7 版本新增的备库并行策略，修改了 binlog 的内容，也就是说 binlog 协议并不是向上兼容的，在主备切换、版本升级的时候需要把这个因素也考虑进去。

---
# 3 引用