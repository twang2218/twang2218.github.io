---
layout: post
category: tutorial
title: Hadoop 1.2.1 单节点伪分布配置
tags : [hadoop, hadoop install]
date: 2014/02/19
---


前言
=============

本安装指南是针对 Ubuntu Server 12.04 进行 Hadoop 1.x 单节点伪分布模式配置的。其它系统或其它版本请酌情调整配置方法。

一、准备工作
================

我们需要有一台 Ubuntu Server 12.04 64位的虚拟机，如果尚未建立，请参考：【[在 VirtualBox 中安装 Ubuntu Server 12.04 64位版](/tutorial/hadoop-install/install-ubuntu-server-12.04-on-virtualbox.html)】 进行安装。

此外，我们还需要在这台虚拟机中为 Hadoop 的运行环境进行一些准备工作，具体的内容，请参考： 【[准备 Hadoop 运行环境](/tutorial/hadoop-install/prepare-hadoop-runtime-environment.html)】。

二、安装 Hadoop
==================


2.1 下载 Hadoop 安装文件
------------------


我们将安装当前 1.x 系列最新的版本 1.2.1，访问官网：http://hadoop.apache.org 

`Download Hadoop` ⇨ `releases` ⇨ `Download` ⇨ `Download a release now!` ⇨ 选择合适的镜像 ⇨ `hadoop-1.2.1` ⇨ 我们将下载 `hadoop-1.2.1-bin.tar.gz` 这个文件。

这个文件可以下载到本地，在传到 Ubuntu 上，但是更好的办法是直接在 Ubuntu 中下载。由于我们已经配置好了静态 IP，因此现在可以在主机上使用 SSH 连接 `hadoop-master`了。

```bash
ssh hadoop-master
```

登录进去后，我们准备 hadoop 的环境以及下载。我们将创建一个 `~/hadoop` 目录，以后所有的 hadoop 相关的软件都会放到该目录下。

```bash
mkdir ~/hadoop
cd ~/hadoop
```

> 这里需要说明的是，许多教程都最终将 hadoop 放到了 `/usr/local/hadoop` 目录下，这样做会带来很多权限问题。而很多教程，或者最后干脆使用 `root` 用户操作一切，或者使用 `chown` 来解决权限冲突，这都是非常不好的，很多时候是因为写这些教程的人对 Linux 不熟悉，把 Windows 的一些习惯带到了 Linux 里来。生产环境的搭建绝不是这么简单粗暴的改变所有权，更不可能是这样子滥用`root`权限。除了需要将 hadoop 放到符合 Linux 目录结构的位置外，包括配置文件和日志，也都应该指向正确的位置，并且建立适当的用户组，以及对不同的目录使用不同的权限，这将引入很多额外的工作，因此不适合在初学时涉及。对于开发和实验环境的搭建，我们这里的做法非常简单，足以胜任后续的学习、开发。

```bash
wget http://apache.dataguru.cn/hadoop/common/hadoop-1.2.1/hadoop-1.2.1-bin.tar.gz
```

如果网络环境较差，则需要检查下载的文件是否正确。执行 

```bash
sha1sum hadoop-1.2.1-bin.tar.gz
```

如果输出和下面的内容一样，则说明下载正确了。

```
267748f61c9e27c3e1895453863b435458255939  hadoop-1.2.1-bin.tar.gz
```

如果下载正确，则对压缩文件进行解压缩：

```bash
tar -zxvf hadoop-1.2.1-bin.tar.gz
```

执行上述命令后，会生成 `hadoop-1.2.1` 目录，至此，我们可以开始正式的配置了。



2.2 配置 Hadoop
------------------



### 2.2.1 配置 `PATH` 环境变量


hadoop 的可执行文件在 `hadoop-1.2.1/bin` 下，我们如果想执行该文件，或者使用完整路径的方式，或者将该路径加入到 `PATH` 环境变量中，以后我们直接执行文件即可。

```bash
nano ~/.profile
```

在最后一行添加：

```bash
export PATH=$PATH:$HOME/hadoop/hadoop-1.2.1/bin
```

保存退出。然后应用该配置。

```bash
source ~/.profile
```

> 这里注意到有的教程提到的是修改 `~/.bashrc` 文件。在很多情况下这是工作的，但是某些情况下则不能工作，因此[Ubuntu 官网关于环境变量的配置](https://help.ubuntu.com/community/EnvironmentVariables#A.2BAH4ALw.profile) 建议放在`~/.profile`文件中。

我们可以通过显示 hadoop 的版本来判断之前的配置是否正确。

```bash
hadoop version
```


### 2.2.2 配置 `*-site.xml`


之前的配置一直是环境配置，现在才是真正的 Hadoop 相关的配置。对于单节点伪分布模式，配置非常简单，我们只需要修改三个 `*-site.xml` 文件即可。默认情况下，这三个文件都包含一个空的`<configuration>` ，在这里，我们需要为三个文件的`<configuration>`中加入对应的配置。

为方便起见，我们将当前目录换到 `conf` 目录下：

```bash
cd hadoop-1.2.1/conf
```

#### `core-site.xml`:

```bash
nano core-site.xml
```

在 `<configuration>`中加入：

```xml
    <property>
        <name>fs.default.name</name>
        <value>hdfs://hadoop-master:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>${user.home}/hadoop/tmp</value>
    </property>
```

- `fs.default.name`： 通过这里指定 NameNode 的位置。我们可以在这里手动指定端口号，也可以忽略以使用默认的端口号。
- `hadoop.tmp.dir`：Hadoop 临时文件的位置，默认是 `/tmp`。这里如果不设置 `hadoop.tmp.dir`，Hadoop 是可以正常运行的。但由于使用的是 `/tmp` 目录，即内存，这显然不符合大容量存储的意图，而且数据也会随着关机而丢失。因此，我们需要将目录指定到硬盘上的位置。在这里，我们使用了 `${user.home}` 变量，该变量代表的是用户主目录 `$HOME`。 **这个目录需要记住，因为后续排障过程中，可能会需要到这个目录中检查 `current/VERSION` 文件的`namespaceID`以及`storageID`等。**


#### `hdfs-site.xml`:

默认的情况下，`dfs.replication`，也就是数据块默认副本份数，是3份。由于这里是单节点的伪分布模式，只存在一个 DataNode，因此，数据只可能有1份，所以我们需要将其修改为1。

```bash
nano hdfs-site.xml
```

在 `<configuration>`中加入：

```xml
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
```

#### `mapred-site.xml`:

```bash
nano mapred-site.xml
```

在 `<configuration>`中加入：

```xml
    <property>
        <name>mapred.job.tracker</name>
        <value>hadoop-master:9001</value>
    </property>
```

分别保存退出。


三、操作伪分布集群
===================

3.1 格式化 NameNode
-------------------


在配置完成后，我们需要进行一次 NameNode 的格式化。如果未格式化，NameNode 会因 HDFS 所需的数据结构错误而无法启动。

```bash
hadoop namenode -format
```

格式化之后，应该会出现与下面相似的输出：

```bash
14/02/18 19:38:16 INFO namenode.NameNode: STARTUP_MSG: 
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = hadoop-master/10.0.1.110
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 1.2.1
STARTUP_MSG:   build = https://svn.apache.org/repos/asf/hadoop/common/branches/branch-1.2 -r 1503152; compiled by 'mattf' on Mon Jul 22 15:23:09 PDT 2013
STARTUP_MSG:   java = 1.7.0_51
************************************************************/
14/02/18 19:38:16 INFO util.GSet: Computing capacity for map BlocksMap
14/02/18 19:38:16 INFO util.GSet: VM type       = 64-bit
14/02/18 19:38:16 INFO util.GSet: 2.0% max memory = 1013645312
14/02/18 19:38:16 INFO util.GSet: capacity      = 2^21 = 2097152 entries
14/02/18 19:38:16 INFO util.GSet: recommended=2097152, actual=2097152
14/02/18 19:38:16 INFO namenode.FSNamesystem: fsOwner=tao
14/02/18 19:38:17 INFO namenode.FSNamesystem: supergroup=supergroup
14/02/18 19:38:17 INFO namenode.FSNamesystem: isPermissionEnabled=true
14/02/18 19:38:17 INFO namenode.FSNamesystem: dfs.block.invalidate.limit=100
14/02/18 19:38:17 INFO namenode.FSNamesystem: isAccessTokenEnabled=false accessKeyUpdateInterval=0 min(s), accessTokenLifetime=0 min(s)
14/02/18 19:38:17 INFO namenode.FSEditLog: dfs.namenode.edits.toleration.length = 0
14/02/18 19:38:17 INFO namenode.NameNode: Caching file names occuring more than 10 times 
14/02/18 19:38:17 INFO common.Storage: Image file /home/tao/hadoop/tmp/dfs/name/current/fsimage of size 109 bytes saved in 0 seconds.
14/02/18 19:38:17 INFO namenode.FSEditLog: closing edit log: position=4, editlog=/home/tao/hadoop/tmp/dfs/name/current/edits
14/02/18 19:38:17 INFO namenode.FSEditLog: close success: truncate to 4, editlog=/home/tao/hadoop/tmp/dfs/name/current/edits
14/02/18 19:38:17 INFO common.Storage: Storage directory /home/tao/hadoop/tmp/dfs/name has been successfully formatted.
14/02/18 19:38:17 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at hadoop-master/10.0.1.110
************************************************************/
```

特别是关注其中是否报告格式化成功：

```
Storage directory /home/tao/hadoop/tmp/dfs/name has been successfully formatted.
```

如果没有，则需要根据错误信息进行分析。

至此，Hadoop 单节点伪分布模式就配置完毕了。


3.2 启动 Hadoop
----------------



```bash
start-all.sh
```

这条命令将根据之前配置的 `*-site.xml`，来启动 Hadoop 云（当然，此时，我们这个云是伪分布，就一个节点）。

然后，我们需要确定节点是否正常运行了。

```bash
jps
```

这条命令可以帮助我们查看当前系统运行的 Java 程序进程。如果配置正确，我们将看到类似下面的结果：

```bash
2613 Jps
2018 NameNode
2388 JobTracker
2143 DataNode
2514 TaskTracker
2290 SecondaryNameNode
```

需要注意的是，单节点伪分布正确运行后，应该会有5个组件运行：`NameNode`、`DataNode`、`SecondaryNameNode`、`JobTracker`、以及`TaskTracker`。缺少任何一个，都说明配置上有所错误。需要回去检查各项配置。

> 在出现故障后向论坛、朋友求助的时候，需要给对方提供你的 `conf` 目录下的配置文件、以及日志，否则对方无法理解你的故障可能的问题。日志目录：`$HOME/hadoop/hadoop-1.2.1/logs`


3.3 运行示例程序
---------------


就如同我们搭建好其它语言的编译环境后，会编写一个 `HelloWorld` 程序来测试整个环境是否工作一样，在 Hadoop 中，我们有自己的`HelloWorld`类的程序，叫做`WordCount`。

`ＷordCount` 是对纯文本进行单词出现次数统计的，为了使用 `WordCount`，我们需要先准备一些数据，也就是一些文本文件。这里，我们使用 [古腾堡计划](http://www.gutenberg.org/)中的英文图书。


### 3.3.1 下载


```bash
mkdir -p ~/hadoop/data/book/download
cd ~/hadoop/data/book/download
wget -r -l 2 -H "http://www.gutenberg.org/robot/harvest?filetypes[]=txt&langs[]=en"

```

这里`-l 2`是只取 2 级，因此并没有下载很多书，大约有200本英文书籍，解压缩后大约 90MB 左右。，如果觉得数据量不够可以酌情增加一些。根据当前的网速，下载需要一段时间。下载完成后，会出现类似于：

```
FINISHED --2014-02-18 19:44:10--
Total wall clock time: 7m 42s
Downloaded: 204 files, 34M in 6m 39s (86.0 KB/s)
```

### 3.3.2 解压缩


```bash
for f in `find ~/hadoop/data/book/download -name *.zip` ; do unzip -j $f *.txt -d ~/hadoop/data/book/input; done
```


### 3.3.3 将书籍放到 HDFS 云


```bash
hadoop fs -put ~/hadoop/data/book/input /data/book/input
```

上传到云后，我们可以用过命令 `hadoop fs -ls /data/book/input` 来检查是否都已进入云了。


### 3.3.4 执行 `WordCount` 示例程序


```bash
hadoop jar ~/hadoop/hadoop-1.2.1/hadoop-examples-1.2.1.jar wordcount /data/book/input /data/book/output
```

> 在执行过程中，可以新开一个 SSH 终端窗口连接到 `hadoop-master`，然后运行命令 `htop` 来观察任务执行期间的节点负载情况，包括内存占用率、CPU占用率、最消耗资源的程序等等。这些信息将来可能会作为云性能优化调整的依据。

执行结束后，会输出下面类似的输出：

```
14/02/18 20:26:29 INFO input.FileInputFormat: Total input paths to process : 201
14/02/18 20:26:29 INFO util.NativeCodeLoader: Loaded the native-hadoop library
14/02/18 20:26:29 WARN snappy.LoadSnappy: Snappy native library not loaded
14/02/18 20:26:30 INFO mapred.JobClient: Running job: job_201402190610_0002
14/02/18 20:26:31 INFO mapred.JobClient:  map 0% reduce 0%
14/02/18 20:26:42 INFO mapred.JobClient:  map 1% reduce 0%
14/02/18 20:26:45 INFO mapred.JobClient:  map 2% reduce 0%
...
14/02/18 20:30:32 INFO mapred.JobClient:  map 99% reduce 32%
14/02/18 20:30:34 INFO mapred.JobClient:  map 100% reduce 32%
14/02/18 20:30:40 INFO mapred.JobClient:  map 100% reduce 100%
14/02/18 20:30:41 INFO mapred.JobClient: Job complete: job_201402190610_0002
14/02/18 20:30:41 INFO mapred.JobClient: Counters: 29
14/02/18 20:30:41 INFO mapred.JobClient:   Job Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Launched reduce tasks=1
14/02/18 20:30:41 INFO mapred.JobClient:     SLOTS_MILLIS_MAPS=451625
14/02/18 20:30:41 INFO mapred.JobClient:     Total time spent by all reduces waiting after reserving slots (ms)=0
14/02/18 20:30:41 INFO mapred.JobClient:     Total time spent by all maps waiting after reserving slots (ms)=0
14/02/18 20:30:41 INFO mapred.JobClient:     Launched map tasks=201
14/02/18 20:30:41 INFO mapred.JobClient:     Data-local map tasks=201
14/02/18 20:30:41 INFO mapred.JobClient:     SLOTS_MILLIS_REDUCES=228055
14/02/18 20:30:41 INFO mapred.JobClient:   File Output Format Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Bytes Written=8206570
14/02/18 20:30:41 INFO mapred.JobClient:   FileSystemCounters
14/02/18 20:30:41 INFO mapred.JobClient:     FILE_BYTES_READ=41640360
14/02/18 20:30:41 INFO mapred.JobClient:     HDFS_BYTES_READ=92553354
14/02/18 20:30:41 INFO mapred.JobClient:     FILE_BYTES_WRITTEN=90941789
14/02/18 20:30:41 INFO mapred.JobClient:     HDFS_BYTES_WRITTEN=8206570
14/02/18 20:30:41 INFO mapred.JobClient:   File Input Format Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Bytes Read=92529611
14/02/18 20:30:41 INFO mapred.JobClient:   Map-Reduce Framework
14/02/18 20:30:41 INFO mapred.JobClient:     Map output materialized bytes=37792708
14/02/18 20:30:41 INFO mapred.JobClient:     Map input records=1868098
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce shuffle bytes=37792708
14/02/18 20:30:41 INFO mapred.JobClient:     Spilled Records=5323388
14/02/18 20:30:41 INFO mapred.JobClient:     Map output bytes=151082969
14/02/18 20:30:41 INFO mapred.JobClient:     Total committed heap usage (bytes)=27599167488
14/02/18 20:30:41 INFO mapred.JobClient:     CPU time spent (ms)=251850
14/02/18 20:30:41 INFO mapred.JobClient:     Combine input records=15649835
14/02/18 20:30:41 INFO mapred.JobClient:     SPLIT_RAW_BYTES=23743
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce input records=2528771
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce input groups=561390
14/02/18 20:30:41 INFO mapred.JobClient:     Combine output records=2636459
14/02/18 20:30:41 INFO mapred.JobClient:     Physical memory (bytes) snapshot=36463890432
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce output records=561390
14/02/18 20:30:41 INFO mapred.JobClient:     Virtual memory (bytes) snapshot=218530009088
14/02/18 20:30:41 INFO mapred.JobClient:     Map output records=15542147
```

如果有错误，会有异常报错。执行下面的命令确定正常执行完毕：

```bash
hadoop fs -ls /data/book/output/
```

该命令会列出我们指定的HDFS云中的输出目录内容：

```
Found 3 items
-rw-r--r--   1 tao supergroup          0 2014-02-18 20:30 /data/book/output/_SUCCESS
drwxr-xr-x   - tao supergroup          0 2014-02-18 20:26 /data/book/output/_logs
-rw-r--r--   1 tao supergroup    8206570 2014-02-18 20:30 /data/book/output/part-r-00000
```

注意其中若存在 `_SUCCESS` 文件，即说明执行成功了。统计的结果就在 `part-r-xxxxx` 这类文件里。可以拷贝到本地查看其内容，也可以通过 `http://hadoop-master:50070` 的 Web 界面来查看其内容。


3.4 停止 Hadoop
----------------


我们知道了怎么启动 Hadoop 云；知道了怎么运行 Hadoop 程序；接下来我们需要停止 Hadoop 云：

```bash
stop-all.sh
```

可以通过 `jps` 确定一下确实都关闭了。如果需要关机，则执行 `sudo poweroff` 即可。














