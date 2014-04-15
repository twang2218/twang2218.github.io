---
layout: post
category: readings
title: 《Hadoop Operations》读书笔记 - 2 - 第三章 MapReduce
tags : [hadoop, Hadoop Operations]
date: 2014/03/24
---
> **Chapter 3: MapReduce**
> Eric Sammer ["Hadoop Operations" - O'Reilly (2012)](http://shop.oreilly.com/product/0636920025085.do) ... (p25 ~ p39)


MapReduce，在这里实际上有两个含义，一个是一种分布式计算模型；另一个是某种特定实现，比如Apache Hadoop MapReduce。其设计目的是为了简化大规模、分布式、高容错性的数据处理应用的开发，目前MapReduce是首选方案。

在MapReduce中，将任务拆分成了两部分，Map 函数和 Reduce 函数，开发人员只需要关注这二者实现即可，而底层构架则负责如何并行化、如何调度任务在不同的节点上、监视和恢复各个任务。

这种方式和 Java EE 中的 Application Server 很像，框架负责在请求来了的时候调用适当的 servlets，开发人员只需要关注上层逻辑即可，而不必关心底层的 TCP Socket 如何编程、如何处理事件环、如何处理线程调度等。

MapReduce 起源于 Google 工程师在 2004 年发的论文《MapReduce: Simplified Data Processing on Large Clusters》，论文中详细描述了这种计算模型，以及在Google实现的细节。Hadoop MapReduce 是对该计算模型的开源实现。

MapReduce 的四个阶段
=====================

客户端提交 Job
---------------

客户端程序使用 MapReduce API 向集群提交任务，JobTracker 将负责接受这些任务。客户端提交任务的时候，可以是一个独立的计算机，也可以是节点中的任何一个。

框架将决定如何对数据进行拆分，拆成一个个 **input splits** ，以方便进行并行处理。在 Hadoop 中，负责这件事情的是 **InputFormat** ，并且默认提供了一部分常用的 `InputFormat`。

执行 Map 任务
--------------

每一个 input split 对应一个 **Map 任务** ，每一个 Map 任务会诸条对 input split 中的记录执行 **map函数** 。Map任务的数量可以多于当前节点可以并行处理的能力，多余的任务会在队列中等候调度。Map函数的输入是 **一个键值对** ，输出是 **0个或多个键值对** 。负责将 input split 拆分为一个个键值对的依旧是 InputFormat。

输出的键值对中，对每一个 **Key** 会分配一个 **Partition（分区）** ，进行分区的是 **Partitioner** 。Hadoop MapReduce 中默认的分区是 **Hash Partitioner** ，先对 Key 计算 Hash，然后对结果取 reducer 个数的余数。此时，这种分区只是逻辑上的，每个分区的键值对并没有汇聚到一起，依旧分散在各个运行 map 的节点上。

Shuffle 和 Sort
-----------------

在将键值对送给 reducer 之前，会先进行 shuffle 和 sort，这是为了保证 reducer 的输入满足下面几个特性：

- 如果 reducer 看到了一个 key，它就可以看到该 key 对应的所有的 value。
- 一个 key 的键值对只可能会送到一个 reducer。
- 每一个 reducer 看到 key 时，要确保 key 是有序的

shuffle 和 sort 实际上是由 reducer 来执行的。

- 首先，reducer 会联系每一个 TT，去取得属于自己 **partition** 的键值对。
- 由于 mapper 输出的中间过程数据可能会很大，因此，为了避免当所有 map 任务执行完毕的时候，突然间所有 reducer 开始复制数据，导致出现激增的网络流量的情况，当中间结果一旦生成，就允许 reducer 开始复制数据了。中间这个时间差是可以通过 `mapred.reduce.slowstart.completed.maps` 来调整的。
- 当 reducer 收到所有数据后，所面对的是一堆无序的、分为不同分区的键值对，每个分区会按照 key 排序。随后， reducer 会对各个分区进行 **merge sort**。这样进行可以很节约内存。
- 当 Reducer 面对按 key 排序好的键值对时，就很容易顺序操作，将每一个键所对应所有值全部送入 reduce 函数。

执行 Reduce 任务
-----------------

每一个 reducer 的输出都送到 **单独** 的文件中去，从而很大程度上避免了不同 reducer 之间协调操作同一个文件的复杂性。

输出文件格式由 **OutputFormat** 决定。默认情况下，输出文件名会以 **part-xxxxx** 的形式存在，xxxxx是这个 reducer 在这个job 中的编号，开始于 **0** 。


MapReduce 的局限性
===================

- MapReduce 是 **批处理系统** ：换句话说，它假定于任务会 **以分钟甚至小时为单位** 执行。它适合于完整的扫描，而不适合传统OLTP那种需要低延时、随机访问模式的系统。
- MapReduce 是非常简单的：简单是好事，同时也是坏事。很多时候开发人员会感到这种模型的局限性。同样，由于模型的简单，导致 MapReduce 并不是最高效的计算模型；
- 相对于 SQL 这类高级语言而言，MapReduce 是非常 **低级** 的语言。可以称其为 **分布式计算的汇编语言** 。一般会在其上构建更高级的语言以方便使用。
- 很多算法无法并行化处理。也许其中部分算法可以被某种方式在 MapReduce 中运行，不过显然不是高效的实现。

Hadoop MapReduce 的简介
========================

Hadoop MapReduce 是对 MapReduce 计算模型的开源实现。如 Google 中的 GFS 与 MapReduce 结合一样，Hadoop 的HDFS和MapReduce的结合是非常强大的。在 MapReduce 调度 Job 时，由于可以从 NameNode 中得到数据的位置，因此可以选择最合适的 TaskTracker 来执行对应的任务，从而使得 map 最大可能的从本地读取数据，因此大大的减少了网络流量，这在处理海量数据时尤为重要。

Hadoop MapReduce 像其它分布式计算系统一样，它是一个框架，用户提交 Job 到这个框架中运行。客户端程序通过 Hadoop API 提交 job。可以是同步的，既在任务结束前，程序不退出；也可以是异步的，既提交任务后，客户端程序就退出，然后可以通过查询 JobTracker 任务执行情况。集群中的 TaskTracker 会启动一个独立的进程来执行用户提交的代码，虽然这引入了额外的资源占用，但是很好的隔离了故障，防止客户端提交的代码破坏集群的运行。此外，由于 MapReduce 的设计目标是针对批处理的，因此这种额外的资源占用，并不是很严重的影响。


两个主要的服务进程
===================

JobTracker
-----------

JobTracker 是主服务，负责从客户端承接 Job、在集群中调度任务、检查节点健康状态、任务执行状态等。一个 Hadoop MapReduce 集群中只有一个 JobTracker，如果这个 JobTracker 挂了，所有正在运行的 Job 就都挂了。客户端、TaskTracker 和 JobTracker 之间的通讯使用的是 RPC。

同 DN 向 NN 发送心跳健康报告一样， TT 也会向 JT 发送心跳报告。心跳报告中，包括 TT 中总的 map 和 reduce slots 的数量、以及占用的数量、还有当前运行 Tasks 的详细信息。如果 JT 一段时间没有收到某个 TT 的心跳，则假定其死亡。JT 使用线程池来处理心跳，TT 则是并行发送心跳报告。

当一个 job 被提交给 JT 时，所有其 tasks 信息都存储于JT 的内存中，并在TT发来心跳时被更新，因此这是某种程度上的实时 task 进展报告。当 Job 结束后，这些信息依旧会在内存中保留一段时间(可配置)，或执行过的 Job 达到一定数量。

对于活跃的集群，拥有很多 Job，而且每个也拥有很多 tasks 的，会在 JT 上消耗大量的内存。但是，除非真的运行那些 Job，无法事先评估 JT 上面可能的内存占用情况。 **因此，监视 JT 内存使用情况是非常关键的。**

JT 提供了一个 Web 界面，虽然比较丑，但是信息很丰富。包含了所有当前的Job信息、tasks 信息、各种统计数据以及task级别的日志。维护 Hadoop 集群的人可能会经常检查这个页面。

分配合适的 TT 去运行适当的 Task，这事情叫做task scheduling (任务调度)。第7章将着重讲解资源调度问题。

TaskTracker
------------

TaskTracker 负责从 JT 那里接任务，在本地执行用户代码，然后报告进度回 JT。集群中，每一个节点最多执行一个 TT。由于 TT 和 DN 往往在一个节点，因此该节点同时是 **计算节点** 和 **存储节点** 。

每一个 TT 会明确配置自己可以分别并行处理 map 或 reduce 的 tasks 的数量，这个可执行称为 **slots** 。因此 Hadoop MapReduce 中的并行是两个级别的，首先是 **集群级别** 的，一个 job 会在多个节点上并行运行；另一个级别是 **进程级别** 的，一个节点内会并行运行多个 tasks。

TT 接到 task 后，会对 task 进行一次 attempt，就是一次执行。之所以这里称为 attempt ，是因为这有可能失败。TT 和 task attempt 之间的通讯是通过本地回环的 RPC，称为 umbilical protocol。

TT 会将 task 中间结果存入用户指定的本地目录。一般中间结果都会很大，或者很多 job 在运行，无法保存于内存中。

TT 也有 Web 界面，不过一般很少会需要直接去 TT 上看，大部分信息都在 JT 页面上了，而且 JT 页面上会有指向 TT 的链接。


当事情出错了
============

MapReduce 在容错性上设计的非常好。在大型的 Hadoop 集群中，如果有 2% ~ 5% 的节点由于某种故障，特别是硬盘故障，这并不罕见。此外除了硬件故障外，还可能是用户提交的 job 导致了故障、网络故障、甚至仅仅是数据故障。

task 失败
-----------

导致 task 失败的三种情况：

- 抛出了异常
- 退出代码非0
- 很长时间 task 都没有向 TT 报告进展

失败发生后：

- 当 TT 检测到失败，TT 会在 **下一个心跳** 时报告给 JT。
- JT 会标记一下这个 task 失败，如果还有更多的重试机会（默认是4次），则会重新调度这个 task；
- 重新调度的 task 可能在 **同一个** 机器，也可以在 **不同** 的机器，取决于其资源情况；
- 如果一个 Job 的多个 task 都在某一个 TT 上失败了，这显然不正常。因此这个 TT 会被 JT 加入 **该 Job 的黑名单** ，以后该 Job 的任务都不会发给该 TT；
- 如果不同 Job 的任务都在某个 TT 上失败了，那么这个 TT 的问题就比较大了，会被 JT 加入到 **全局的黑名单** 上持续 **24小时** ，这样任何 Job 都不会分配给该 TT。

TT 或者 节点故障
-----------------

JT 如果一段时间内没有收到 TT 的心跳报告，那么就会假定这个 TT 挂了。所有分配到该 TT 的任务，会重新进入调度，调整到别的 TT 上去运行。客户端对此将 **毫无感觉** ，最多只会感觉到在某个位置，任务进度突然变慢了。

JT 故障
--------

JT 故障最简单了，别想了，一旦 JT 故障了，所有正在运行的 Job 就全都挂了。目前 **JT 是单点故障** ，是 Hadoop MapReduce 的局限之一。

HDFS 故障
----------

HDFS 故障，就相当于传统关系性数据库面对文件系统故障一样，不是很好的事情。最开始如果只是一个节点故障，TT 会像前一章错误处理的流程操作。除非全部的节点针对该Block 都出错了，否则这是一个可恢复的情况，程序运行不会受到影响。

如果不幸整个 HDFS 都挂了，比如NN也挂了，MapReduce 会经过一系列联系和重试，然后最终由于均失败，最后也只能回报客户端失败。


YARN
=====

在 Yahoo! 部署 Hammer 集群的时候，这大约是一个 4000 节点的集群，他们发现一个单独的 JT 很难处理这么大规模的集群，并且单点故障也是很悲催的。于是他们提出了一个新的构架叫做 YARN (Yet Another Resource Negotiator)。

YARN 将传统的 JobTracker 拆分为两部分： **负责资源分配** 部分成为了新的服务，叫做 **ResourceManager** ；每一个 Job 都是一个 application（应用），每一个Job 都会拥有一个 **ApplicationMaster** 存在，这个相当于 JobTracker **负责 Job 调度** 的部分。

YARN 与传统集中式 JobTracker 非常不同，将不同的 Job 之间 **完全隔离** 开来。换句话说，如果某一个 Job 的 ApplicatoinMaster 出现故障，不会影响其它 Job 的运行。因此，相当于是有很 **多个 JobTracker** 在集群中运行。

集群节点中，替代 TaskTracker 服务称为 NodeManager。TT 是明确主要负责处理 MapReduce 任务的，而 NM 则支持更广泛一些， **不局限于 MapReduce** 。NM 会在称为 **ApplicationContainer** 中运行任何类型的进程。比如，MapReduce的应用，在NM中可能会同时运行 ApplicationMaster 和具体的一个个 map 与 reduce 任务。

由于 YARN 可以运行任何应用程序，从而导致 YARN 更加计算模型无关化，不再局限于 MapReduce。Hadoop 社区中已经开始考虑其它可以运用 Hadoop 构架的计算模型，比如图形学计算、或者大规模并行计算的模型。


