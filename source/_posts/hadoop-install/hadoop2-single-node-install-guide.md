---
layout: post
category: tutorial
title: Hadoop 2.2.0 单节点伪分布配置
tags : [hadoop, hadoop install]
date: 2014/02/22
---

前言
=============


本安装指南是针对 Ubuntu Server 12.04 进行 Hadoop 2.x 单节点伪分布模式配置的。其它系统或其它版本请酌情调整配置方法。

一、准备工作
================

我们需要有一台 Ubuntu Server 12.04 64位的虚拟机，如果尚未建立，请参考：【[在 VirtualBox 中安装 Ubuntu Server 12.04 64位版](/tutorial/hadoop-install/install-ubuntu-server-12.04-on-virtualbox.html)】 进行安装。

此外，我们还需要在这台虚拟机中为 Hadoop 的运行环境进行一些准备工作，具体的内容，请参考： 【[准备 Hadoop 运行环境](/tutorial/hadoop-install/prepare-hadoop-runtime-environment.html)】。

**需要注意的是，如果之前做了 Hadoop 1.x 的实验，那么为了避免IP冲突，我们需要定义一套新的各节点 IP 和主机名，在准备 Hadoop 运行环境的时候，参考下面定义的节点进行配置。**

```bash
10.0.1.210 hadoop2-master1
10.0.1.211 hadoop2-master2
10.0.1.221 hadoop2-data1
10.0.1.222 hadoop2-data2
10.0.1.223 hadoop2-data3
```

我们这里假定已经拥有了 Hadoop 2.2.0 的编译好的64位二进制包，`hadoop-2.2.0.tar.gz`，并且位于本机：`~/hadoop2/`目录下。如果尚未拥有编译好的安装包，请参考另一篇文章：【[编译 Hadoop 2.2](/tutorial/hadoop-install/compile-hadoop2.html)】。


二、安装 Hadoop
==================


2.1 解压缩 Hadoop 包
------------------


我们所有的Hadoop相关文件都会位于 `~/hadoop2`目录下。

```bash
cd ~/hadoop2
tar -zxvf hadoop-2.2.0.tar.gz
```

> 这里需要说明的是，许多教程都最终将 hadoop 放到了 `/usr/local/hadoop` 目录下，这样做会带来很多权限问题。而很多教程，或者最后干脆使用 `root` 用户操作一切，或者使用 `chown` 来解决权限冲突，这都是非常不好的，很多时候是因为写这些教程的人对 Linux 不熟悉，把 Windows 的一些习惯带到了 Linux 里来。生产环境的搭建绝不是这么简单粗暴的改变所有权，更不可能是这样子滥用`root`权限。除了需要将 hadoop 放到符合 Linux 目录结构的位置外，包括配置文件和日志，也都应该指向正确的位置，并且建立适当的用户组，以及对不同的目录使用不同的权限，这将引入很多额外的工作，因此不适合在初学时涉及。对于开发和实验环境的搭建，我们这里的做法非常简单，足以胜任后续的学习、开发。



2.2 配置 Hadoop
------------------


### 2.2.1 配置 `PATH` 环境变量


hadoop 的可执行文件在 `hadoop-2.2.0/bin` 下，我们如果想执行该文件，或者使用完整路径的方式，或者将该路径加入到 `PATH` 环境变量中，以后我们直接执行文件即可。

```bash
nano ~/.profile
```

在最后一行添加：

```bash
export PATH=$PATH:$HOME/hadoop2/hadoop-2.2.0/bin:$HOME/hadoop2/hadoop-2.2.0/sbin
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

为方便起见，我们将当前目录换到 `etc/hadoop/` 目录下：

```bash
cd hadoop-2.2.0/etc/hadoop
```

#### `core-site.xml`:

```bash
nano core-site.xml
```

在 `<configuration>`中加入：

```xml
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop2-master1:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>${user.home}/hadoop2/tmp</value>
    </property>
```

- `fs.defaultFS`： 在 1.x 中，该项名为 `fs.default.name`，在 2.x 中改名为 `fs.defaultFS`。通过这里指定 NameNode 的位置。我们可以在这里手动指定端口号，也可以忽略以使用默认的端口号。
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


#### `yarn-site.xml`:


```bash
nano yarn-site.xml
```

在 `<configuration>`中加入：

```xml
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>hadoop2-master1</value>
    </property>
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>2048</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.remote-app-log-dir</name>
        <value>${hadoop.tmp.dir}/nm-remote-app-logs</value>
    </property>
```

分别保存退出。

三、操作伪分布集群
===================

3.1 格式化 NameNode
-------------------


在配置完成后，我们需要进行一次 NameNode 的格式化。如果未格式化，NameNode 会因 HDFS 所需的数据结构错误而无法启动。

```bash
hdfs namenode -format
```

> 原有的 `hadoop` 命令已不推荐使用，使用 `hdfs` 命令代替。

格式化之后，应该会出现与下面相似的输出：

```bash
14/02/22 21:45:11 INFO namenode.NameNode: STARTUP_MSG: 
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = hadoop2-master/10.0.1.210
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.2.0
STARTUP_MSG:   classpath = /home/tao/hadoop2/hadoop-2.2.0/etc/hadoop:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jasper-compiler-5.5.23.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jasper-runtime-5.5.23.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jersey-server-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-beanutils-1.7.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jackson-core-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jsch-0.1.42.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/xz-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/zookeeper-3.4.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/xmlenc-0.52.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-lang-2.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/protobuf-java-2.5.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jsr305-1.3.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-compress-1.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-net-3.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/hadoop-auth-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/log4j-1.2.17.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-cli-1.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jersey-core-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jackson-xc-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-logging-1.1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jetty-6.1.26.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-collections-3.2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/servlet-api-2.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jackson-mapper-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/paranamer-2.3.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jsp-api-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/stax-api-1.0.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/hadoop-annotations-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/guava-11.0.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jaxb-api-2.2.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/snappy-java-1.0.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-io-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jettison-1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-digester-1.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jackson-jaxrs-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-codec-1.4.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-configuration-1.6.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/avro-1.7.4.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-httpclient-3.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jersey-json-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-el-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/netty-3.6.2.Final.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jetty-util-6.1.26.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/mockito-all-1.8.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jets3t-0.6.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/activation-1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/jaxb-impl-2.2.3-1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/junit-4.8.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/slf4j-api-1.7.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-beanutils-core-1.8.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/commons-math-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/lib/asm-3.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/hadoop-common-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/hadoop-common-2.2.0-tests.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/common/hadoop-nfs-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jasper-runtime-5.5.23.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jersey-server-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jackson-core-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/xmlenc-0.52.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-lang-2.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/protobuf-java-2.5.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jsr305-1.3.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/log4j-1.2.17.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-cli-1.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jersey-core-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-logging-1.1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jetty-6.1.26.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/servlet-api-2.5.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jackson-mapper-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jsp-api-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/guava-11.0.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-io-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-codec-1.4.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-daemon-1.0.13.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/commons-el-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/netty-3.6.2.Final.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/jetty-util-6.1.26.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/lib/asm-3.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/hadoop-hdfs-nfs-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/hadoop-hdfs-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/hdfs/hadoop-hdfs-2.2.0-tests.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/jersey-server-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/jackson-core-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/xz-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/protobuf-java-2.5.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/guice-3.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/junit-4.10.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/commons-compress-1.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/aopalliance-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/log4j-1.2.17.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/guice-servlet-3.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/jersey-core-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/jackson-mapper-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/paranamer-2.3.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/hadoop-annotations-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/jersey-guice-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/snappy-java-1.0.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/commons-io-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/avro-1.7.4.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/netty-3.6.2.Final.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/hamcrest-core-1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/javax.inject-1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/lib/asm-3.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-server-web-proxy-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-client-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-server-nodemanager-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-server-tests-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-site-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-common-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-applications-unmanaged-am-launcher-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-server-common-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-api-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/yarn/hadoop-yarn-server-resourcemanager-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/jersey-server-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/jackson-core-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/xz-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/protobuf-java-2.5.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/guice-3.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/junit-4.10.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/commons-compress-1.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/aopalliance-1.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/log4j-1.2.17.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/guice-servlet-3.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/jersey-core-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/jackson-mapper-asl-1.8.8.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/paranamer-2.3.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/hadoop-annotations-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/jersey-guice-1.9.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/snappy-java-1.0.4.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/commons-io-2.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/avro-1.7.4.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/netty-3.6.2.Final.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/hamcrest-core-1.1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/javax.inject-1.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/lib/asm-3.2.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.2.0-tests.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-plugins-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-app-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-shuffle-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.2.0.jar:/home/tao/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.2.0.jar:/contrib/capacity-scheduler/*.jar
STARTUP_MSG:   build = Unknown -r Unknown; compiled by 'tao' on 2014-02-22T06:48Z
STARTUP_MSG:   java = 1.7.0_51
************************************************************/
14/02/22 21:45:11 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT]
Formatting using clusterid: CID-0107de99-dfbd-4142-b321-72c40297b8fd
14/02/22 21:45:12 INFO namenode.HostFileManager: read includes:
HostSet(
)
14/02/22 21:45:12 INFO namenode.HostFileManager: read excludes:
HostSet(
)
14/02/22 21:45:12 INFO blockmanagement.DatanodeManager: dfs.block.invalidate.limit=1000
14/02/22 21:45:12 INFO util.GSet: Computing capacity for map BlocksMap
14/02/22 21:45:12 INFO util.GSet: VM type       = 64-bit
14/02/22 21:45:12 INFO util.GSet: 2.0% max memory = 966.7 MB
14/02/22 21:45:12 INFO util.GSet: capacity      = 2^21 = 2097152 entries
14/02/22 21:45:12 INFO blockmanagement.BlockManager: dfs.block.access.token.enable=false
14/02/22 21:45:12 INFO blockmanagement.BlockManager: defaultReplication         = 1
14/02/22 21:45:12 INFO blockmanagement.BlockManager: maxReplication             = 512
14/02/22 21:45:12 INFO blockmanagement.BlockManager: minReplication             = 1
14/02/22 21:45:12 INFO blockmanagement.BlockManager: maxReplicationStreams      = 2
14/02/22 21:45:12 INFO blockmanagement.BlockManager: shouldCheckForEnoughRacks  = false
14/02/22 21:45:12 INFO blockmanagement.BlockManager: replicationRecheckInterval = 3000
14/02/22 21:45:12 INFO blockmanagement.BlockManager: encryptDataTransfer        = false
14/02/22 21:45:12 INFO namenode.FSNamesystem: fsOwner             = tao (auth:SIMPLE)
14/02/22 21:45:12 INFO namenode.FSNamesystem: supergroup          = supergroup
14/02/22 21:45:12 INFO namenode.FSNamesystem: isPermissionEnabled = true
14/02/22 21:45:12 INFO namenode.FSNamesystem: HA Enabled: false
14/02/22 21:45:12 INFO namenode.FSNamesystem: Append Enabled: true
14/02/22 21:45:12 INFO util.GSet: Computing capacity for map INodeMap
14/02/22 21:45:12 INFO util.GSet: VM type       = 64-bit
14/02/22 21:45:12 INFO util.GSet: 1.0% max memory = 966.7 MB
14/02/22 21:45:12 INFO util.GSet: capacity      = 2^20 = 1048576 entries
14/02/22 21:45:12 INFO namenode.NameNode: Caching file names occuring more than 10 times
14/02/22 21:45:12 INFO namenode.FSNamesystem: dfs.namenode.safemode.threshold-pct = 0.9990000128746033
14/02/22 21:45:12 INFO namenode.FSNamesystem: dfs.namenode.safemode.min.datanodes = 0
14/02/22 21:45:12 INFO namenode.FSNamesystem: dfs.namenode.safemode.extension     = 30000
14/02/22 21:45:12 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
14/02/22 21:45:12 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
14/02/22 21:45:12 INFO util.GSet: Computing capacity for map Namenode Retry Cache
14/02/22 21:45:12 INFO util.GSet: VM type       = 64-bit
14/02/22 21:45:12 INFO util.GSet: 0.029999999329447746% max memory = 966.7 MB
14/02/22 21:45:12 INFO util.GSet: capacity      = 2^15 = 32768 entries
14/02/22 21:45:12 INFO common.Storage: Storage directory /home/tao/hadoop2/tmp/dfs/name has been successfully formatted.
14/02/22 21:45:12 INFO namenode.FSImage: Saving image file /home/tao/hadoop2/tmp/dfs/name/current/fsimage.ckpt_0000000000000000000 using no compression
14/02/22 21:45:12 INFO namenode.FSImage: Image file /home/tao/hadoop2/tmp/dfs/name/current/fsimage.ckpt_0000000000000000000 of size 195 bytes saved in 0 seconds.
14/02/22 21:45:12 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
14/02/22 21:45:12 INFO util.ExitUtil: Exiting with status 0
14/02/22 21:45:12 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at hadoop2-master/10.0.1.210
************************************************************/
```

特别是关注其中是否报告格式化成功：

```
Storage directory /home/tao/hadoop2/tmp/dfs/name has been successfully formatted.
```

如果没有，则需要根据错误信息进行分析。

至此，Hadoop 单节点伪分布模式就配置完毕了。



3.2 启动 Hadoop
----------------


```bash
start-dfs.sh
start-yarn.sh
```

这条命令将根据之前配置的 `*-site.xml`，来启动 Hadoop 云（当然，此时，我们这个云是伪分布，就一个节点）。

> 这里与 1.x 不同，`start-all.sh` 已不推荐使用，推荐对 `HDFS` 和 `YARN` 分开操作。

然后，我们需要确定节点是否正常运行了。

```bash
jps
```

这条命令可以帮助我们查看当前系统运行的 Java 程序进程。如果配置正确，我们将看到类似下面的结果：

```bash
28264 ResourceManager
28671 Jps
28365 NodeManager
28122 SecondaryNameNode
27972 DataNode
27854 NameNode
```

需要注意的是，单节点伪分布正确运行后，应该会有5个组件运行：`NameNode`、`DataNode`、`SecondaryNameNode`、`ResourceManager`、以及`NodeManager`。缺少任何一个，都说明配置上有所错误。需要回去检查各项配置。

> 在出现故障后向论坛、朋友求助的时候，需要给对方提供你的 `conf` 目录下的配置文件、以及日志，否则对方无法理解你的故障可能的问题。日志目录：`$HOME/hadoop/hadoop-2.2.0/logs`


3.3 运行示例程序
---------------

就如同我们搭建好其它语言的编译环境后，会编写一个 `HelloWorld` 程序来测试整个环境是否工作一样，在 Hadoop 中，我们有自己的`HelloWorld`类的程序，叫做`WordCount`。

`ＷordCount` 是对纯文本进行单词出现次数统计的，为了使用 `WordCount`，我们需要先准备一些数据，也就是一些文本文件。这里，我们使用 [古腾堡计划](http://www.gutenberg.org/)中的英文图书。


### 3.3.1 下载


```bash
mkdir -p ~/hadoop2/data/book/download
cd ~/hadoop2/data/book/download
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
for f in `find ~/hadoop2/data/book/download -name *.zip` ; do unzip -j $f *.txt -d ~/hadoop2/data/book/input; done
```


### 3.3.3 将书籍放到 HDFS 云


```bash
hdfs dfs -mkdir -p /data/book
hdfs dfs -put ~/hadoop2/data/book/input /data/book
```

上传到云后，我们可以用过命令 `hdfs dfs -ls /data/book/input` 来检查是否都已进入云了。

> 原有的 `hadoop fs` 已不推荐使用，使用 `hdfs dfs` 代替。


### 3.3.4 执行 `WordCount` 示例程序


```bash
hadoop jar ~/hadoop2/hadoop-2.2.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.2.0.jar wordcount /data/book/input /data/book/output
```

> 在执行过程中，可以新开一个 SSH 终端窗口连接到 `hadoop-master1`，然后运行命令 `htop` 来观察任务执行期间的节点负载情况，包括内存占用率、CPU占用率、最消耗资源的程序等等。这些信息将来可能会作为云性能优化调整的依据。

执行结束后，会输出类似下面的输出：

```
14/02/22 22:58:52 INFO Configuration.deprecation: session.id is deprecated. Instead, use dfs.metrics.session-id
14/02/22 22:58:52 INFO jvm.JvmMetrics: Initializing JVM Metrics with processName=JobTracker, sessionId=
14/02/22 22:58:52 INFO input.FileInputFormat: Total input paths to process : 201
14/02/22 22:58:52 INFO mapreduce.JobSubmitter: number of splits:201
14/02/22 22:58:52 INFO Configuration.deprecation: user.name is deprecated. Instead, use mapreduce.job.user.name
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.jar is deprecated. Instead, use mapreduce.job.jar
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.output.value.class is deprecated. Instead, use mapreduce.job.output.value.class
14/02/22 22:58:52 INFO Configuration.deprecation: mapreduce.combine.class is deprecated. Instead, use mapreduce.job.combine.class
14/02/22 22:58:52 INFO Configuration.deprecation: mapreduce.map.class is deprecated. Instead, use mapreduce.job.map.class
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.job.name is deprecated. Instead, use mapreduce.job.name
14/02/22 22:58:52 INFO Configuration.deprecation: mapreduce.reduce.class is deprecated. Instead, use mapreduce.job.reduce.class
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.input.dir is deprecated. Instead, use mapreduce.input.fileinputformat.inputdir
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.output.dir is deprecated. Instead, use mapreduce.output.fileoutputformat.outputdir
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.map.tasks is deprecated. Instead, use mapreduce.job.maps
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.output.key.class is deprecated. Instead, use mapreduce.job.output.key.class
14/02/22 22:58:52 INFO Configuration.deprecation: mapred.working.dir is deprecated. Instead, use mapreduce.job.working.dir
14/02/22 22:58:53 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_local693675167_0001
14/02/22 22:58:53 WARN conf.Configuration: file:/home/tao/hadoop2/hdfs-tmp/mapred/staging/tao693675167/.staging/job_local693675167_0001/job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.retry.interval;  Ignoring.
14/02/22 22:58:53 WARN conf.Configuration: file:/home/tao/hadoop2/hdfs-tmp/mapred/staging/tao693675167/.staging/job_local693675167_0001/job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.attempts;  Ignoring.
14/02/22 22:58:53 WARN conf.Configuration: file:/home/tao/hadoop2/hdfs-tmp/mapred/local/localRunner/tao/job_local693675167_0001/job_local693675167_0001.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.retry.interval;  Ignoring.
14/02/22 22:58:53 WARN conf.Configuration: file:/home/tao/hadoop2/hdfs-tmp/mapred/local/localRunner/tao/job_local693675167_0001/job_local693675167_0001.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.attempts;  Ignoring.
14/02/22 22:58:53 INFO mapreduce.Job: The url to track the job: http://localhost:8080/
14/02/22 22:58:53 INFO mapreduce.Job: Running job: job_local693675167_0001
14/02/22 22:58:53 INFO mapred.LocalJobRunner: OutputCommitter set in config null
14/02/22 22:58:53 INFO mapred.LocalJobRunner: OutputCommitter is org.apache.hadoop.mapreduce.lib.output.FileOutputCommitter
14/02/22 22:58:53 INFO mapred.LocalJobRunner: Starting task: attempt_local693675167_0001_m_000000_0
14/02/22 22:58:53 INFO mapred.LocalJobRunner: Waiting for map tasks
14/02/22 22:58:53 INFO mapred.Task:  Using ResourceCalculatorProcessTree : [ ]
14/02/22 22:58:53 INFO mapred.MapTask: Processing split: hdfs://hadoop2-master1:9000/data/book/input/00ws110.txt:0+4651867
14/02/22 22:58:53 INFO mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
14/02/22 22:58:53 INFO mapred.MapTask: (EQUATOR) 0 kvi 26214396(104857584)
14/02/22 22:58:53 INFO mapred.MapTask: mapreduce.task.io.sort.mb: 100
14/02/22 22:58:53 INFO mapred.MapTask: soft limit at 83886080
14/02/22 22:58:53 INFO mapred.MapTask: bufstart = 0; bufvoid = 104857600
14/02/22 22:58:53 INFO mapred.MapTask: kvstart = 26214396; length = 6553600
14/02/22 22:58:54 INFO mapred.LocalJobRunner: 
14/02/22 22:58:54 INFO mapred.MapTask: Starting flush of map output
14/02/22 22:58:54 INFO mapred.MapTask: Spilling map output
14/02/22 22:58:54 INFO mapred.MapTask: bufstart = 0; bufend = 7680316; bufvoid = 104857600
14/02/22 22:58:54 INFO mapred.MapTask: kvstart = 26214396(104857584); kvend = 22943156(91772624); length = 3271241/6553600
14/02/22 22:58:54 INFO mapreduce.Job: Job job_local693675167_0001 running in uber mode : false
14/02/22 22:58:54 INFO mapreduce.Job:  map 0% reduce 0%
14/02/22 22:58:55 INFO mapred.MapTask: Finished spill 0
14/02/22 22:58:55 INFO mapred.Task: Task:attempt_local693675167_0001_m_000000_0 is done. And is in the process of committing

...

14/02/22 23:01:23 INFO mapred.LocalJobRunner: Finishing task: attempt_local827466818_0001_m_000200_0
14/02/22 23:01:23 INFO mapred.LocalJobRunner: Map task executor complete.
14/02/22 23:01:23 INFO mapred.Task:  Using ResourceCalculatorProcessTree : [ ]
14/02/22 23:01:23 INFO mapred.Merger: Merging 201 sorted segments
14/02/22 23:01:23 INFO mapred.Merger: Merging 3 intermediate segments out of a total of 201
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 199
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 190
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 181
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 172
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 163
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 154
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 145
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 136
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 127
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 118
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 109
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 100
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 91
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 82
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 73
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 64
14/02/22 23:01:24 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 55
14/02/22 23:01:25 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 46
14/02/22 23:01:25 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 37
14/02/22 23:01:25 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 28
14/02/22 23:01:25 INFO mapred.Merger: Merging 10 intermediate segments out of a total of 19
14/02/22 23:01:25 INFO mapred.Merger: Down to the last merge-pass, with 10 segments left of total size: 37524727 bytes
14/02/22 23:01:25 INFO mapred.LocalJobRunner: 
14/02/22 23:01:25 INFO Configuration.deprecation: mapred.skip.on is deprecated. Instead, use mapreduce.job.skiprecords
14/02/22 23:01:27 INFO mapred.Task: Task:attempt_local827466818_0001_r_000000_0 is done. And is in the process of committing
14/02/22 23:01:27 INFO mapred.LocalJobRunner: 
14/02/22 23:01:27 INFO mapred.Task: Task attempt_local827466818_0001_r_000000_0 is allowed to commit now
14/02/22 23:01:27 INFO output.FileOutputCommitter: Saved output of task 'attempt_local827466818_0001_r_000000_0' to hdfs://hadoop2-master1:9000/data/book/output/_temporary/0/task_local827466818_0001_r_000000
14/02/22 23:01:27 INFO mapred.LocalJobRunner: reduce > reduce
14/02/22 23:01:27 INFO mapred.Task: Task 'attempt_local827466818_0001_r_000000_0' done.
14/02/22 23:01:27 INFO mapreduce.Job:  map 100% reduce 100%
14/02/22 23:01:27 INFO mapreduce.Job: Job job_local827466818_0001 completed successfully
14/02/22 23:01:27 INFO mapreduce.Job: Counters: 32
	File System Counters
		FILE: Number of bytes read=228369909
		FILE: Number of bytes written=5350292504
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=13902047117
		HDFS: Number of bytes written=8206570
		HDFS: Number of read operations=41815
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=204
	Map-Reduce Framework
		Map input records=1868098
		Map output records=15542147
		Map output bytes=151082969
		Map output materialized bytes=37525957
		Input split bytes=23944
		Combine input records=15542147
		Combine output records=2509502
		Reduce input groups=561390
		Reduce shuffle bytes=0
		Reduce input records=2509502
		Reduce output records=561390
		Spilled Records=8275287
		Shuffled Maps =0
		Failed Shuffles=0
		Merged Map outputs=0
		GC time elapsed (ms)=3524
		CPU time spent (ms)=0
		Physical memory (bytes) snapshot=0
		Virtual memory (bytes) snapshot=0
		Total committed heap usage (bytes)=105619390464
	File Input Format Counters 
		Bytes Read=92529611
	File Output Format Counters 
		Bytes Written=8206570
```

如果有错误，会有异常报错。执行下面的命令确定正常执行完毕：

```bash
hdfs dfs -ls /data/book/output/
```

该命令会列出我们指定的HDFS云中的输出目录内容：

```
Found 2 items
-rw-r--r--   1 tao supergroup          0 2014-02-22 23:01 /data/book/output/_SUCCESS
-rw-r--r--   1 tao supergroup    8206570 2014-02-22 23:01 /data/book/output/part-r-00000
```

注意其中若存在 `_SUCCESS` 文件，即说明执行成功了。统计的结果就在 `part-r-xxxxx` 这类文件里。可以拷贝到本地查看其内容，也可以通过 `http://hadoop-master1:50070` 的 Web 界面来查看其内容。


3.4 停止 Hadoop
----------------

我们知道了怎么启动 Hadoop 云；知道了怎么运行 Hadoop 程序；接下来我们需要停止 Hadoop 云：

```bash
stop-yarn.sh
stop-dfs.sh
```

可以通过 `jps` 确定一下确实都关闭了。如果需要关机，则执行 `sudo poweroff` 即可。














