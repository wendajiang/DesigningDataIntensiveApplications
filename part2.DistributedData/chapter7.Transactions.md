+++
template = "ddia_page.html"
date = "2021-02-02 18:39:18"
title = "chapter7.Transacitons"
weight = 7
+++

>> 有些作者表示由于性能或可用性问题，普通的两阶段提交开销太大。我们认为最好由应用开发者去优化事务滥用带来的性能问题，而不是不提供事务能力 -- James Corbett et al., Spanner: Google’s Globally-Distributed Database (2012)

在严格的数据系统中，会发生很多错误：
- 数据库软件或者硬件随时会出错（包括写操作到一半）
- 应用可能随时 crash （包括一系列操作没有完成时）
- 网络可能随时中断，客户端到数据库，数据库节点之间
- 多个客户端同时写入数据库，互相覆盖
- 客户端可能读到没有意义的数据，因为只更新了一部分
- 客户端之间的竞争可能导致神奇的bug

为了做一个可靠的系统，必须处理这些故障并且确保不会导致整个系统的灾难性故障。但是实现容错机制需要大量工作。需要仔细考虑所有可能出错的场景，并进行大量测试确保方案有效。

@todo 已看完，待重读的时候润色翻译

## Summary 
transaction 是一个可以将并发问题和软硬件错误对应用隐藏的抽象层。很大一类错误被简单的 transaction abort 消除，应用只需要重试一次即可。

本章中，我们看到 transaction 可以解决的很多具体问题实例。不是所有应用都对这些问题比较敏感： 一个访问数据模式很简单的应用，比如读写单条 record，可以完全不用 transaction。但是对于复杂访问模式而言，transaciton 可以大大降低应用开发可能的错误

如果没有 transaction，各种问题（进程 crash，网络中断，掉电，磁盘满，不可预测的并发问题等）导致数据不一致。比如， JSON 类型的部分更新很容易与源数据不一致。没有 transaction，对于数据库的复杂交互访问会很难定位影响。

本章中，我们深入探究了并发控制的话题。我们谈论了多种隔离 level，read committed, snapshot isolatio(也被称为 repeatable read)，和 serializable。我们通过讨论了下面的竞争条件来区分多种 level

- Dirty reads : 一个客户端读到了另一个客户端还没有 commited 的写。 read commited isolation level 或者更强的 level 消除了脏读
- Diry writes : 一个客户端覆盖了另一个客户端还没有 commited 的写。所有 transaction 的实现都会消除脏写
- Read Skew (nonrepeatable reads) : 一个客户端在不同的时间点看到数据不同。这个问题基本由 snapshot isolation 消除，使一次 transaction 一次读到的数据是一致的。通常使用 MVCC 实现
- Lost updates : 两个客户端并发执行 read-modify-write 过程，一个会覆盖掉另一个的写入，所有数据丢失了。有些 snapshot isolation 实现避免了这个问题，但是有些需要显式加锁 (SELECT FOR UPDATE)
- Write Skew : 一次 transaction 读一些数据，然后根据读到的信息做一些决策，写回数据库。但是，写入后，决策的前提条件数据实际已经不满足做决策时的数据条件。只有 serializable isolation 可以消除这个问题
- Phantom reads : 一次 transaction 读到了满足搜索条件的一些数据。另一个客户端的写会影响搜索的结果。 snapshot isolation 可以消除直接的 phantom read，但是 write skew 上下文中的 phantoms 需要特别处理，比如 index-range locks (next-key lock)

Weak isolation level 处理了一些问题，但是还留下一些给开发者来显式处理（比如使用显式锁。只有 serializable isolation 可以消除所有这些问题。我们讨论了三种实现 serializable isolation 的方式：

- Literally executing transactions in a serial order : 如果你可以使得每个 transaction 执行的飞起，然后单 CPU core 足够处理 transaction 的吞吐，这是简单有效的方法
- Two-phase locking : 过去多年这是标准实现，但是很多应用避免使用因为性能有点差
- Serializable snapshot isolation (SSI) : 比较新的算法，可以很大程度避免上述两种方法的缺点。使用乐观方法，允许 transaction 无锁执行。当 transaction 要 commit 的时候，检查是否 serializable 决定是否 abort

本章的这些例子使用了关系数据模型。但是正如提到过的 multi-object transactions 的需要，transaction 是一个有价值的数据库特性，与数据模型无关

本章，我们更多探讨数据库运行在单节点这样上下文的 idea 和算法。 Transactions 在分布式数据库中有新的困难与挑战，我们在接下来的两章中讨论