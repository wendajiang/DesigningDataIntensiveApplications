# 《Designing Data-Intensive Applications》翻译

> 2020.08.05 开始更新
>
> 源自阅读 [How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
>
> 提到在本书 8，9 章有着更进一步的论述于是找来本书
>
> 锻炼英文阅读能力，以及本书极高的价值
>
> PS. 本书已有中文版出版，并且开源版本有 __@Vonng__ 翻译完成的 [精翻版本](https://github.com/Vonng/ddia)

## 目录

### Part I 数据系统的基本论述

> 第一部分的四个章节阐述了适用于系统运行于单机或者集群系统的所有数据系统的基本观点
>
> 1. 第一章 介绍了整本书使用的术语和方法，并且阐述了所谓的可靠性、可扩展性和可维护性的实际含义，以及如何尝试实现这些目标
> 2. 第二章 从开发人员的角度比较了几种不同的数据模型和查询语言-对比几种数据库之间最明显的差别。我们将看到不通的模型如何适应不通的场景
> 3. 第三章 剖析数据库的引擎和在硬盘上的数据排列方式。从而发现不同的数据引擎对于不通的工作流做了各自的优化，针对不同的场景选择合适的数据引擎可以获得巨大的性能回报
> 4. 第四章 比较几种格式的数据编码方案（序列化），尤其检验它们在应用要求发生变更和模式需要随着时间适应的环境中的表现

Chapter 1 [可靠、可扩展和可维护的应用](https://github.com/wendajiang/DesigningDataIntensiveApplications/blob/master/part1.FoundationasOfDataSystems/chapter1.Reliabel_ScalabelAndMaintainableApplications.md)

Chapter 2 数据模型和查询语言

Chapter 3 [存储和检索](https://github.com/wendajiang/DesigningDataIntensiveApplications/blob/master/part1.FoundationasOfDataSystems/chapter3.Storage&Retrieval.md)

Chapter 4 编码和进化

### Part II 分布式数据

> 在本书的第一部分，我们讨论了存储于单机的数据系统的各个方面。现在，在第二部分，我们升级一下问题：如果多台计算机参与数据存储和检索会发生什么？
>
> 出于多重原因，您可能希望数据系统跨计算机分布：
>
> - 可扩展性
>   如果你的数据量，读写负载已经超过了单机处理的最高水平，可能会想将负载均衡到多台计算机
> - 容错性/高可用性
>   如果你的应用在即使一台计算机（或者几台计算机，网络，甚至整个数据中心）发生宕机后仍然能继续工作，则需要多台机器提供冗余，当一个系统宕机，可以有备份系统来接管
> - 延迟
>   如果你的用户遍布全球，则可能需要在全球各地部署服务，以便全球的用户在最近的数据中心来访问服务。这样避免了用户网络的延迟
>
> #### 扩展到更高的负载
>
> 如果你需要提升负载，最简单的方案就是购买一台性能更好的机器（有时称为垂直扩展）。可以在一个操作系统下将许多 CPU，许多 RAM 芯片和许多磁盘连接在一起，并且快速互连允许任何 CPU 访问内存或磁盘的任何部分。在这种共享内存体系结构中，所有组件都可以视为一台机器。
>
> 这种共享内存的方法的问题是成本增长太快：两倍 CPU，两倍 RAM，两倍硬盘容量的机器的价格远不止两倍。而且由于系统瓶颈，两倍硬件的机器可能并不能处理两倍的负载。
>
> 这种共享内存的方法只能提供有限的容错能力，即使是高端热拔插组件的计算机在地理位置这个维度，容错能力也是有限的。
>
> 另一种方案是共享硬盘存储，可以多台机器有独立的 CPU 和 RAM，但是共享硬盘存储（通过快速的网络）。这种体系结构用于某些数据仓库的场景，但是数据竞争和锁限制了这种方案的可扩展性。
>
> #### 无共享架构
>
> 相比之下，无共享架构（有时也称为水平扩展）广受欢迎。这种方案中，每台运行数据库的物理机或者虚拟机被称为节点（***node***）, 每个节点使用独立的 CPU，RAM 和硬盘。节点之间的协调都通过网络在软件层面进行。
>
> 无共享系统不需要任何特殊的硬件，所以你可以最具性价比的机器。还可以将机器分布在不同的地理位置，从而使得用户都拥有最近的服务器使用，并且可以避免整个数据中心丢失。通过上云虚拟机的方式，即使对于小型公司，也可以采用多区域分布式架构了。
>
> 在本书的这一部分，我们重点关注无共享架构，并不是因为对于每种场景都是最佳选择，而是因为，对于开发者来说，这里面的坑最多。如果你的数据分布在不同的节点，你就需要了解在分布式系统中的约束和其中的关于一致性/正确性的权衡等。
>
> 尽管无共享分布式架构具有很多优点，但是它通常会给应用带来额外的复杂性，并且有时会限制数据模型的表达能力。（PS（译者著）.redis 单机版和集群版对比，集群版的数据支持要少于单机版）在某些情况下，一个简单的单线程程序可能要比具有 100 个 CPU 核心的集群程序性能要好。另一方面，无共享架构可以表现的非常强大。接下来的章节将讨论一些数据分布中出现问题的细节。
>
> #### 复制 VS 分区
>
> 有两种常见的方式将数据分布到多个节点上：
>
> - 复制
>   将相同的数据的副本保存在多个不同的节点（可能处于不同的地理位置）。复制提供冗余：即使某些节点不可用，仍然可以从其他节点提供数据。复制还可以提高性能，我们将在第 5 章讨论复制
> - 分区
>   将大型数据库拆分为较小的子集称为分区，以便可以将不同分区分配给不同的节点（也称为分片）。我们将在第 6 章讨论分区
>
> 这是两种独立的机制，而且它们经常并存。
>
> 了解了这些概念之后，我们需要在分布式系统中进行艰难的权衡。在第 7 章中，我们讨论事务，这将帮助你理解数据系统中可能出错的地方以及如何处理。在第 8 章和第 9 章中，我们讨论分布式系统重要的局限性。
>
> 然后，在本书的第三部分，我们将讨论如何使用多重数据存储，并将其整合到一个巨大复杂的应用体系中。这种系统构建的需求通常被号称满足一切需求的供应商所忽略。实际上，

Chapter 5 复制

Chapter 6 分区

Chapter 7 事务

Chapter 8 分布式系统的麻烦

Chapter 9 一致性和正确性

### Patr III 派生数据

> 在本书的第一部分和第二部分，我们讨论了深入分布式数据库的方方面面，从数据在磁盘上的排列方式到一定出错概率下的分布式一致性的限制。然而，这些所有讨论的前提是应用中只有一个数据库。
>
> 实际情况是，数据系统通常是复杂的。在一个大型应用中，你通常需要以不同的方式处理数据，没有一个数据库可以应对所有的需要。因此应用需要使用不同的数据库来用作数据存储，索引，缓存，分析系统等，并实现可以将数据迁移的机制。
>
> 在本书的最后一部分，我们将讨论如何将不同类型的数据系统集成到一个一致的应用体系中。实际上，将不同的系统整合起来是应用程序中最重要的事情之一。
>
> #### 记录系统和派生数据系统
>
> 高层抽象来讲，存储和处理数据的系统可以被分为两个大类：
>
> - 记录系统
>   记录系统（也称为事实源），保存着数据的权威版本。当类似用户输入的新数据到来时，首先写入记录系统。如果存在另一个系统与记录系统存在差异，则以记录系统为准。
> - 派生数据系统
>   派生数据系统中的数据是从另一个系统中获取已有数据然后通过某种方式转换或者处理得到的。如果丢失了派生数据，可以从源数据重新计算得到。一个典型的例子是缓存：数据如果在缓存系统中存在可以直接获取，但是如果缓存中没有该数据，你需要从底层数据库中加载到缓存系统。非规范化的值，索引和视图也属于此类。在推荐系统中，预测性的摘要数据通常来自于日志系统。
>
> 从技术上来说，派生的数据是冗余的，这个角度来说，它们只是从现有数据的拷贝。然而，派生的数据通常对于查询的性能来说至关重要（通常会被归一化）。你可以通过一份数据源派生出几种不同的数据集，使得你可以从不同的角度来观察这份源数据。
>
> 并非所有的系统都对于记录系统还是派生数据系统有明确的区分，但是这是一个有用的区分，可以帮助阐明你系统的数据流：区分使得系统的各个部分有哪些输入，哪些输出，以及它们之间的依赖关系。
>
> 大多数数据库，存储引擎，查询语言既不是记录系统也不是数据派生系统。数据库只是一种工具，如何使用取决于你自己。记录系统与数据派生系统的区别不在于工具本身，而在于你如何在应用中使用它们。
>
> 通过清楚地了解到哪些数据是从其他数据中派生出来的，可以使得本来令人困惑的系统体系清晰起来。这一点将贯穿本书的生剩余部分。
>
> #### 章节概览
>
> 我们将在第 10 章讨论批处理数据流的系统，比如 MapReduce，并了解它们如何向我们提供了构建大型数据系统的工具和原理。在第 11 章，我们将采用这些思路来应用到数据流，这使得我们可以以较低的延迟来做到同样的事。第 12 章通过探讨如何使用这些工具来构建可靠，可扩展以及可维护的应用总结了本书。

Chapter 10 批处理

Chapter 11 流处理

Chpater 12 数据系统的未来

## 法律声明

译者纯粹出于**学习目的**与**个人兴趣**翻译本书，不追求任何经济利益。

译者保留对此版本译文的署名权，其他权利以原作者和出版社的主张为准。

本译文只供学习研究参考之用，不得公开传播发行或用于商业用途。有能力阅读英文书籍者请购买正版支持。
