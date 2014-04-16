---
layout: post
category: readings
title: 《Hadoop Operations》读书笔记 - 3 - 第四章 规划集群
tags : [hadoop, Hadoop Operations]
date: 2014/04/05
---
> **Chapter 4: Planning a Hadoop Cluster**
> Eric Sammer ["Hadoop Operations" - O'Reilly (2012)](http://shop.oreilly.com/product/0636920025085.do) ... (p41 ~ p73)


选择 Hadoop 发布以及版本
=========================

计划部署 Hadoop 集群的第一件事情就是选择 Hadoop 的发布和版本。需要开发人员、分析师、以及BI类其他系统共同来决定。

一般提到 Hadoop 往往除了 Hadoop 核心外，还会需要其生态圈的其它组成部分。所有这些组成部分必须要考虑到兼容性的问题，包括二进制兼容和API兼容。

Apache Hadoop
--------------------

在 1.0 以前，Apache 很久才会发布一个版本，不过随着Hadoop越来越受到关注，有更多的人投入进来，现在频繁多了。从 1.0 之后， Apache 的发布除了原有的 tarball 外，还增加了 RPM 和 Deb 来满足 RedHat和Debian两大包管理体系的需要。

CDH (Cloudera's Distribution Including Apache Hadoop)
--------------------------------------------------------

Cloudera 是一个为 Hadoop 提供管理、咨询、技术支持的公司。他们有自己的发行，称为 CDH。而且像 Apache Hadoop 一样，CDH发布的软件都是开源的。

Cloudera 会基于一个 Hadoop 的稳定版本，然后积极维护一些补丁，而且负责处理一些与系统相关的版本问题。Cloudera 公司的很多雇员同时也是 Apache Hadoop 的 committer （committer 是指有权利向 Apache 提交源代码的人，这些人一般是这个项目的积极维护者）。

由于大多数人部署 Hadoop 都是部署很多生态圈相关的部分，所以 CDH 除了 Hadoop 本身外，还包含了 HBase, Hive, Pig, Sqoop, Flume, ZooKeeper, Oozie, Mahout, Hue, Impala 等。

CDH 基本每年一个主要的版本发布，大约每个季度会有以修复补丁为主的小版本发布。CDH4包含了 Hadoop 2.0 因此包含了 YARN。即将发布的 CDH 5 将基于 Hadoop 2.3。

CDH 同样提供了 tarball 安装，以及 RHEL 5/6 的RPM包，SuSE 的 RPM包，以及 Debian 的 Deb包。此外，还提供了包管理软件 yum, zypper, apt 的软件库的源，因此各个系统部署非常简单。

版本和功能
-----------

选择版本的考虑主要有以下几个方面：

- 所需要的稳定程度
- 所需要的功能

Hadoop 的版本号是很混乱的，而且小分支很多。就现在情况而言，可以总结为两大分支：

- 0.20 ⇨ 1.x
- 0.23 ⇨ 2.x

按照功能考虑：

| Feature | 0.20 | 0.23 | 1.x | 2.x | CDH 3 | CDH 4 |
| ------- | ---- | ---- | --- | --- | ----- | ----- |
| HDFS append |  | ✔      | ✔    | ✔    | ✔       | ✔       |
| Kerberos|      | ✔      | ✔    | ✔    | ✔       | ✔       |
| HDFS symlink | | ✔      | ✔    | ✔    | ✔       | ✔       |
| YARN (MRv2) |  | ✔      |     | ✔    |       | ✔       |
| MRv1    | ✔      |      | ✔    |     | ✔       | ✔       |
| Namenode Federation||✔|     | ✔   |       | ✔       |
| Namenode HA |  | ✔      |     | ✔    |       | ✔       |


硬件选择
==========

Hadoop 集群的硬件选型和其它类型的集群没有太大差异，选择时需要对将要在上面运行的应用的特性有深入的了解。

有一个常容易出现的误解，**“Hadoop的一个优势是可以运行在普通的商用服务器上”，很多人把这句话理解为 “Hadoop 可以运行在很廉价的计算机上”。这是不对的。**这句话的意思是指你没必要去购买那些顶级服务器以完成大数据计算；没必要在集群中普遍部署RAID或者其它昂贵的存储构架；没必要在集群中的每个服务器上进行高可用的热备配置。但是，服务器最基本的稳定性和性能是要有保证的。

**Hadoop 的节点可以分为两类，master节点和worker节点。** 二者对硬件的要求并不完全相同。对于 master 而言，一旦 master 节点挂了，基本上会导致集群出现服务中断；而对于 worker 而言，挂了是正常现象，不会太大的影响集群可用性。

这种不同导致在选型的时候，依据需求，有可能会对master节点选择一种配置，而 worker 节点选用另一种配置。对于有些企业来说，也可能更倾向于对所有的master和worker使用单一配置的服务器，这样更方便维护和部署。

master 节点硬件选择
---------------------

master 节点，既运行 NameNode, JobTracker, SecondaryNameNode 的节点。这些节点对于集群来说是至关重要的，它们的故障将很可能导致所提供的服务中断。要维护高可用性，冗余是主旋律。因此，虽然 Hadoop 在绝大多数节点上只需要普通商用服务器即可，但是在 master 节点上则可能会进行更多的投资，比如双电源、双网卡、RAID等等。

**master 的特点是高内存需求，低存储需求。**

对于20节点的小集群，可以考虑以这个配置作为基线：

- 双 4核 2.6G CPU
- 24 GB DDR3 内存
- 双 1Gb 网卡
- 至少两块 SATA 硬盘(不配RAID)

如果节点数量达到300节点，这就是中等规模的集群了，可以考虑再增加 24 GB 内存，这样就是 **48GB** 内存了。对于大型集群，最好再翻一番， **96GB** 会好一些。

### NameNode 选型

注意三件事情，这与 NameNode 的正常工作息息相关：

- 足够的内存
- 适当的但是专用的硬盘
- 只运行 NameNode，不要和别的东西混在一起

NameNode 要记录 HDFS 中的元数据，即包括文件名、权限、所有者、所有组、每个文件对应的Block列表，以及每个Block的副本目前存在于哪个机器上。这些信息会随着集群的使用以及规模而增加。

由于Block的大小一般来说是64MB, 128MB 这个级别。这是为了处理大文件时，比较节约Block元数据所占用的资源。那么随之而来的问题就是如果有大量的小文件，则会破坏这种优化。任何一个Block，或任何一个文件名，都会占用 NameNode 的内存（要记住，这些信息是在内存里的，硬盘的只是固化的东西），而目前每个这样的东西大约占用 **140** 个字节。 **粗略的估计，大约1百万个Block或文件，会占据NameNode 1GB的内存** 。

这些数据会被固化到硬盘上，再加上日志， **基本不会太消耗存储空间，可以估算为小于1TB** 。但是，存储的稳定性非常重要，一般是使用JBOD方式或者RAID方式，而且应该考虑还将一份保存到NFS上，以防止机器内部的严重损毁（比如机架火灾）。

需要注意的是，实际选型过程中很可能会因为考虑为了部署方便而购买具有单一配置的机器，需要针对实际的环境对多种因素考量，这里只是给出一些考虑因素。

### SecondaryNameNode 硬件选择

SecondaryNameNode 的特征和 NameNode 是一样的，因此二者往往使用一个一个硬件配置。此外，由于 Hadoop 2.x 开始，支持 NameNode HA，而其中 Standby 的 NameNode 具有 SecondaryNameNode 一样的功能，因此将只有 Standby NameNode 而没有 SecondaryNameNode 了，因此二者硬件配置必然一样。

### JobTracker 硬件选择

和 NameNode 与 SecondaryNameNode 一样，JobTracker 需要大内存。内存中将记录所有 Job 和 Task 的状态、计数器、进展情况。默认 JT 在内存中保留 100 个运行过的 Job 信息。

当然，如果集群中运行了很多的 Job 而且有大量的子 Task，那么内存占用很快会上去。 ** 不像 NameNode，JobTracker 的内存使用是无法估计的，因此一定要关注 JobTracker 的内存占用情况。**

Worker 节点选型
------------------

由于每个工作节点同时既是存储也是计算，因此选型时需要考虑有足够的存储空间，以及要有足够的计算能力（CPU速度和内存大小）。一般来说，都是从相对平衡的角度开始选型，然后再加以变化。

举个例子，假设系统每天产生1TB数据，由于 Hadoop HDFS 默认是3个副本，因此每日我们对HDFS的需求就增长 3TB。由于每个工作节点同时还运行 MapReduce 的事情，shuffle/sort 需要一定的临时空间， **一般的考虑是按照磁盘空间20%~30%作为MapReduce的临时目录** 。如果服务器的配置为 12个2TB的磁盘，那么以为这将只有18TB将用于HDFS，换句话说，这样的一台服务器可以保留6天数据。

下面是一个典型的配置，可以以此作为基线考虑：

- CPU： 2 x 6核 2.9GHz
- 内存： 64GB DDR3 ECC
- 硬盘控制器： SAS 6Gb/s
- 硬盘： 12 x 3 TB LFF SATA
- 网络： 2 x 1Gb Ethernet

下面是一个高端的配置：

- CPU： 2 x 6 核 2.9GHz
- 内存： 96 GB DDR3 ECC
- 硬盘控制器： 2 x SAS 6Gb/s
- 硬盘： 24 x 1TB SFF Nearline/MDL SAS 7200 RPM
- 网络控制器： 1 x 10Gb Ethernet

*注：硬件发展很快，当读到这段文字时，可能已经有较大变化，因此这里只是一个参考。*


集群规模
---------

当机器硬件确定后，随之而来的就是规模问题，到底需要多少机器？规模取决于将要运行的任务的负载，包括所需的CPU、内存、存储、磁盘IO、以及任务执行频率。

通常，最初决定集群规模的是所需的存储空间。

还是按照之前的每天1TB数据的情况举例，那么需要61个标准配置的节点可以存储1年的数据；假设数据规模每月增长5%，则需要81个节点才行；如果数据规模增长是10%，则需要109个节点才能够满足存储需求。

这里我们假设的是12 x 2TB的存储配置。但是我们可以采用 6 x 2TB，但是节点数量翻翻，这样同样可以满足存储需求。需要注意的是，这种变化实际上增加了计算能力，但是，需要增加更多的电力、制冷、机架空间、网络端口密度。所以这是一种权衡，根据实际的需求考虑。

这里有一个先有鸡还是先有蛋的问题。只有 Job 运行了我们才知道真正的需求，而我们应该根据需求去选择硬件配置。这里有两方面考虑，一方面，我们需要通过运行小的数量级的数据，对复杂度进行分析，从而估算大概的需求；另一方面，集群部署后，应该严密关注资源开销情况，从而给下一次硬件升级给予一个更准确的指导性意见。

刀片服务器、SAN、虚拟化技术
----------------------------

“螺旋式前进”这种东西存在于各个领域，在大规模数据存储与处理上，一样如此。

曾经，当管理人员购买服务器的时候，如需更高的性能，则会购买更高配置的服务器，这种做法称之为 **“纵向扩展(Scale up)”** ；后来当我们意识到纵向扩展会带来更高的开销时，我们开始采用购买更多的服务器来解决问题，而不是购买更高端的服务器，这种做法叫做 **“横向扩展(Scale Out)”** 。今天的数据中心就是如此，由于机架空间是很重要的一个因素，因此发展出了现在我们称为 **“刀片服务器”** 的1U、2U 这种概念的服务器，可以使得在一个机架内装入更多的服务器。再后来，发现其实很多服务器的利用率都不高，于是又发展出了虚拟化技术，以更充分的利用各个服务器资源。

存储发展也是同样，先是单机内单块儿硬盘的存储追求更高的容量和更高的IO，遇到困难后，在单机内发展多块硬盘做RAID做小规模横向扩展，再后来，独立出SAN和NAS用很多硬盘来做大规模的横向扩展。

Hadoop 也是基于同样的横向扩展理念发展出来的。那么 Hadoop 在使用现在流行的刀片服务器、SAN和虚拟化技术上，是否和传统服务一样会提高性能呢？不尽然。

先说 Hadoop 操作的特点。Hadoop 是对IO性能非常敏感的，操作过程中将会有大量的IO操作。Hadoop 不需要 RAID，相反，使用RAID可能会降低 worker 的性能； Hadoop 希望硬盘使用 JBOD (Just Bunch of Disk 就是简单的一堆硬盘) 的简单形式，说白了，磁盘不需要特殊处理，简单的由操作系统挂载即可。这样的好处是，每一个磁盘将有独立的 IO，互相之间不需要协作，谁的数据准备好了，就把谁的数据从缓冲直接送到对应进程。而RAID的麻烦在于磁盘之间有协作，往往最慢的磁盘决定了整个RAID的性能，更严重的是，由于所有磁盘都通过一块RAID控制芯片，往往该芯片成为了磁盘性能的瓶颈。

对于 SAN 和 NAS 来说，具有和 RAID 同样的特性，由于大量的磁盘位于1-2个控制芯片之后，他们的IO无法像JBOD那样充分利用，受限于控制芯片，从而导致当集群中大量的节点并发访问的时候，很容易会造成其控制芯片的拥堵。这与传统意义上的服务器不同，在传统意义上 SAN 和 NAS 会大幅提高 IO 性能，但是 Hadoop 所需的 IO 要远远超过当前 SAN 或 NAS 的能力，至少在性价比上，SAN 和 NAS 并不是很适用于 Hadoop 的操作特性。

那么，当 Hadoop 遇到虚拟化后，问题同样就会暴露出来，由于虚拟化的系统无法意识到别的系统的存在，因此在磁盘IO调度上，无法整体的优化调度。我们知道磁盘和Flash不同，其固有的物理特性是顺序读写速度很快，但是寻道时间则很慢。而 **虚拟化技术会使得操作系统无法优化调度寻道操作，导致会产生大量的寻道，从而无法充分利用磁盘顺序读写的能力** ，而降低了性能（这也是为什么Hadoop Block非常大的原因之一，用于将数据顺序写入磁盘，并顺序读出，以大幅增加IO性能）。

**刀片服务器的问题是在于内部空间十分有限**，前文所知，Hadoop 的需要大量的IO，因此一个节点内，如果有更多的独立磁盘，其IO性能会更好。这也是为什么前文中提议配置时，24 x 1TB 的磁盘比 12 x 3TB 的磁盘要更好的原因。刀片服务器内部的空间限制，往往束缚了增加更多硬盘的可能性。

从这里，我们就更好的能够看出 Hadoop 所谓的运行于独立的商用服务器，以及其特意强调的 Share Nothing 的构架的原因。任务独立、IO独立对于Hadoop来说，会产生更好的性能，换句话说，这种 Share Nothing的构架，会更充分的利用硬件资源。

操作系统选择和准备
====================

Hadoop 本身绝大多数使用的是 Java 写的，但其中也有C/C++代码。另外，由于撰写的时候，基本以Linux为设计目标系统，因此其中充斥了大量的使用 Linux 构架的思想的代码，因此一般来说会选择在 Linux 上部署。需要注意的是，Hadoop 2.x 开始，已经可以在 Windows 上运行了，并且 HortonWorks 和微软公司合作，所做的 HDP 是有 Windows 发布版本的，可以在 Windows 上安装运行，而微软的 Azure 云中的 HDInsight 也同样使用了这种方式使得 Hadoop 可以在微软云环境中运行。

现行系统中，RedHat Enterprise Linux, CentOS, Ubuntu Server Edition, SuSE Enterprise Linux 以及 Debian 都可以在生产环境中很好的部署 Hadoop。因此选择系统更多的是取决于你的管理工具所支持的系统、硬件支持能力、以及你们所使用的商业软件所支持的系统，还有很重要的考量是哪个系统管理人员最熟悉。

配置系统是非常消耗时间，以及容易出错的，建议采用软件配置管理系统来进行维护，而不要手动去配置。现在比较流行的是 Puppet 和 Chef。

部署布局
----------

当我们通过解压缩方式安装 Hadoop 实验环境的时候，只需要把 Hadoop 视为一个目录即可调试运行。但是，实际上这里面包含了好几个不同性质的目录，有长期占用的存储目录、有只在MapReduce运行时在使用的目录，还有不需要写入的程序目录、日志目录等等。这些目录的特性不同，因此处理方式也不同，更重要的是，权限需求也不同。第五章中将重点讲，这里只是提一下。

- Hadoop 主目录： Hadoop 软件安装的目录，运行时除了脚本外只需要只读权限即可，脚本需要可执行权限。
- DataNode 数据目录： 用于保存 HDFS Block 数据。如果在设置中指定多块儿硬盘，会使用 round-robin 方法依次在磁盘中写入 Block，这样可以增加并发时的IO。
- NameNode 目录： 这里面将保留内存中的 fsimage 和 editlogs。如果配置时制定了多个位置，将会在所有位置中写入完全一样的内容，目的在于冗余备份。通常其中一个位置是NFS。
- MapReduce 本地目录： 这个目录是临时数据存放目录，其中内容仅在 MapReduce Job运行时才会存在。用于之前提到过的 MapReduce 中 shuffle/sort 过程。通常会安排在与 DataNode 数据目录同样的设备上。
- Hadoop 日志目录： 用于存储 Hadoop 运行日志的。
- Hadoop pid 目录： 当 Hadoop 进程运行时，会需要保留一个 .pid 文件来记录这个服务的进程号。文件很小，而且大小固定。
- Hadoop 临时目录： 用于存储一些临时的文件，典型的用处是向集群提交 Job 时，本机临时存储 jar 文件。一般默认位置为 `/tmp/hadoop-${user.name}/` 下。一般不需要更改。


软件
-----

最基本的，Hadoop 需要 JDK 方可运行。JDK 的版本是需要关注的，如果使用的是比较老的 Java 6，那么需要安装 Oracle (Sun) 的 JDK；如果是 Java 7 则可以使用系统默认的 OpenJDK 7。具体的兼容性经过了官方一些用户的测试后发布在： 

http://wiki.apache.org/hadoop/HadoopJavaVersions

一般来说是选择64位系统，因为一般所配置的内存都远远大于 4GB。

除此以外，还有一些软件虽然不一定是必须的，但是却可以很好帮助维护、监控系统。

- **cron**： 如果需要计划任务定时执行，一般会选择使用 cron。
- **ntp**： 服务器的时间如果是混乱的将会在排障时带来很大困难，而且对于上层软件也很可能导致错误。对于 Ubuntu 而言，ntp 是自动启动的。其它系统可能需要手动配置一下。
- **ssh**： 远程用户连接执行命令，这个重要性毋容质疑。
- **postfix/sendmail**： 一般会使用这个来发送邮件，虽然 Hadoop 自身不发送邮件，但是一些管理工具可能会需要。
- **rsync**： 这是一个很重要的同步软件，对于多台服务器的软件或配置同步非常有帮助。手动复制很容易出错。

主机名、DNS、身份认证
------------------------

这里一定要非常注意的，Hadoop 以及 HBase 对这个东西非常挑剔，在官方的邮件列表里，以及其它一些论坛中，经常看到很多人抱怨这类问题。要了解这里面为什么出问题，就需要对Hadoop到底是对域名、主机名、IP是如何处理的搞清楚。

我们都知道 DateNode 和 TaskTracker 都会定期向 NameNode 和 JobTracker 发送心跳。当第一次心跳发生时，master 开始意识到对应 worker 的存在。在心跳中，worker 会报告自己的标识符（可能是主机名、域名或者IP）。而 master 节点，将会以该标识符来表示该 worker。前文曾说过，当 HDFS client 与 NameNode 通讯要求打开一个文件的时候，NameNode 将会返回一系列 worker 节点，而这些 worker 节点就将以心跳中所报告的标识符来表示对应 worker。这就导致了一个问题，HDFS client 除了必须拥有直接和 Hadoop 云中所有的节点直接通讯的能力外， **还必须拥有正确解析 worker 所报告的标识符的能力** 。

明白这个过程后，就需要搞明白 worker 是如何找到自己是谁的？

配置中有些东西会影响下面的操作，这里我们先说一下默认的情况。

- 使用 Java 的 **InetAddress.getLocalHost()** 来取得主机名；
- 调用 **InetAddress#getCanonicalHostName()** 来对上一步取得的主机名取得 FQDN 形式的名称；
- 将得到的名字保存下来，然后用这个名字向 master 宣称自己的标识；

好吧，现在问题就变成了 `getLocalHost()` 和 `getCanonicalHostName()` 是做什么的了。

问题由此开始变得复杂了，因为很不幸，这东西返回结果是跟平台相关的，而且还会受到系统环境变量的影响。

在 Linux 上，如果用的是 Oralce 的 HotSpot JVM的话，`getLocalHost()` 使用的是 POSIX，即 Linux 中的 `gethostname()`，调用 `uname()` 系统调用。这显然不是去查询 DNS 或者 `/etc/hosts` 文件，虽然大多数情况，返回结果一致。系统命令 `hostname` 就使用了 `gethostname()` 和 `sethostname()`，而命令 `host` 和 `dig` 就是用了 `gethostbyname()` 和 `gethostbyaddr()`。前者是内核所见主机名的样子，后者是正常 Linux 名字解析的过程。

**getLocalHost()** 于 Linux 而言，如果 `gethostbyname()` 无法解析，则会出现问题，不过一般至少会有 `/etc/hosts` 中有一项匹配。而 Mac OS X 则在无法解析情况下，依旧返回首选网卡的活跃IP。

事情到了 **getCanonicalHostName()** 后就比较诡异了。根据 OpenJDK 中的实现，该函数调用了 `InetAddress.getHostFromNameService()`，而这个函数将直接使用地址到操作系统的名称解析服务中取得域名全称。这步还好，有问题的是下一步，它还会取得主机名所对应的全部IP列表，确保送查的IP在这个列表中。如果其间发生了任何错误，最初的IP地址会作为域名全称返回。

可以使用下面的代码在对应主机上运行一下看看：

```java
InetAddress addr = InetAddress.getLocalHost();
System.out.println("IP: " + addr.getHostAddress());
System.out.println("hostname: " + addr.getHostName());
System.out.println("canonical name: " + addr.getCanonicalHostName());
```

对于一些系统，特别是 RHEL 或者 CentOS，可能会自认为本机IP为 `127.0.0.1`，主机名为 `localhost`，甚至这是默认配置，这会导致严重的问题。因为它会报告给 master 自己为 `localhost`，结果master也是这么告知client，而client在连接 `localhost` 的时候，实际上是连接的自己，自然，就导致了失败。因此，对于这些系统要尤为注意。

用户、组和权限
----------------

处于安全隔离的目的，建议 HDFS 和 MapReduce 的服务使用不同的用户。而 MapReduce 的 Job 的执行用户，既可以选择是与 TaskTracker 相同的用户，也可以是选择提交 Job 的那个用户。**后者只有在 Hadoop 集群使用 secure mode 的模式配置启动时才可以使用。**

- 用户 hdfs： 运行 NameNode, SecondaryNameNode, DataNode
- 用户 mapred： 运行 JobTracker, TaskTracker, (或 Jobs)

默认情况下，如果通过 RPM 或者 Deb 方式、软件库源的方式安装 CDH Hadoop 的话，那么会自动创建这些用户，并且配置好以正确的用户启动这些服务。

还有一个需要调整的是 `/etc/security/limits.conf`，如：

```bash
hdfs	-	nofile	32768
mapred	-	nofile	32768
hbase	-	nofile	32768
```

此外，还需要调整各个服务所使用的目录的权限，如 HDFS 相关的目录需要调整权限为：

```
hdfs:hadoop		0700
```

而 MapReduce 相关目录要调整为：

```
mapred:hadoop	0700
```

日志目录则应该为 `root:hadoop	0775`。

这些目录的权限配置应该是在Hadoop部署运行前就配置好。一般推荐使用 Puppet 或者 Chef 来进行这种在很多台服务器上的控制。


内核调整
============

一般 Hadoop 所用的都是专用服务器，其上不会运行 Hadoop 以外的服务，因此，我们可以对内核进行优化配置。内核配置应置于 `/etc/sysctl.conf` 文件中，否则重启后会失效。

vm.swappiness
----------------

该选项控制了内核将内存交换到磁盘的倾向性。**数值范围 0~100** ，其值越大，内核就越倾向于将数据交换到磁盘。对于 Hadoop 而言，交换内存数据到硬盘可能会导致某些操作超时，特别是当磁盘有其他 IO 操作时。

这对于 HBase 尤为重要，因为 Region Server 需要维持其与 ZooKeeper 之间的通讯在一个特定的时间范围内，否则会被标记为故障。

要避免这类问题， ** vm.swappiness 应该被设置为 0，来告诉内核，如果可能的话，永远不要交换数据到磁盘 **。大多数系统默认是 60 ~ 80。

vm.overcommit_memory
----------------------

进程一般通过 `malloc()` 来申请内存，内核则根据可用内存来判断是否允许这个申请。而 Linux 则进一步的允许申请的内存超过物理内存+Swap，这称为 **Overcommit memory**。 

`vm.overcommit_memory` 内核选项有三种值：

- 0: **不允许** overcommit memory。有多少分配多少，超过了就拒绝分配；
- 1: **允许** overcommit memory，允许超过的百分比由内核选项 `vm.overcommit_ratio` 来控制。假设 `vm.overcommit_ratio` 是 50，物理内存是 1GB，那么允许多分配 500MB，换句话说，允许申请的总内存为 1GB + 500MB + Swap。再多申请，就返回失败。
- 2: **永远批准申请**，无论申请的内存有多大。嗯，这个东西看起来很可怕的选项。

一般来说，我们这里会选择 `1` 并且调整配额百分比。

磁盘配置
===========

选择文件系统
-------------

Hadoop 是可以运行在任何正常的文件系统上的。现在生产环境中经常使用的有 ext3, ext4 和 xfs。

**不要使用 LVM (Logical Volume Manager)** ，很不幸，对于 CentOS 和 RHEL 而言，这是默认选项，不要忘记修改。

### ext3

这是相对而言，经过了时间考验的文件系统。ext3 是在2001年11月份，加入到 2.4.15中的。相对于 ext2 而言，增加了写入日志，提高了稳定性。格式化分区的时候，建议使用下面的配置：

```bash
mkfs -t ext3 -j -m 1 -O sparse_super,dir_index /dev/sdXN
```

其中参数：

- -j： 启用日志
- -m 1： 默认情况下系统会为 root 保留 5% 的空间，这对于很大的文件系统而言，这个比例不小。对于 Hadoop 数据分区而言，这完全没必要。 **这个选项将其降为 1%** 。
- -O： 后面是可选项：
  - sparse_super： 减少 super-block 的备份数量；
  - dir_index： **使用 b-tree 索引来** 作为目录索引以提高访问性能，一般来说这都是默认选项。
  
### ext4

ext4 是 ext3 的继任者，最大的特点是是基于 extent 的，这对于顺序读取大段数据非常有帮助。此外，ext4 还支持日志 checksum，从而增加了数据的健壮性。这些特性都很好，不过 ext4 唯一稍显不足的是时间的考验。ext4 是在 2008年10月 添加到 2.6.28 中的。

格式化分区的时候，建议使用下列参数：

```bash
mkfs -t ext4 -m 1 -O dir_index,extent,sparse_super /dev/sdXN
```

大部分参数和 ext3 一样，只需要增加 `extent` 这个可选项以启用 extent 特性即可。

### xfs

这是 SGI 在1994年发明的文件系统，有很多独特的特性。和 ext3 与 ext4 一样，这也是支持日志的文件系统，但是其对并发读写操作支持非常好，因此很多系统如关系型数据库这类需要并行处理的系统会选择XFS作为数据存储的文件系统。

格式化时没有特殊选项：

```bash
mkfs -t xfs /dev/sdXN
```

挂载选项
----------

挂载不复杂，但是之所以需要单独提出，是因为有一个地方可以优化，需要特别注意。 **在挂载的时候，需要禁用文件访问时间的记录** 。默认情况下，文件系统会记录用户什么时候访问过该文件，对于文档类的系统来说很有帮助，但是对于Hadoop而言，这完全没有用处。相反，这还会因为多了一次写入磁盘的操作而降低磁盘IO性能。

禁用文件访问时间，只需要挂载的时候添加 `noatime, nodiratime` 即可，如：

```bash
LABEL=/data/1	/data/1	ext3	noatime,nodiratime	1	2
```

网络设计
==========

Hadoop 是IO hungry 的，既是磁盘IO hungry，也是网络IO hungry。虽然 Hadoop 在 Map 阶段调度任务时，会尽量使任务本地化，但是对于 shuffle/sort 以及 Reducer 输出来说，都会产生大量的IO。

虽然 Hadoop 不要求非要部署 10 Gb 网络，但是更高的带宽肯定会带来更好的性能。一旦你感觉需要2个以上1Gb网卡绑定以增加带宽的时候，就是考虑部署10Gb的时候了。

网络拓扑结构对 Hadoop 在某种程度上是有影响的。由于 shuffle/sort 的阶段会有大量的**东西向/横向**网络访问，因此网络的特点是任意节点间的带宽需求都很高。这与传统的Web服务形式的**南北向/纵向**带宽需要很高截然不同。如果网络拓扑设计时纵向深度很大（层级很多）就会降低网络性能。

![tree topology](/pic/hadoop-operations/tree-topology.svg)


**对于 Hadoop 而言，对横向带宽需求很高。** 由于这种原因，传统的树形拓扑网络就不是很适用与 Hadoop 的特性，更合适的是 **spine fabric** 拓扑结构。

![spine/leaf topology](/pic/hadoop-operations/spine-leaf-topology.svg)

进一步的信息可以参考 Brad Hedlund 的博客： http://bradhedlund.com/2012/01/25/construct-a-leaf-spine-design-with-40g-or-10g-an-observation-in-scaling-the-fabric/


