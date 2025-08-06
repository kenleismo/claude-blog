--- 
layout: ../../layouts/BlogPost.astro
title: "KMemStoreDB undo 日志与事务可见性"
description: "介绍 KMemStoreDB 内存引擎 undo 日志实现与事务可见性判断"
publishDate: "2025-07-12"
tags: ["数据库内核原理", "事务", "undo"]
---

本文介绍我们在 KMemStoreDB 内存引擎中实现的 undo 日志以及基于 undo 日志的事务可见性判断。

## 事务可见性判断

事务开始时获取开始时间戳 sts，提交时获取提交时间戳 cts。sts 和 cts 并非真正的时间戳，而是标识先后顺序的 uint64 类型整数，并且只有低 63 位有效，最高位始终为 0。对于活跃事务，还未提交，所以没有 cts，但其 cts 被我们用来表示事务 id，事务 id 最高位为 1。也就是说，一个 uint64 类型的整数被分为了两个部分，最高位为状态标志，低63位才是时间戳或事务id，如下所示：
```
uint64 值 = [1位状态标志][63位时间戳或事务ID]
```

这样设计能够保证事务 id（最高位为1）永远大于 sts 和 cts，可以更方便地判断事务的可见性：如果一个事务 a 的 sts 大于另外一个事务 b 的 cts，即事务 a 在事务b 后发生，那么 a 能够看到 b 的修改，
判断的逻辑就是：`a.sts > b.cts`，如果 b 尚未提交，那么 a.sts 必然小于 b.cts，所以 a 也就无法看到 b 的修改。


事务可见性核心逻辑如下所示：

```c++
auto canSee = [ & ](uint64 next_undo_ts) -> bool {
    return (next_undo_ts == txn_cts) || (next_undo_ts < txn_sts);
};
```

这段代码用于遍历 undo 列表，找到第一个本事务可见的版本，是 KMemStoreDB MVCC 核心，含义如下：
1. 本事务的修改对本事务可见
2. 其他事务修改的版本，如果其提交时间戳（cts）小于当前事务的开始时间戳（sts），即发生在当前事务之前则可见，否则不可见


## undo 版本链

KMemStoreDB 内部维护数据的历史版本，具体而言是 undo 链，下面我们从事务流程角度，一起看看 undo 的设计与实现。

### Undo日志设计

undo 日志目前包含：
1. row_id: undo log 对应的数据行
2. m_type：undo 日志的类型，什么操作产生的 undo 日志
3. m_ts：undo 的事务 id，也就是 cts
4. older_ts_：上一个操作日志对应的事务号
5. older_undo_entry，上一个操作的undo日志，用于产生版本链

undo 日志类型有很多，我们主要关心：
- INSERT：对应插入操作
- DELETE：对应删除操作
- SELECT_FOR_UPDATE：对应有锁读
- INPLACE_UPDATE：对应原地更新


### 事务流程

#### 开启事务

KMemStoreDB 事务的开启是在第一次 lock_tables 阶段。

开启事务会创建事务上下文，这里是 `DMLTransactionContext`，并且获取全局自增的 sts，cts 在 sts 基础上将最高位置 1。

```c++
 cts = std::make_shared<std::atomic<cid_t>>(sts | (1ULL << 63));
```

代码流程如下：
```c++
ha_external_lock -> external_lock -> 创建 DMLTransactionContext
                                  `-> 开启事务 -> 获取 bucket id
                                             |-> 添加到活跃事务列表
                                             |-> 获取 sts
                                             `-> cts：将 sts 的最高位置1
```

全局事务管理器中维护活跃事务列表和已完成的事务列表：
```c++
  std::array<std::unique_ptr<TransactionSTSWithLock>, TXN_MAP_BUCKETS> active_txn_map_array_;
  std::array<std::unique_ptr<TransactionCTSWithLock>, TXN_MAP_BUCKETS> finished_txn_map_array_;
```

活跃事务列表和已完成的事务列表都是大小固定为 64 的哈希桶，开启事务时，会根据事务的 bucket id 找到对应的桶。
桶的实现是带锁的队列，每个事务在获取 sts 时需要先对桶加锁，然后获取全局自增的 id，再将 sts push 到槽中。
也就是说，每个桶按序保存 sts。

通过分桶，实现细粒度锁操作，提升获取 sts 的性能。


#### 写入

插入数据流程如下：
```c++
ha_write_row -> write_row -> km_row_insert -> InsertRow -> InsertTuple
                                                        |-> PushInsert
                                                        `-> InsertUndoEntryToFront：将undo log存入mapping table
```

InsertTuple 负责将数据插入行存页，如果行存页空间不足，会尝试获取新的行存页。当数据插入成功以后，会为 Insert 操作生成一个 UndoLogEntry，UndoLogEntry 中保存插入的 row_id，事务id（以 cts 形式）等信息。

生成的 undo 日志，会保存在两个地方：
1. 事务上下文中
2. mapping table 中

事务上下文是线程私有的，存储某个事务的信息。事务的操作保存在事务上下文中，很好理解。下面介绍一下 mapping table。

mapping table 保存数据与版本链的映射关系，每个 row page 对应一个 mapping table，当我们往 row page 中插入一行时，也会往 mapping table 中插入一行，mapping table 保存 row id 和它对应的 undo 日志链。

核心数据结构：
```c++
vector<array<UndoLogEntry*, 128>>
```

这个数据结构说明了 mapping table 的设计原理：

mapping table采用了等高的桶，每个桶可以保存 128 个 undo 指针，桶的总数是不固定的（vector）。
mapping table 的设计是保存每个 row id 的 undo 指针，row id 是连续的，所以你可以理解为第一个桶保存 `1~128` 行，第二个桶保存 `129~256`，依次类推。

采用这种预分配空间的策略，访问 mapping table 可以不用加锁，能够提升性能。当然在 vector 扩容时，目前是需要加锁的。


#### 更新

更新数据形成版本链，update 首先会执行 select for update，然后再执行 update。

当执行 select for update 读取某一行时，首先为这一行生成一个 undo 日志，并将这个 undo 日志挂到版本链上，同时也会将这个 undo 保存到事务上下文中。

update 更新会找到刚刚 select for update 插入的 undo 日志，然后将 undo 日志的类型由 SELECT_FOR_UPDATE 更新为 INPLACE_UPDATE。


更新数据的流程如下：
```c++
// 待补充
```

#### 删除