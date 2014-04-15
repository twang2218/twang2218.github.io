---
layout: post
category: tutorial
title: 编译 Hadoop 2.2.0
tags : [hadoop, hadoop install]
date: 2014/02/20
---


前言
=====


Hadoop 2.2 所提供的二进制包是针对32位系统进行编译的，其中并未像1.x那样提供了64位的原生库文件。

如果直接使用该二进制包安装，会导致无法使用原生库进行加速。

执行过程中可能会出现如下的警告：

```bash
OpenJDK 64-Bit Server VM warning: You have loaded library /home/tao/hadoop2/t/hadoop-2.2.0/lib/native/libhadoop.so.1.0.0 which might have disabled stack guard. The VM will try to fix the stack guard now.
It's highly recommended that you fix the library with 'execstack -c <libfile>', or link it with '-z noexecstack'
14/02/22 23:38:47 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
```

虽然可以运行，但是性能可能会受到影响，而且警告信息也比较干扰。因此建议从源代码编译该软件。对于生产环境而言，则可以直接使用 CDH 这类成熟的发布版本，其中已包涵编译好的64位的二进制原生库，因此不用自己编译。

**尽量不要选择在生产服务器进行编译，可以在任意一台64位的 Ubuntu 的机器上进行编译，编译后将生成用于安装的 `hadoop-2.2.0.tar.gz` 文件，将其复制到目标服务器上的`~/hadoop2`目录即可。**


一、下载 Hadoop 源代码
=======================


我们将编译当前 2.x 系列的稳定版本 2.2.0，访问官网：http://hadoop.apache.org 

`Download Hadoop` ⇨ `releases` ⇨ `Download` ⇨ `Download a release now!` ⇨ 选择合适的镜像 ⇨ `hadoop-2.2.0/` ⇨ 我们将下载 `hadoop-2.2.0-src.tar.gz` 这个文件。

我们可以用浏览器下载后，上传到 Ubuntu 的机器上，也可以像下面这样，直接在 Ubuntu 下载该文件。

假设我们当前在一台用于编译的 Ubuntu 64位的机器上，首先将创建一个 `~/hadoop2` 目录，然后在这个目录里进行编译。

```bash
mkdir -p ~/hadoop2
cd ~/hadoop2
wget http://apache.dataguru.cn/hadoop/common/hadoop-2.2.0/hadoop-2.2.0-src.tar.gz
```


如果网络环境较差，则需要检查下载的文件是否正确。执行 


```bash
sha1sum hadoop-2.2.0-src.tar.gz
```

如果输出和下面的内容一样，则说明下载正确了。

```
dac2c5773080c8a4b8fb3ce306df1c742351c6f2  hadoop-2.2.0-src.tar.gz
```

如果下载正确，则对压缩文件进行解压缩：

```bash
tar -zxvf hadoop-2.2.0-src.tar.gz
```

执行上述命令后，会生成 `hadoop-2.2.0-src` 目录，我们可以继续进行编译了。


二、编译 Hadoop 
================


2.1 安装所需的软件包
---------------------


首先需要安装构建所需的软件包。

```bash
sudo apt-get install build-essential openjdk-7-jdk maven subversion git cmake zlib1g-dev libssl-dev pkg-config python-software-properties curl patch
```

除此以外，我们还需要 `protobuf 2.5`，我们可以直接从 `PPA` 上安装编译好的包：

```bash
sudo add-apt-repository ppa:chris-lea/protobuf
sudo apt-get update && sudo apt-get install protobuf-compiler libprotobuf7
```

2.2 打补丁
-----------


官方带的源代码中有几个 bug，需要在编译前先打上布丁。

当然，在此之前，我们要先进入源代码目录：

```bash
cd ~/hadoop2/hadoop-2.2.0-src
```


### AbstractLifeCycle not found


针对错误：

```bash
cannot access org.mortbay.component.AbstractLifeCycle
[ERROR] class file for org.mortbay.component.AbstractLifeCycle not found
```

需要打补丁：

```bash
curl https://issues.apache.org/jira/secure/attachment/12614482/HADOOP-10110.patch | patch -u -p0
```


### Failed tests: testNotifyRetries  


如果开启测试 (没有使用 `-DskipTest` 参数)，则可能触发测试代码中的一个错误：

```bash
Failed tests: testNotifyRetries(org.apache.hadoop.mapreduce.v2.app.TestJobEndNotifier): Should have taken more than 5 seconds it took 1803
```

因此需要打下一个补丁：

```bash
curl https://issues.apache.org/jira/secure/attachment/12614696/MAPREDUCE-5631.patch | patch -p0
```


2.3 编译
---------


```bash
mvn clean package -Pdist,native -DskipTests -Dtar
```

> 如果编译出错，需要先 `mvn -rf:xxxx` 删除最后未完成的一项。

编译成功后，会出现下列内容：

```bash
[INFO] Reactor Summary:
[INFO] 
[INFO] Apache Hadoop Main ................................ SUCCESS [0.966s]
[INFO] Apache Hadoop Project POM ......................... SUCCESS [0.678s]
[INFO] Apache Hadoop Annotations ......................... SUCCESS [2.810s]
[INFO] Apache Hadoop Assemblies .......................... SUCCESS [0.201s]
[INFO] Apache Hadoop Project Dist POM .................... SUCCESS [1.639s]
[INFO] Apache Hadoop Maven Plugins ....................... SUCCESS [2.251s]
[INFO] Apache Hadoop Auth ................................ SUCCESS [2.712s]
[INFO] Apache Hadoop Auth Examples ....................... SUCCESS [1.990s]
[INFO] Apache Hadoop Common .............................. SUCCESS [1:00.553s]
[INFO] Apache Hadoop NFS ................................. SUCCESS [6.313s]
[INFO] Apache Hadoop Common Project ...................... SUCCESS [0.028s]
[INFO] Apache Hadoop HDFS ................................ SUCCESS [58.046s]
[INFO] Apache Hadoop HttpFS .............................. SUCCESS [12.247s]
[INFO] Apache Hadoop HDFS BookKeeper Journal ............. SUCCESS [10.676s]
[INFO] Apache Hadoop HDFS-NFS ............................ SUCCESS [2.319s]
[INFO] Apache Hadoop HDFS Project ........................ SUCCESS [0.032s]
[INFO] hadoop-yarn ....................................... SUCCESS [0.693s]
[INFO] hadoop-yarn-api ................................... SUCCESS [25.461s]
[INFO] hadoop-yarn-common ................................ SUCCESS [16.745s]
[INFO] hadoop-yarn-server ................................ SUCCESS [0.078s]
[INFO] hadoop-yarn-server-common ......................... SUCCESS [4.837s]
[INFO] hadoop-yarn-server-nodemanager .................... SUCCESS [9.803s]
[INFO] hadoop-yarn-server-web-proxy ...................... SUCCESS [2.433s]
[INFO] hadoop-yarn-server-resourcemanager ................ SUCCESS [6.797s]
[INFO] hadoop-yarn-server-tests .......................... SUCCESS [0.283s]
[INFO] hadoop-yarn-client ................................ SUCCESS [2.877s]
[INFO] hadoop-yarn-applications .......................... SUCCESS [0.048s]
[INFO] hadoop-yarn-applications-distributedshell ......... SUCCESS [2.234s]
[INFO] hadoop-mapreduce-client ........................... SUCCESS [0.047s]
[INFO] hadoop-mapreduce-client-core ...................... SUCCESS [13.074s]
[INFO] hadoop-yarn-applications-unmanaged-am-launcher .... SUCCESS [1.545s]
[INFO] hadoop-yarn-site .................................. SUCCESS [0.079s]
[INFO] hadoop-yarn-project ............................... SUCCESS [3.591s]
[INFO] hadoop-mapreduce-client-common .................... SUCCESS [9.824s]
[INFO] hadoop-mapreduce-client-shuffle ................... SUCCESS [2.262s]
[INFO] hadoop-mapreduce-client-app ....................... SUCCESS [6.073s]
[INFO] hadoop-mapreduce-client-hs ........................ SUCCESS [3.176s]
[INFO] hadoop-mapreduce-client-jobclient ................. SUCCESS [6.431s]
[INFO] hadoop-mapreduce-client-hs-plugins ................ SUCCESS [2.073s]
[INFO] Apache Hadoop MapReduce Examples .................. SUCCESS [3.805s]
[INFO] hadoop-mapreduce .................................. SUCCESS [3.051s]
[INFO] Apache Hadoop MapReduce Streaming ................. SUCCESS [3.258s]
[INFO] Apache Hadoop Distributed Copy .................... SUCCESS [5.818s]
[INFO] Apache Hadoop Archives ............................ SUCCESS [3.203s]
[INFO] Apache Hadoop Rumen ............................... SUCCESS [4.007s]
[INFO] Apache Hadoop Gridmix ............................. SUCCESS [3.283s]
[INFO] Apache Hadoop Data Join ........................... SUCCESS [2.112s]
[INFO] Apache Hadoop Extras .............................. SUCCESS [2.453s]
[INFO] Apache Hadoop Pipes ............................... SUCCESS [7.199s]
[INFO] Apache Hadoop Tools Dist .......................... SUCCESS [1.787s]
[INFO] Apache Hadoop Tools ............................... SUCCESS [0.020s]
[INFO] Apache Hadoop Distribution ........................ SUCCESS [26.669s]
[INFO] Apache Hadoop Client .............................. SUCCESS [6.453s]
[INFO] Apache Hadoop Mini-Cluster ........................ SUCCESS [0.469s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 5:58.487s
[INFO] Finished at: Sat Feb 22 17:54:05 EST 2014
[INFO] Final Memory: 135M/360M
[INFO] ------------------------------------------------------------------------
```

编译后，会生成我们所需要的 `.tar.gz` 安装包。位于 `hadoop-2.2.0-src/hadoop-dist/target/hadoop-2.2.0.tar.gz`。

为了之后操作方便，我们将其复制到 `~/hadoop2` 目录下：

```bash
cp hadoop-dist/target/hadoop-2.2.0.tar.gz ~/hadoop2/
```

三、复制到指定机器
==================


编译好后，我们需要将编译好的这个 Hadoop 2.2.0 的安装包，复制到即将安装 Hadoop 的节点上指定的位置。

假设我们需要将该安装包复制到 `hadoop2-master1` 主机上，我们可以使用 `scp` 命令做跨主机的文件复制。

```bash
ssh hadoop2-master1 "mkdir -p hadoop2"
scp ~/hadoop-2.2.0.tar.gz hadoop2-master1:hadoop2/
```

第一行是试图建立 `~/hadoop2` 目录，以防万一目标主机中没有这个目录。第二行则是进行文件复制。










