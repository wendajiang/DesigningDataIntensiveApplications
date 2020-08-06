# 《Designing Data-Intensive Applications》翻译

> 2020.08.05 开始更新
>
> 源自阅读[How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
>
> 提到在本书8，9章有着更进一步的论述于是找来本书记录

## 目录

### Part I 数据系统的基本论述

> 第一部分的四个章节阐述了适用于系统运行于单机或者集群系统的所有数据系统的基本观点
>
> 1. 第一章 介绍了整本书使用的术语和方法，并且阐述了所谓的可靠性、可扩展性和可维护性的实际含义，以及如何尝试实现这些目标
> 2. 第二章 从开发人员的角度比较了几种不同的数据模型和查询语言-对比几种数据库之间最明显的差别。我们将看到不通的模型如何适应不通的场景
> 3. 第三章 剖析数据库的引擎和在硬盘上的数据排列方式。从而发现不同的数据引擎对于不通的工作流做了各自的优化，针对不同的场景选择合适的数据引擎可以获得巨大的性能回报
> 4. 第四章 比较几种格式的数据编码方案（序列化），尤其检验它们在应用要求发生变更和模式需要随着时间适应的环境中的表现

Chapter 1 可靠、可扩展和可维护的应用

Chapter 2 数据模型和查询语言

Chapter 3 存储和检索

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
> 如果你需要提升负载，最简单的方案就是购买一台性能更好的机器（有时称为垂直扩展）。可以在一个操作系统下将许多CPU，许多RAM芯片和许多磁盘连接在一起，并且快速互连允许任何CPU访问内存或磁盘的任何部分。在这种共享内存体系结构中，所有组件都可以视为一台机器。
>
> 这种共享内存的方法的问题是成本增长太快：两倍CPU，两倍RAM，两倍硬盘容量的机器的价格远不止两倍。而且由于系统瓶颈，两倍硬件的机器可能并不能处理两倍的负载。
>
> 这种共享内存的方法只能提供有限的容错能力，即使是高端热拔插组件的计算机在地理位置这个维度，容错能力也是有限的。
>
> 另一种方案是共享硬盘存储，可以多台机器有独立的CPU和RAM，但是共享硬盘存储（通过快速的网络）。这种体系结构用于某些数据仓库的场景，但是数据竞争和锁限制了这种方案的可扩展性。
>
> #### 无共享架构
>
> 相比之下，无共享架构（有时也称为水平扩展）广受欢迎。这种方案中，每台运行数据库的物理机或者虚拟机被称为节点（***node***）,每个节点使用独立的CPU，RAM和硬盘。节点之间的协调都通过网络在软件层面进行。
>
> 无共享系统不需要任何特殊的硬件，所以你可以最具性价比的机器。还可以将机器分布在不同的地理位置，从而使得用户都拥有最近的服务器使用，并且可以避免整个数据中心丢失。通过上云虚拟机的方式，即使对于小型公司，也可以采用多区域分布式架构了。
>
> 在本书的这一部分，我们重点关注无共享架构，并不是因为对于每种场景都是最佳选择，而是因为，对于开发者来说，这里面的坑最多。如果你的数据分布在不同的节点，你就需要了解在分布式系统中的约束和其中的关于一致性/正确性的权衡等。
>
> 尽管无共享分布式架构具有很多优点，但是它通常会给应用带来额外的复杂性，并且有时会限制数据模型的表达能力。（PS（译者著）.redis单机版和集群版对比，集群版的数据支持要少于单机版）在某些情况下，一个简单的单线程程序可能要比具有100个CPU核心的集群程序性能要好。另一方面，无共享架构可以表现的非常强大。接下来的章节将讨论一些数据分布中出现问题的细节。
>
> #### 复制VS分区
>
> 有两种常见的方式将数据分布到多个节点上：
>
> - 复制
>   将相同的数据的副本保存在多个不同的节点（可能处于不同的地理位置）。复制提供冗余：即使某些节点不可用，仍然可以从其他节点提供数据。复制还可以提高性能，我们将在第5章讨论复制
> - 分区
>   将大型数据库拆分为较小的子集称为分区，以便可以将不同分区分配给不同的节点（也称为分片）。我们将在第6章讨论分区
>
> 这是两种独立的机制，而且它们经常并存。
>
> 了解了这些概念之后，我们需要在分布式系统中进行艰难的权衡。在第7章中，我们讨论事务，这将帮助你理解数据系统中可能出错的地方以及如何处理。在第8章和第9章中，我们讨论分布式系统重要的局限性。
>
> 然后，在本书的第三部分，我们将讨论如何使用多重数据存储，并将其整合到一个巨大复杂的应用中。

Chapter 5 复制

Chapter 6 分区

Chapter 7 事务

Chapter 8 分布式系统的麻烦

Chapter 9 一致性和正确性

### Patr III 派生数据

> 

Chapter 10 批处理

Chapter 11 流处理

Chpater 12 数据系统的未来