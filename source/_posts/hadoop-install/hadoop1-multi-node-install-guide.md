---
layout: post
category: tutorial
title: Hadoop 1.2.1 多节点全分布配置
tags : [hadoop, hadoop install]
date: 2014/03/03
---

一、集群配置
============

一般来说，按照集群的规模会有不同的拓扑结构。

1.1 小规模集群(2~10节点)
--------------------

这类小规模集群主要是针对测试、练习、开发，甚至可能是比较小一些的生产集群。

这种规模下，一般会将 NameNode 和 JobTracker 都集中到一台计算机上。小的集群一般不大需要 SecondaryNameNode，如果有的话，也很有可能将其放到和NameNode相同的计算机上。

当然，这样的问题很明显，如果这台主节点崩溃了，比如硬盘彻底坏掉了，集群也就崩溃了，所有 HDFS 上的内容就全部丢失了。不过这种规模的集群一般也不太担心数据的丢失问题。比如只是开发、测试用，重新安装、配置以及导入数据也不是什么大不了的事情。

对于生产环境中的小规模集群，如果有可能，建议尽可能存在 SecondaryNameNode，并放到不同的计算机上，最起码是不同的硬盘上。这样当 NameNode 崩溃后，HDFS 的信息依旧可以从 SecondaryNameNode恢复，而不会造成太严重的数据丢失问题。

除了上述的主节点，其它的节点都将是同时运行 DataNode 和 TaskTracker。


1.2 中规模集群(10~40节点)
-------------------------


只要不是那些拥有几千节点的大公司的话，大多生产环境的集群规模是这种1个或有限的几个机架的规模。

这种情况，依旧会把 NameNode 和 JobTracker 安装到同一个节点上，反正他们都是单点故障，如果 NameNode 坏了，JobTracker也跑不了。但是会配备 SecondaryNameNode，并部署到一个独立的计算机上。

其它的节点和小规模一样都是同时跑 DataNode 和 TaskTracker。

1.3 大规模的集群(100+节点)
--------------------------

在集群规模比较大的情况下，NameNode 恐怕已经非常繁忙于处理这么多节点的数据维护事务，而 JobTracker 也会很繁忙的调度安排 MapReduce 的任务。因此，NameNode、JobTracker和SecondaryNameNode 都将会部署在独立的计算机上，以分散负载。

由于必然是多机架的环境，还需要配置 Rack awareness，也就是让 Hadoop 意识到机架和节点的关系以帮助优化。此外，还会分别对网络、HDFS以及MapReduce部分进行优化设置。


1.4 本实验节点拓扑结构
-----------------------

我们将配置5节点全分布模式，其中包含：

- 两个主节点：
	- 一个 `NameNode`：
		- `hadoop-master` (`10.0.1.110`)
	- 一个 `SecondaryNameNode`：
		- `hadoop-secondary` (`10.0.1.111`)
- 三个从节点(`DataNode` + `TaskTracker`)：
	- `hadoop-data1` (`10.0.1.121`)
	- `hadoop-data2` (`10.0.1.122`)
	- `hadoop-data3` (`10.0.1.123`)

选择这种实验环境可以涵盖多种实际生产环境中可能碰到的问题。比如模拟NameNode 宕机，需要利用 SecondaryNameNode 进行恢复；或者添加节点、移除节点、均衡节点数据等。

我们将基于之前所做的[单节点伪分布](/tutorial/hadoop-install/hadoop1-single-node-install-guide.html)的结果进行配置，因此请先完成单节点伪分布实验： http://twang2218.github.io/tutorial/hadoop-install/hadoop1-single-node-install-guide.html 。


二、`hadoop-master` 节点配置
=============================


经过前面的实验，我们有了一台伪分布的 `hadoop-master` 节点，要想将其配置改为全分布模式，还需要进行一些额外的修改。


2.1 指定 SecondaryNameNode
---------------------------


我们需要编辑 `conf/masters` 文件。不要被这个文件名所迷惑了，它虽然名为 `masters` 但并不是配置 NameNode 或者 JobTracker 的，相反，它内部包含的是 SecondaryNameNode 的主机名。默认这里面是 `localhost` 也就是本机，这也是为什么伪分布模式，我们可以看到 SecondaryNameNode 运行在本机上的原因。我们将该文件内容改为：

```
hadoop-secondary
```

这样将来启动 HDFS 的时候，NameNode 就会通过 SSH 连接 `hadoop-secondary` 以启动 SecondaryNameNode了。

> 如果不需要集群中存在 SecondaryNameNode，只需要将 `conf/masters` 中的内容清空即可。

2.2 指定数据节点
-----------------

我们需要编辑 `conf/slaves` 文件。默认情况下里面是 `localhost`，也就是本机，我们将其改为：

```
hadoop-data1
hadoop-data2
hadoop-data3
```

这个文件将会由 NameNode 和 JobTracker 分别读取，NameNode 从该文件确定 DataNode 的节点，而 JobTracker 需要由该文件确定 TaskTracker 的节点。这一点在 NameNode 和 JobTracker 在一个节点，并且 DataNode 和 TaskTracker 在一起的时候，不是很明显。但是一旦规模扩大需要进一步的分离负载的时候，比如 NameNode 和 JobTracker 分离在两个计算机时，这里需要注意一下该文件的一致性。

2.3 `conf/*-site.xml`
----------------

在单节点配置的时候，在配置 `conf/*-site.xml` 文件时提到过，主机名可以使用 `localhost`，因为反正也是自己。但是，在现在多节点的时候，则务必要将 `conf/*-site.xml` 中的 `localhost` 替换为 `hadoop-master`。因为，DataNode 在启动后，会读取 `conf/core-site.xml` 相关内容，其中很重要的一项，就是寻找所指定的 NameNode 的位置，即 `fs.default.name` 选项；TaskTracker 也是一样，需要通过读取 `conf/mapred-site.xml` 文件中的 `mapred.job.tracker` 项，得知 JobTracker 的位置。

除此之外，我们还需要修改 `conf/hdfs-site.xml` 中的 `dfs.replication` 项。在单节点配置中，我们由于只有一个 DataNode，曾经将其设置为`1`。现在我们将包含3个数据节点，已经满足了默认3个副本的需求。不过为了后续“移除节点”的实验需求，这里我们不使用默认的 `3` 分副本，而将其设置为`2`：

```xml
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
```

三、其它节点
============

我们的 `hadoop-master` 现在实际上是包含了 NameNode, SecondaryNameNode, DataNode, JobTracker 以及 TaskTracker 所有的所需的东西。因此对于其它节点，我们只需要“克隆”一下 `hadoop-master`，然后，修改一下主机名和IP地址，就OK了。

幸运的是，在虚拟机世界里面，这是一个非常简单的事情。VirtualBox 支持“复制”，也就是克隆虚拟机。

不过在克隆虚拟机之前，我们需要在克隆前最后一次启动 `hadoop-master` 机器 **【注意，只是启动 Ubuntu 系统，不要启动 Hadoop】** ，并且额外做两个操作。

3.1 清空 `hadoop.tmp.dir` 目录
-------------------------------

在前面配置单节点伪分布的配置过程中，曾经在 `conf/core-site.xml` 文件中配置了 `hadoop.tmp.dir` 项，当时我们将其中的值设为 `${user.home}/hadoop/tmp`。这里面包含了所有Hadoop的数据文件，特别是 HDFS 的文件。

我们现在要启动一个新的集群，将重新格式化 NameNode，为了防止出现 `namespaceID` 或 `storageID` 冲突的问题，我们这里将清空该文件夹，以确保 Hadoop 集群将处于一个全新的环境。

```bash
rm -rf ~/hadoop/tmp
```

3.2 删除网卡MAC地址绑定规则文件
-------------------------------


网卡MAC地址绑定规则文件是为了防止硬件变化后软件错误的映射了网卡的名字。

比如，有一台服务器插入了一块儿`网卡A`和一块`网卡B`。假设，默认时，`网卡A`对应于 `eth0`，而`网卡B`对应于`eth1`。由于某些原因，我们拔掉了`网卡A`，或者交换了两块网卡的插槽，这个时候， **如果没有固化MAC地址绑定** ，那么按照顺序`网卡B`会成为`eth0`，而导致原先针对`eth1`也就是`网卡B`的配置失效了，而原来针对`网卡A`的配置，却错误的应用到了`网卡B`上。这是不应该发生的事情。

因此，为了解决这个问题，当第一次发现某块儿网卡并且按照顺序分配了名字后，会将其写入到一个文件中，这样在未来无论怎样插拔、重新启动，服务器都会记得物理的网卡和名称之间的关系。这个文件就是 `/etc/udev/rules.d/70-persistent-net.rules`。

如果我们克隆了虚拟机，分配了新的 MAC 地址后，由于这个配置文件记录了原虚拟机的网卡MAC地址的原因，系统会认为这是一块儿新的网卡，因此应该分配一个新的名称给这个网卡，如`eth1`，由此就导致了原有针对 `eth0` 的配置都失效了，而 `eth1` 的配置却没有。

Ubuntu 在这种情况，启动时会有错误提示：

```
Waiting for network configuration...
Waiting up to 60 more seconds for network configuration...
Booting system without full network configuration...
```

并且，默认会等待`120秒`，看看是不是有没有系统程序能够把“新”的`eth1`配置好（中间可以手动终止等待）。当然这里是因为错误导致的，所以，什么都不会发生，只会在无谓的等待后，才会进入系统，而且无法使用网络。

解决这个问题，很简单，只需要删除掉 `/etc/udev/rules.d/70-persistent-net.rules` 文件，重新启动即可。 **不用担心，这个文件本身就是自动生成的，因此会在下次启动时自动创建。**

那么我们可以选择克隆前，最后一次运行 `hadoop-master` 时，删除这个文件；也可以选择克隆之后，每个虚拟机启动后，分别手动删除一遍。显然我们应该选择前者：

```bash
sudo rm -f /etc/udev/rules.d/70-persistent-net.rules
sudo poweroff
```

第一行是删除该文件，第二行则是关机，以准备克隆虚拟机。


3.3 克隆虚拟机
---------------

右键点击我们的虚拟机，选择`“复制...”`

![clone-vbox-1](/pic/vbox/clone-vbox-1.png)

接下来会提示新虚拟机的名字，这个名字是 VirtualBox 显示用的名字，而不是主机名，我们需要在虚拟机里面才能进行主机名设置。当然，这里的名字一般尽量和主机名一致，防止混淆。

**需要注意的是，我们一定要勾选“重新初始化所有网卡的MAC地址”，这样防止 MAC 地址冲突。**

![clone-vbox-2](/pic/vbox/clone-vbox-2.png)

对于我们的情况而言，副本类型是完全复制。

![clone-vbox-3](/pic/vbox/clone-vbox-3.png)

稍微等待几分钟后，虚拟机就准备好了。

![clone-vbox-4](/pic/vbox/clone-vbox-4.png)

3.4 修改主机名和IP地址
-------------------------

我们已经在单节点伪分布配置的时候已经修改过主机名和IP地址，因此，请参考[准备 Hadoop 运行环境](/tutorial/hadoop-install/prepare-hadoop-runtime-environment.html)  ⇨ {`3.1 修改本机主机名`, `3.2 配置静态 IP`} 来配置主机名和IP地址。分别配置4台克隆的虚拟机：

- `hadoop-secondary`
- `hadoop-data1`
- `hadoop-data2`
- `hadoop-data3`

由于克隆虚拟机的原因，我们不需要配置 `/etc/hosts` 等其它配置了。

四、操作集群
============

4.1 格式化 NameNode
-------------------

第一次启动 Hadoop 集群前，我们需要先格式化 NameNode。

首先将各个虚拟机启动起来，然后在 `hadoop-master` 上面格式化 NameNode：

```bash
hadoop namenode -format
```

4.2 启动集群
--------------


面对集群环境时，就不建议使用 `start-all.sh` 了，应该分步启动，这样更便于各节点间的协调以及排障。

首先，应该启动 HDFS：

```bash
start-dfs.sh
```

我们可以通过在各个节点执行 `jps`，判断是否已经启动了该节点对应的服务。比如，在 `hadoop-master` 上验证 `NameNode`；在 `hadoop-data1` 上验证 `DataNode`等等。

此外，我们还应该去 [http://hadoop-master:50070](http://hadoop-master:50070) 去检查 HDFS 是否运行正常。比如， `Live Node` 是否是3个；如果进入了 Safe Mode，是否在30秒后，自动退出了 Safe Mode 等。出现了任何问题，到相应的节点上的 `~/hadoop/hadoop-1.2.1/logs/` 目录下寻找对应于节点的日志，根据里面的错误进行排障。

在确认 HDFS 启动正常后，启动 MapReduce：

```bash
start-mapred.sh
```

同样需要到各个节点去验证 `jps` 是否 `JobTracker` 和 `TaskTracker` 都启动正常，以及通过 [http://hadoop-master:50030](http://hadoop-master:50030) 判断 JobTracker 和 TaskTracker 是否正常。

4.3 运行示例程序
--------------------

当一个集群建立后，一般会通过运行示例程序的方式以确定其能够工作正常。 

这里我们可以参考[单节点伪分布配置](/tutorial/hadoop-install/hadoop1-single-node-install-guide.html) ⇨ `3.3 运行示例程序`一节，在这个多节点中的集群中运行示例程序。注意，最好在 `hadoop-master` 上运行。

4.4 移除 DataNode
------------------

假设我们需要维护一个 DataNode 所在的服务器，维护操作有可能会导致其运行不正常，甚至需要关机。

如果可能，我们当然不希望产生数据丢失的情况。虽然 Hadoop 默认对数据保留了 3 份副本，但一般来说，我们是不希望通过粗暴的关机来测试这个能力的。

最好的做法，是在维护前通知 NameNode，让 NameNode 有所准备，将必要的数据移动到别的节点上。当一切准备就绪后，我们再断开 DataNode。这个行为，我们称之为 `Decommission`。

首先，我们需要在 NameNode，也就是 `hadoop-master` 上准备一个文件

```bash
nano ~/hadoop/hadoop-1.2.1/conf/excludes
```

在其中，加入要移除的节点，和 `slaves` 一样，一个节点一行，假设我们需要移除 `hadoop-data2`

```
hadoop-data2
```

其次，我们需要修改 `conf/hdfs-site.xml`，在其中添加一项 `dfs.hosts.exclude`，并在对应的值中，写入刚才新建的这个文件的绝对路径：

```xml
  <property>
    <name>dfs.hosts.exclude</name>
    <value>${user.home}/hadoop/hadoop-1.2.1/conf/excludes</value>
  </property>
```

然后，我们需要通知 NameNode，允许使用的 DataNode 已经有所变更：

```bash
hadoop dfsadmin -refreshNodes
```

然后，我们打开 Web 界面： [http://hadoop-master:50070](http://hadoop-master:50070) ，我们可以查看 `Decommissioning Nodes` 的信息，会看到 `hadoop-data2` 在其中，以及需要复制的数据块数。在 `Live Node` 页面中，我们也可能会发现 `hadoop-data2` 正处于 `Decommission In Progress` 的状态。

我们也可以在命令行查看节点 `Decommission` 状态：

```bash
hadoop dfsadmin -report
```

此时，NameNode 会根据 `hadoop-data2` 上所存储的数据块的信息进行分析，看哪些数据块在 `hadoop-data2` 关闭后，会出现无法满足其副本数目的需求。如果存在这样的数据块，NameNode 就会将其复制到其它节点上，以保证 `hadoop-data2` 移除后不会影响系统数据的完整性需求。

根据其节点上需要复制的数据的多少，等待一段时间后，节点状态会变为 `Decommissioned`，并从 `Live Nodes` 以及 `Decommissioning Nodes` 中去掉，进入了 `Dead Nodes` 中。这就说明一切已经准备就绪，我们可以关闭这个节点了。

我们有3个从节点，如果之前配制过程中，`dfs.replication` 设置为 `3`（或者注释掉该项，使用默认的3个副本）,那么在`Decommission`的过程中，会导致 NameNode 无法移动数据块以达到所有数据块都满足其副本数量。因为移除节点后，将只有2个节点，任何数据块最多也只能有2个副本，而无法满足3个副本的需求，从而，许多数据块都将处于`Under-Replicated Blocks`的状态。因此，`hadoop-data2`将永远处于 `Decommission In Progress` 的状态。

此外，需要注意 `Under-Replicated Blocks` 的数量，如果该数值不为`0`，也会导致 Decommission 无法结束。可以使用 `hadoop fsck /` 去检查是不是有曾经未完成的任务导致了副本不足的问题。

> 有趣的是，`conf/excludes` 配置只对 HDFS 有效，而对 MapReduce 无效。换句话说，经过上面的配置后，NameNode 会停止使用 `hadoop-data2` 了，但是，JobTracker 似乎依旧会使用 `hadoop-data2` 上面的 TaskTracker 执行任务。因此，为了安全起见，应该在 Decommission 之后，停止 `hadoop-data2` 上面的 Hadoop 服务。

当节点状态已经是 `Decommissioned` 的时候，我们已经可以停止该节点上的服务，并进行维护了。我们需要在 `hadoop-data2` 上面运行停止 DataNode 和 TaskTracker 服务的命令：

```bash
hadoop-daemon.sh stop datanode
hadoop-daemon.sh stop tasktracker 
```


假设我们维护完 `hadoop-data2` 了，也重新启动该虚拟机了，现在需要让它重新加到集群中去。

首先，需要 NameNode 知道 `hadoop-data2` 不应该被排除在外了。因此，需要在 NameNode 上，将 `conf/excludes` 文件中的 `hadoop-data2` 删除。然后通知 NameNode 变更情况：

```bash
hadoop dfsadmin -refreshNodes
```

这样，NameNode 就允许这个节点重新加入集群了。

接下来，我们需要在 `hadoop-data2` 上启动 DataNode 和 TaskTracker：

```bash
hadoop-daemon.sh start datanode
hadoop-daemon.sh start tasktracker 
```

> 如果未通知 NameNode `hadoop-data2` 已经可以重新上线，会使得，在 `hadoop-data2` 上，因为 NameNode 不允许其加入集群，而启动 DataNode 失败。

4.5 添加 DataNode
------------------

Hadoop 允许我们在运行时添加节点，操作方式很简单。

首先，我们需要修改 `hadoop-master` 上面的 `conf/slaves` 文件，将新的节点加入进去。然后通知 NameNode：

```bash
hadoop dfsadmin -refreshNodes
```

最后，我们在配置好的新的节点上启动 DataNode 和 TaskTracker：

```bash
hadoop-daemon.sh start datanode
hadoop-daemon.sh start tasktracker 
```

4.6 均衡 DataNode
------------------

如果有很多节点被 `Decommission`，或者我们添加了一些新的节点，集群中的数据很有可能会不均衡，导致有的节点负载过重，有的则无事可做。此时，我们可能需要重新平衡一下各节点之间的数据：

```bash
start-balancer.sh
```


4.7 停止集群
-------------

停止的集群的顺序应该和启动集群顺序相反。

首先，停止 MapReduce：

```bash
stop-mapred.sh
```

确认 MapReduce 停止后，继续停止 HDFS：

```bash
stop-dfs.sh
```


 















