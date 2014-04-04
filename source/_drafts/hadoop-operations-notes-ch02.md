---
layout: post
category: readings
title: 《Hadoop Operations》读书笔记 - 1
tags : [hadoop]
---

{% blockquote Eric Sammer http://shop.oreilly.com/product/0636920025085.do "Hadoop Operations" - O'Reilly (2012) ... (p7 ~ p24)  %}{% endblockquote %}

第二章 HDFS
=================

传统存储是 SAN 或者 NAS，提供了集中化、低延时的块存储或者文件系统，以支持TB级数据。在面对关系型数据库之类的服务时，这是很好的选择。但是面对上万台计算机同时提取几百TB的数据时，这种集中型存储就难以胜任了。

HDFS的设计目标
---------------

- 存储上百万的大文件，每个文件都大于几十TB的数量级；
- 使用普通服务器，横向扩展，不必使用RAID；
- 针对大规模、流式读写进行优化，而不考虑低延时或者小文件。批处理重要性优于即时响应；
- 对机器、硬盘错误可以从容应对。

HDFS 和传统文件系统异同
-------------------------

相似处：

- 使用块儿保存文件
- metadata保存了文件名、所用的块儿、目录结构、权限等。

不同处：

- 没有使用内核驱动实现，是userspace的文件系统
- 分布式文件系统：metadata存在NameNode上，Block数据存在DataNode上。均是构建于现有文件系统之上的文件系统。
- 本地文件系统Block大小是4KB、8KB这个级别；而为了应对大规模的数据文件，HDFS的Block大小，一般是64MB(1.x默认), 128MB(2.x默认)，甚至管理员会调整为256MB以及1GB也不罕见。Block越大，意味着磁盘寻址时间越少，因为更多的将是大块的顺序读写。

HDFS会以Block为单位保存多份副本（默认是3份）。更多的副本除了更好的容错性，也会因为应用可以获取最近的副本的数据而取得更好的性能。

HDFS API 类似于 POSIX API，提供了文件IO操作，因此用户不需要关心底层的分布式、以及副本问题。

三个服务
-----------

### DataNode

HDFS的DN不需要RAID，只需要JBOD(Just a Bunch of Disks 就一堆磁盘)。对于 Hadoop 的应用特征而言，一大堆独立的硬盘要比RAID性能好。

DN启动后，每个小时会向NN发送Block Report（包括DN刚启动的时候），其中包含了该节点所存储的所有Block列表。这样做的好处是，当DN的IP或者主机名发生改变后，不影响其所存储的数据。另外一种常见的问题是当一个DN的主板类硬件损坏后，管理员只需要把硬盘拔下来，插在一个新的计算机上，启动DN，数据并不会丢失。

### NameNode

NameNode 所记录的所有HDFS文件系统元数据全部存储于内存，以方便快速遍历。因此在1.x中NN的内存大小决定了集群的规模。粗略估计的话，1百万Block(或1百万空文件)需要1GB内存。

### SecondaryNameNode


SecondaryNameNode 只是负责打理日志合并等事物，并不是热备，不过在必须时，NN可以从SNN中回复最近的元数据。


HDFS 读文件
------------

假设一个客户端，比如Java Jar程序，需要一份 `/user/esammer/foo.txt`。

- 首先，客户端中会指定NN的位置，或者是配置文件，或者是代码中直接指定；
- 客户端会先联系 NN，告知其想读取该文件
- 对于NN来说，首先需要验证对方身份。验证分两种，一种是相信客户端，客户端说自己是谁就是谁，比如默认会直接取当前系统的用户名；另一种则是使用Kerberos认证。
- 然后NN需要确定该用户是否拥有适当的权限访问该文件
- 如果文件存在，并且权限允许，则NN向客户端返回第一个Block的ID，以及一个保存有该Block的DN列表，列表由据客户端距离排序，最近的放在最上面。距离由所定义的拓扑结构来决定：同一个机器 < 同一个机架 < 不同机架
- 收到了Block ID和DN列表后，客户端会尝试联系第一个DN读取该Block。读完之后，会向NN请求下一个Block，以此类推，直到读完所有 Block了。
- 如果客户端联系DN的过程中发生了错误，DN不可用，客户端则会尝试列表中下一个DN，直至所有DN均无法访问，则读操作失败，客户端产生异常


HDFS 写文件
------------

HDFS 写文件操作比读文件略为复杂。

- 客户端会先使用 Hadoop `FileSystem` API 打开一个命名文件用于写入。这个请求会发送给 NameNode。
- NN 接受请求，验证用户有相应权限后，开始在元数据中添加对应文件项（此时不分配Block），并回应客户端说文件已创建/打开成功；
- 如果NN回应，客户端会打开一个流供客户端写入数据；
- 当客户端写入数据时，并不直接发送出去，而是会先在本地以一段段数据包(既不是TCP包，也不是HDFS Block)的形式于内存中压入队列；
- 一个独立的线程会开始消化这个队列中的包。这个独立的线程，会向NN发请求，宣称需要一个数据块以写入数据。
- NN接到请求后，按照剩余空间、拓扑、副本数（默认3份）等综合考虑后，会产生一个将要建立数据块的DN列表返回给客户端
- 客户端收到列表后，会连接到列表中第一个DN，以写入数据；注意，第一个DN会去连接列表中第二DN，以将客户端写入的副本传递给第二个DN；以此类推，一直到最后一个DN收到连接写入数据为止。这被称为 Replication Pipeline，复制流水线。
- 任何一个DN成功写入数据后，会向流水线中上一级回报写入成功。
- 当客户端收到写入成功后，说明所有副本都成功写入，客户端则会去写下一个数据包，直至需要下一个数据块为止。
- 当需要下一个数据块时，客户端会联系NN重新分配一个Block，并建立一个新的复制流水线，直至所有数据写入完毕，客户端会close文件，所有数据都会写入磁盘。
- 当流水线中间一个节点出现问题时，会向流水线上游报告错误，整个流水线会立刻关闭。并且分配一个新的Block ID，对剩余健康节点建立一个新的流水线，将之前的数据包，重新写入。分配新的ID的目的，是防止坏的DN又再次上线，为该ID的Block提供了错误的数据。
- 可能会有更多流水线中的DN出现错误，重建流水线的过程会持续，直至达到最低副本书（默认是1份），如果无法保证最低副本数，客户端则产生异常报错。
- 如果由于错误导致了副本数量达不到指定副本数量，NN会被告知，从而开始安排针对 Under-replicated 块进行新的复制任务；


元数据
-------

元数据保存于 NN 的本地文件系统中。主要由两部分组成：`fsimage` 和 `edits`（或称为 editlogs）。和数据库一样，fsimage 是元数据完整快照；而 edits 则是增量修改日志。一般是 WAL （Write Ahead Log）的形式。二者都会处于内存，并且本地有一份固化版本。当 NN 启动的时候，会从文件系统加载 fsimage，然后再加载 edits 日志，并回放 edits，从而得到最新的元数据信息。

也因此，随着时间增加，edits会越来越大，NN 启动会需要很长一段时间，甚至由于超过 NN 的资源而导致启动失败。这时 SNN 的作用就体现了，由于SNN的作用是合并元数据，因此当SNN存在的时候，可以使用更新的快照，而使得启动速度会加快。

SNN 与 NN 交流合并元数据的过程
------------------------------

- SNN 通知 NN 开始合并日志，NN 从即刻起将使用 edits.new 保存新的日志
- SNN 从 NN 复制 fsimage 和 edits 到本地的 checkpoint 目录
- SNN 基于 fsimage 开始回放 edits，然后将结果写入新的 fsimage 到磁盘上
- SNN 将新的 fsimage 发送给 NN
- NN 接受 SNN 提交过来的 fsimage 作为新的快照，并且将之前的 edits.new 改名为 edits。这样就完成了一次合并。

每经过一个小时（默认），或者当 edits 到达 64 MB (默认）时，这个合并日志的操作就会发生一次。


NameNode High Availability
----------------------------

很长一段时间以来 HDFS 是管理员的噩梦，因为NN存在单点故障(SPOF)。所以在 Hadoop 2.x 开始，NameNode 开始支持 High Availablity。允许存在一对儿NN以active/standby方式同时运行。NN之间必须有方式可以共享 fsimage 和 edits。目前有两种方式，一种是传统的通过 NFS 共享；另一种是通过新的 Quorum Journal Manager 来实现。

HA可以配置为故障出现后自动激活standby，也可以手动。默认是手动。active/standby可以是由管理员在可控环境下转换，也可以由于失败被迫转换。

管理员需要注意的是，需要确认active的NN确实出了问题，还是说只是单纯的由于网络等问题standby暂时无法访问到active了。否则，贸然激活standby，会导致两个NN都认为自己是active，共同撰写edits，然后就傻了。这被称为 split brain，有人翻译成“脑裂”。当然，有些人会认为只要激活standby，先通过RPC让active停止。不过这可能会导致另一个问题，STONITH (Shoot the other node in the head 一枪爆头……)，有些时候这是没办法的事情，但是依旧需要小心。

standby的NN会承担起 SNN 中合并日志的职责，因此配置了NN HA后，不再需要额外的SNN了。

1.x的时候由于无奈，只能考虑使用传统的HA方式，比如Linux HA进行热备。但是，对于HDFS来说并不是一个很好的方案。主要的问题是，各个DN的 Block Report 只会保留于内存中，而不会写入磁盘。从而导致切换时会产生和某些DN Block不一致的东西。而且更惨的是由于采用虚拟IP技术，各个DN并不会意识到NN已经挂了，还会以为NN啥都明白呢，而不会发送新的Block Report过去。就算各个DN开始发送Block Report，对于几千节点的集群而言，依旧会有许多分钟的时间NN才能收集齐以及消化这些Block Report。


Namenode Federation
---------------------

1.x的Hadoop还有一个问题，集群的规模受NN的内存所限制。为了解决这个问题 2.x 提出了 Namenode Federation。这个东西实际上非常像 Linux 文件系统中将各个驱动器挂载(`mount`)到各个不同的目录位置的方式。使用 NN Federation 后，会使用一个新的FS叫做 `ViewFS`，然后不同的NN将负责ViewFS下面不同的目录，也称为 Namespace。

访问和使用 HDFS
-----------------

### 使用HDFS API

最常见的使用 HDFS 的方式是直接使用其 Java API。如果要使用HDFS API必须满足两个条件，一个是需要配置文件，通过配置文件可以得知 NN 的位置；另一个可以访问到Hadoop API的库，一般是 JAR 文件。

所谓 Hadoop API 的客户端有许多种情况，比如，独立的应用程序，像`hadoop fs`命令行工具；MapReduce的Job；HBase的 Region Server等。

需要注意的是，由于客户端除了需要联系NN外，还需要直接访问 DN，因此必须要保证客户端和HDFS中所有节点的连接互通能力。

hadoop fs 中可以指定副本数，如：`hadoop fs -setrep 5 -R /user/esammer/tmp`


### FUSE

虽然一般不会这么做，但是 HDFS 支持使用 FUSE (Filesystem In Userspace) 挂载在文件系统中。不过需要注意的是HDFS的特性，比如不支持随机写入、延时较长、随机读性能较差。


### REST API

从 Apache Hadoop 1.0、CDH 4之后，NN的Web接口中提供了 REST API 访问的能力，这称之为 WebHDFS。比如，对应于 `hadoop fs -ls /hbase` 这么一条命令的 REST 访问是： `http://namenode:50070/webhdfs/v1/hbase/?op=liststatus`。需要注意的是，客户端使用WebHDFS访问HDFS和通过HDFS API的流程几乎相同，换句话说，客户端需要自己去连接所需数据块的DataNode，从而以取得数据。

于此同时，还引入了一个成为 HttpFS 的东西，这个解决了上面 WebHDFS 的问题，它相当于一个代理，客户端只需连接HttpFS所在服务器，该服务器会负责与集群内的其他DN联系。当然，这样也会导致数据集中于此服务器，如果很多客户端的话，可能会在这里产生瓶颈。






















