+++
title = "part1.Fundations of Data Systems"
template = "ddia_section.html"
sort_by = "weight"
paginate_by = 100
weight = 0
+++

前四章讨论适用于所有数据系统的基本idea，无论是运行在单个机器还是集群之上：

1. [第一章](./chapter1-reliabel-scalabelandmaintainableapplications) 第一章 介绍了整本书使用的术语和方法，并且阐述了所谓的可靠性、可扩展性和可维护性的实际含义，以及如何尝试实现这些目标
2. [第二章](./chapter2-datamodel-querylanguages) 从开发者最关注的角度，比较了几种不同的数据模型和查询语言。我们可以看到为了适配不同的场景，数据模型的区别有多大
3. 
4. [第三章](./chapter3-storage-retrieval) 深入存储引擎内部，了解数据库的数据如何存储在硬盘上。不同的存储引擎是为了不同的工作负载类型优化的，选择适当的存储引擎可以带来巨大的性能提升
5. [第四章](./chapter4-encoding-evolution) 比较几种格式的数据编码方案（序列化），尤其检验它们在应用要求发生变更和模式需要随着时间适应的环境中的表现

稍后，[第二部分](../part2.DistributedData) 将讨论一些分布式数据系统的特定问题