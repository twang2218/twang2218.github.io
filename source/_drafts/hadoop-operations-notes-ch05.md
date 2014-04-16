---
layout: post
category: readings
title: 《Hadoop Operations》读书笔记 - 4 - 第五章 安装与配置
tags : [hadoop, Hadoop Operations]
date: 2014/04/10
---
> **Chapter 5: Installation and Configuration**
> Eric Sammer ["Hadoop Operations" - O'Reilly (2012)](http://shop.oreilly.com/product/0636920025085.do) ... (p75 ~ p134)

安装 Hadoop
=============

有无数种办法可以安装 Hadoop，这里给出的只是最佳实践的建议。

对于 tarball 安装来说，拥有很大的灵活性，但同样也带来了很多不确定性。作为管理员需要为其额外的创建用户，以及准备各种目录，配置各种目录的权限。**如果不确定自己应该使用哪种安装方式，应该先从软件源或者 RPM/Deb 软件包安装开始。**

Hadoop 的运行不需要使用 root 权限。但是安装的时候，需要创建用户、调整个目录的权限、配置启动脚本等，因此安装时候一般会使用 root 权限，特别是如果需要 secure mode 时。但是一定要明白，**使用 root 安装，绝不意味着就使用 root 运行。** 

Apache Hadoop
---------------

Apache Hadoop 是 ASF 官方发布的 Hadoop 发行版本。可以直接从 http://hadoop.apache.org 下载。有 tarball 文件，以及 RPM 和 Deb 文件可供安装选择。

如果管理员有极特殊的需求的话，可能会考虑使用 tarball 文件安装。除此以外，一般都会选择使用 RPM/Deb安装方式。因为通过安装包安装的会遵循Linux标准文件结构。

### tarball 的安装

> 这里使用的是当前 1.x 稳定版本 1.2.1，书中使用的是 1.0.0。

首先是下载 hadoop：

```bash
wget http://<selected-mirror>/hadoop/core/hadoop-1.2.1/hadoop-1.2.1.tar.gz
```

解压缩

```bash
vagrant@precise64:~/hadoop$ tar -zxvf hadoop-1.2.1.tar.gz
...
vagrant@precise64:~/hadoop$ ls hadoop-1.2.1
bin          hadoop-ant-1.2.1.jar          ivy          sbin
build.xml    hadoop-client-1.2.1.jar       ivy.xml      share
c++          hadoop-core-1.2.1.jar         lib          src
CHANGES.txt  hadoop-examples-1.2.1.jar     libexec      webapps
conf         hadoop-minicluster-1.2.1.jar  LICENSE.txt
contrib      hadoop-test-1.2.1.jar         NOTICE.txt
docs         hadoop-tools-1.2.1.jar        README.txt
vagrant@precise64:~/hadoop$
```

可以看到一些常见的目录：

- `bin`, `sbin`, `lib`, `libexec`, `share`；
- `conf` 这个目录包含了配置文件；
- Hadoop的主要代码都在 `hadoop-core-1.2.1.jar` 以及 `hadoop-tools-1.2.1.jar`；
- `src` 这个目录包含了 Hadoop 的源代码，参考目的；

很奇怪的是，在 1.2.1 的版本中有不少文件是可写的，可写也就意味着可以被人篡改，这显然是不正确的。

如果需要检查目录里是否存在除了 owner 以外拥有的写权限，可以使用下面的命令：

```bash
find hadoop-1.2.1 -type f -perm -g+w
find hadoop-1.2.1 -type f -perm -o+w
```

一般管理员会禁止这些可写，使用下面的命令。

```bash
chmod -R g-w hadoop-1.2.1
```

对于测试和实验来说，hadoop 留在自己的 home 目录没有任何问题。对于生产环境来说，需要移动到符合系统规范的地方：

```bash
sudo mv hadoop-1.2.1 /usr/local/
```

移动之后，不要忘记了修改文件的所有权，因为生产环境可不是为了给安装用户用的：

```bash
cd /usr/local
sudo chown -R root:root hadoop-1.2.1/
```

默认情况下 `conf` 配置文件在 Hadoop 目录下，我们可以让其留在那里。但是正常情况下，会考虑将其移动到标准的 `/etc` 目录下。这样很多系统方面的工具都可以使用。比如 IDS 的 Tripwire、配置备份，以及将来升级Hadoop软件不必担心配置文件被覆盖掉。

```bash
vagrant@precise64:/usr/local$ sudo mkdir /etc/hadoop/
vagrant@precise64:/usr/local$ cd hadoop-1.2.1/
vagrant@precise64:/usr/local/hadoop-1.2.1$ sudo mv conf /etc/hadoop/conf-1.2.1
vagrant@precise64:/usr/local/hadoop-1.2.1$ sudo ln -s /etc/hadoop/conf-1.2.1/ conf
```

### 安装包安装

使用 rpm 或者 deb 安装包则要比 tarball 简单许多。不过，同样需要一些额外的关注。因为 Apache Hadoop 发行版本没有针对 CentOS, RHEL, Ubuntu 之类系统进行定制，因此有可能会有依赖冲突或者不满足依赖的情况，安装中一定要注意。

- ** CentOS/RedHat **

```bash
[vagrant@vagrant-centos65 hadoop]$ sudo rpm -ivh hadoop-1.2.1-1.x86_64.rpm 
Preparing...                ########################################### [100%]
   1:hadoop                 ########################################### [100%]
[vagrant@vagrant-centos65 hadoop]$ 
```

- ** Ubuntu/Debian **

```bash
vagrant@precise64:~$ sudo dpkg -i hadoop_1.2.1-1_x86_64.deb 
Selecting previously unselected package hadoop.
(Reading database ... 51101 files and directories currently installed.)
Unpacking hadoop (from hadoop_1.2.1-1_x86_64.deb) ...
Setting up hadoop (1.2.1) ...
Processing triggers for ureadahead ...
ureadahead will be reprofiled on next reboot
vagrant@precise64:~$ 
```

使用安装包有很多优势：

- 简单。文件目录都会自动安装到正确的位置，并赋予正确的权限；
- 一致性。由于遵循了标准文件目录结构，配置文件会位于 `/etc/hadoop`、日志会位于 `/var/log/hadoop`，可执行文件会位于 `/usr/bin` 等等；
- 可集成性。由于位置是标准的，所以很容易与配置管理系统，如 Puppet 或 Chef 集成在一起，从而使得 Hadoop 的部署更加简单；
- 版本控制。可以使用系统自身的包管理软件进行升级维护。

关于 rpm 可以使用 `rpm -qlp hadoop-1.2.1-1.x86_64.rpm | less` 来检查包内文件，关于 deb 可以使用 `dpkg -c hadoop_1.2.1-1_x86_64.deb |less`。

需要了解的是下面的文件或目录：

- `/etc/hadoop`： 配置目录；
- `/etc/rc.d/init.d`： 启动脚本。可以通过这些脚本启动和停止 Hadoop 服务；
- `/usr/bin`： hadoop的可执行文件会放置在这里，还包括 `task-controller`；
- `/usr/include/hadoop`： 供 Hadoop Pipes 的 C/C++ 使用的头文件；
- `/usr/lib`： Hadoop 的 C 库，如 `libhdfs.so`, `libhadoop.a`, `libhadoop-pipes.a`等；
- `/usr/libexec`： 一些比较杂的东西，包括一些库所需要的文件和一些脚本；
- `/usr/sbin`： 一系列供管理员维护用的脚本，比如 `start-*.sh`、`stop-*.sh`、`hadoop-daemon.sh`等脚本；
- `/usr/share/doc/hadoop`： 文档、协议类的文件。


CDH
-----

CDH 如同 Apache Hadoop 一样，也是全部开源的项目。在 Apache Hadoop 的基础上，Cloudera 公司还负责将一些重要的补丁回溯打到老的版本上，以进行支持。并且 CDH 包含了大量的 Hadoop 生态圈的其它软件，包括 Hive, HBase, Pig 等等。

CDH 也提供了 tarball、RPM/Deb 之类的安装方式，此外，CDH 提供了更方便的 Yum/Apt 软件源的形式，这会自然解决依赖问题，所以更建议使用这种方式安装。

> 书中使用的是当时的 CDH 3 的版本，这里使用的是当前的 CDH 5。

### 配置软件源

- ** CentOS/RedHat **

```bash
[vagrant@centos ~]$ sudo rpm -ivh http://archive.cloudera.com/cdh5/one-click-install/redhat/6/x86_64/cloudera-cdh-5-0.x86_64.rpm
Retrieving http://archive.cloudera.com/cdh5/one-click-install/redhat/6/x86_64/cloudera-cdh-5-0.x86_64.rpm
Preparing...                ########################################### [100%]
   1:cloudera-cdh           ########################################### [100%]
[vagrant@centos ~]$ rpm -q cloudera-cdh -l
/etc/pki/rpm-gpg
/etc/pki/rpm-gpg/RPM-GPG-KEY-cloudera
/etc/yum.repos.d/cloudera-cdh5.repo
/usr/share/doc/cloudera-cdh-5
/usr/share/doc/cloudera-cdh-5/LICENSE
[vagrant@centos ~]$ 
```


- ** Ubuntu/Debian **

```bash
vagrant@ubuntu:~$ wget http://archive.cloudera.com/cdh5/one-click-install/precise/amd64/cdh5-repository_1.0_all.deb
--2014-04-14 13:11:11--  http://archive.cloudera.com/cdh5/one-click-install/precise/amd64/cdh5-repository_1.0_all.deb
Resolving archive.cloudera.com (archive.cloudera.com)... 54.230.132.16, 54.230.133.164, 54.230.133.136, ...
Connecting to archive.cloudera.com (archive.cloudera.com)|54.230.132.16|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3306 (3.2K) [application/x-debian-package]
Saving to: `cdh5-repository_1.0_all.deb'

100%[======================================>] 3,306       --.-K/s   in 0.001s  

2014-04-14 13:11:12 (2.90 MB/s) - `cdh5-repository_1.0_all.deb' saved [3306/3306]

vagrant@ubuntu:~$ sudo dpkg -i cdh5-repository_1.0_all.deb
Selecting previously unselected package cdh5-repository.
(Reading database ... 51095 files and directories currently installed.)
Unpacking cdh5-repository (from cdh5-repository_1.0_all.deb) ...
Setting up cdh5-repository (1.0) ...
gpg: keyring `/etc/apt/secring.gpg' created
gpg: keyring `/etc/apt/trusted.gpg.d/cloudera-cdh5.gpg' created
gpg: key 02A818DD: public key "Cloudera Apt Repository" imported
gpg: Total number processed: 1
gpg:               imported: 1
vagrant@ubuntu:~$ dpkg-query -L cdh5-repository             
/.
/etc
/etc/apt
/etc/apt/sources.list.d
/etc/apt/sources.list.d/cloudera-cdh5.list
/usr
/usr/share
/usr/share/doc
/usr/share/doc/cdh5-repository
/usr/share/doc/cdh5-repository/copyright
/usr/share/doc/cdh5-repository/cloudera-cdh5.key
vagrant@ubuntu:~$ 
```

### 获得可以安装的软件组件

要列出 CDH 都有哪些组件，可以使用下面的命令：

- ** CentOS/RedHat **

```bash
[vagrant@centos ~]$ yum --disablerepo="*" --enablerepo="cloudera-cdh5" list available
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
Available Packages
avro-doc.noarch                                            1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37.el6                      cloudera-cdh5
avro-libs.noarch                                           1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37.el6                      cloudera-cdh5
avro-tools.noarch                                          1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37.el6                      cloudera-cdh5
bigtop-jsvc.x86_64                                         0.6.0+cdh5.0.0+427-1.cdh5.0.0.p0.34.el6                     cloudera-cdh5
bigtop-jsvc-debuginfo.x86_64                               0.6.0+cdh5.0.0+427-1.cdh5.0.0.p0.34.el6                     cloudera-cdh5
bigtop-tomcat.noarch                                       0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.33.el6                       cloudera-cdh5
bigtop-utils.noarch                                        0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36.el6                       cloudera-cdh5
crunch.noarch                                              0.9.0+cdh5.0.0+21-1.cdh5.0.0.p0.24.el6                      cloudera-cdh5
crunch-doc.noarch                                          0.9.0+cdh5.0.0+21-1.cdh5.0.0.p0.24.el6                      cloudera-cdh5
flume-ng.noarch                                            1.4.0+cdh5.0.0+111-1.cdh5.0.0.p0.22.el6                     cloudera-cdh5
flume-ng-agent.noarch                                      1.4.0+cdh5.0.0+111-1.cdh5.0.0.p0.22.el6                     cloudera-cdh5
flume-ng-doc.noarch                                        1.4.0+cdh5.0.0+111-1.cdh5.0.0.p0.22.el6                     cloudera-cdh5
hadoop.x86_64                                              2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-conf-pseudo.x86_64                             2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-mapreduce.x86_64                               2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-mapreduce-jobtracker.x86_64                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-mapreduce-jobtrackerha.x86_64                  2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-mapreduce-tasktracker.x86_64                   2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-0.20-mapreduce-zkfc.x86_64                          2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-client.x86_64                                       2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-conf-pseudo.x86_64                                  2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-debuginfo.x86_64                                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-doc.x86_64                                          2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs.x86_64                                         2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-datanode.x86_64                                2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-fuse.x86_64                                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-journalnode.x86_64                             2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-namenode.x86_64                                2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-nfs3.x86_64                                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-secondarynamenode.x86_64                       2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-hdfs-zkfc.x86_64                                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-httpfs.x86_64                                       2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-libhdfs.x86_64                                      2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-libhdfs-devel.x86_64                                2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-mapreduce.x86_64                                    2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-mapreduce-historyserver.x86_64                      2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-yarn.x86_64                                         2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-yarn-nodemanager.x86_64                             2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-yarn-proxyserver.x86_64                             2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hadoop-yarn-resourcemanager.x86_64                         2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                     cloudera-cdh5
hbase.x86_64                                               0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hbase-doc.x86_64                                           0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hbase-master.x86_64                                        0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hbase-regionserver.x86_64                                  0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hbase-rest.x86_64                                          0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hbase-solr.noarch                                          1.3+cdh5.0.0+38-1.cdh5.0.0.p0.23.el6                        cloudera-cdh5
hbase-solr-doc.noarch                                      1.3+cdh5.0.0+38-1.cdh5.0.0.p0.23.el6                        cloudera-cdh5
hbase-solr-indexer.noarch                                  1.3+cdh5.0.0+38-1.cdh5.0.0.p0.23.el6                        cloudera-cdh5
hbase-thrift.x86_64                                        0.96.1.1+cdh5.0.0+60-1.cdh5.0.0.p0.39.el6                   cloudera-cdh5
hive.noarch                                                0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-hbase.noarch                                          0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-hcatalog.noarch                                       0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-hcatalog-server.noarch                                0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-jdbc.noarch                                           0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-metastore.noarch                                      0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-server.noarch                                         0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-server2.noarch                                        0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-webhcat.noarch                                        0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hive-webhcat-server.noarch                                 0.12.0+cdh5.0.0+308-1.cdh5.0.0.p0.34.el6                    cloudera-cdh5
hue.x86_64                                                 3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-beeswax.x86_64                                         3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-common.x86_64                                          3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-doc.x86_64                                             3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-hbase.x86_64                                           3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-impala.x86_64                                          3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-pig.x86_64                                             3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-plugins.x86_64                                         3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-rdbms.x86_64                                           3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-search.x86_64                                          3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-server.x86_64                                          3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-spark.x86_64                                           3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-sqoop.x86_64                                           3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
hue-zookeeper.x86_64                                       3.5.0+cdh5.0.0+365-1.cdh5.0.0.p0.42.el6                     cloudera-cdh5
impala.x86_64                                              1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-catalog.x86_64                                      1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-debuginfo.x86_64                                    1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-server.x86_64                                       1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-shell.x86_64                                        1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-state-store.x86_64                                  1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
impala-udf-devel.x86_64                                    1.3.0+cdh5.0.0+0-1.cdh5.0.0.p0.126.el6                      cloudera-cdh5
kite.noarch                                                0.10.0+cdh5.0.0+79-1.cdh5.0.0.p0.21.el6                     cloudera-cdh5
llama.noarch                                               1.0.0+cdh5.0.0+0-1.cdh5.0.0.p0.38.el6                       cloudera-cdh5
llama-doc.noarch                                           1.0.0+cdh5.0.0+0-1.cdh5.0.0.p0.38.el6                       cloudera-cdh5
llama-master.noarch                                        1.0.0+cdh5.0.0+0-1.cdh5.0.0.p0.38.el6                       cloudera-cdh5
mahout.noarch                                              0.8+cdh5.0.0+27-1.cdh5.0.0.p0.19.el6                        cloudera-cdh5
mahout-doc.noarch                                          0.8+cdh5.0.0+27-1.cdh5.0.0.p0.19.el6                        cloudera-cdh5
oozie.noarch                                               4.0.0+cdh5.0.0+174-1.cdh5.0.0.p0.26.el6                     cloudera-cdh5
oozie-client.noarch                                        4.0.0+cdh5.0.0+174-1.cdh5.0.0.p0.26.el6                     cloudera-cdh5
parquet.noarch                                             1.2.5+cdh5.0.0+91-1.cdh5.0.0.p0.31.el6                      cloudera-cdh5
parquet-format.noarch                                      1.0.0+cdh5.0.0+3-1.cdh5.0.0.p0.39.el6                       cloudera-cdh5
pig.noarch                                                 0.12.0+cdh5.0.0+27-1.cdh5.0.0.p0.26.el6                     cloudera-cdh5
pig-udf-datafu.noarch                                      1.1.0+cdh5.0.0+7-1.cdh5.0.0.p0.18.el6                       cloudera-cdh5
search.noarch                                              1.0.0+cdh5.0.0+0-1.cdh5.0.0.p0.19.el6                       cloudera-cdh5
sentry.noarch                                              1.2.0+cdh5.0.0+71-1.cdh5.0.0.p0.25.el6                      cloudera-cdh5
solr.noarch                                                4.4.0+cdh5.0.0+178-1.cdh5.0.0.p0.37.el6                     cloudera-cdh5
solr-doc.noarch                                            4.4.0+cdh5.0.0+178-1.cdh5.0.0.p0.37.el6                     cloudera-cdh5
solr-mapreduce.noarch                                      1.0.0+cdh5.0.0+0-1.cdh5.0.0.p0.19.el6                       cloudera-cdh5
solr-server.noarch                                         4.4.0+cdh5.0.0+178-1.cdh5.0.0.p0.37.el6                     cloudera-cdh5
spark-core.noarch                                          0.9.0+cdh5.0.0+31-1.cdh5.0.0.p0.31.el6                      cloudera-cdh5
spark-master.noarch                                        0.9.0+cdh5.0.0+31-1.cdh5.0.0.p0.31.el6                      cloudera-cdh5
spark-python.noarch                                        0.9.0+cdh5.0.0+31-1.cdh5.0.0.p0.31.el6                      cloudera-cdh5
spark-worker.noarch                                        0.9.0+cdh5.0.0+31-1.cdh5.0.0.p0.31.el6                      cloudera-cdh5
sqoop.noarch                                               1.4.4+cdh5.0.0+43-1.cdh5.0.0.p0.21.el6                      cloudera-cdh5
sqoop-metastore.noarch                                     1.4.4+cdh5.0.0+43-1.cdh5.0.0.p0.21.el6                      cloudera-cdh5
sqoop2.noarch                                              1.99.3+cdh5.0.0+26-1.cdh5.0.0.p0.24.el6                     cloudera-cdh5
sqoop2-client.noarch                                       1.99.3+cdh5.0.0+26-1.cdh5.0.0.p0.24.el6                     cloudera-cdh5
sqoop2-server.noarch                                       1.99.3+cdh5.0.0+26-1.cdh5.0.0.p0.24.el6                     cloudera-cdh5
whirr.noarch                                               0.9.0+cdh5.0.0+4-1.cdh5.0.0.p0.20.el6                       cloudera-cdh5
zookeeper.x86_64                                           3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6                      cloudera-cdh5
zookeeper-debuginfo.x86_64                                 3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6                      cloudera-cdh5
zookeeper-native.x86_64                                    3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6                      cloudera-cdh5
zookeeper-server.x86_64                                    3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6                      cloudera-cdh5
[vagrant@centos ~]$ 
```


- ** Ubuntu/Debian **

```bash
lessvagrant@ubuntu:~$ curl -s http://archive-primary.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/dists/precise-cdh5/contrib/binary-amd64/Packages.gz | gzip -d | grep Package
Package: avro-doc
Package: avro-libs
Package: avro-tools
Package: bigtop-jsvc
Package: bigtop-tomcat
Package: bigtop-utils
Package: crunch
Package: crunch-doc
Package: flume-ng
Package: flume-ng-agent
Package: flume-ng-doc
Package: hadoop
Package: hadoop-0.20-conf-pseudo
Package: hadoop-0.20-mapreduce
Package: hadoop-0.20-mapreduce-jobtracker
Package: hadoop-0.20-mapreduce-jobtrackerha
Package: hadoop-0.20-mapreduce-tasktracker
Package: hadoop-0.20-mapreduce-zkfc
Package: hadoop-client
Package: hadoop-conf-pseudo
Package: hadoop-doc
Package: hadoop-hdfs
Package: hadoop-hdfs-datanode
Package: hadoop-hdfs-fuse
Package: hadoop-hdfs-journalnode
Package: hadoop-hdfs-namenode
Package: hadoop-hdfs-nfs3
Package: hadoop-hdfs-secondarynamenode
Package: hadoop-hdfs-zkfc
Package: hadoop-httpfs
Package: hadoop-mapreduce
Package: hadoop-mapreduce-historyserver
Package: hadoop-yarn
Package: hadoop-yarn-nodemanager
Package: hadoop-yarn-proxyserver
Package: hadoop-yarn-resourcemanager
Package: hbase
Package: hbase-doc
Package: hbase-master
Package: hbase-regionserver
Package: hbase-rest
Package: hbase-solr
Package: hbase-solr-doc
Package: hbase-solr-indexer
Package: hbase-thrift
Package: hive
Package: hive-hbase
Package: hive-hcatalog
Package: hive-hcatalog-server
Package: hive-jdbc
Package: hive-metastore
Package: hive-server
Package: hive-server2
Package: hive-webhcat
Package: hive-webhcat-server
Package: hue
Package: hue-beeswax
Package: hue-common
Package: hue-doc
Package: hue-hbase
Package: hue-impala
Package: hue-pig
Package: hue-plugins
Package: hue-rdbms
Package: hue-search
Package: hue-server
Package: hue-spark
Package: hue-sqoop
Package: hue-zookeeper
Package: impala
Package: impala-catalog
Package: impala-dbg
Package: impala-server
Package: impala-shell
Package: impala-state-store
Package: impala-udf-dev
Package: kite
Package: libhdfs0
Package: libhdfs0-dev
Package: llama
Package: llama-doc
Package: llama-master
Package: mahout
Package: mahout-doc
Package: oozie
Package: oozie-client
Package: parquet
Package: parquet-format
Package: pig
Package: pig-udf-datafu
Package: search
Package: sentry
Package: solr
Package: solr-doc
Package: solr-mapreduce
Package: solr-server
Package: spark-core
Package: spark-master
Package: spark-python
Package: spark-worker
Package: sqoop
Package: sqoop-metastore
Package: sqoop2
Package: sqoop2-client
Package: sqoop2-server
Package: whirr
Package: zookeeper
Package: zookeeper-native
Package: zookeeper-server
vagrant@ubuntu:~$
```

### 安装所需组件

可以根据服务器的任务决定所需要安装的组件。下面以安装 Hadoop 为例。

- ** CentOS/RedHat **

安装任何软件前，应该先执行升级任务。

```bash
sudo yum update
```

然后是安装 Hadoop

```bash
sudo yum install hadoop-hdfs hadoop-mapreduce

[vagrant@centos ~]$ sudo yum install hadoop
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: centos.mirror.serversaustralia.com.au
 * epel: mirror.optus.net
 * extras: mirror.ventraip.net.au
 * updates: mirror.ventraip.net.au
Setting up Install Process
Resolving Dependencies
--> Running transaction check
---> Package hadoop.x86_64 0:2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6 will be installed
--> Processing Dependency: bigtop-utils >= 0.7 for package: hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64
--> Processing Dependency: zookeeper >= 3.4.0 for package: hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64
--> Processing Dependency: avro-libs for package: hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64
--> Processing Dependency: parquet for package: hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64
--> Processing Dependency: redhat-lsb for package: hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64

...

---> Package poppler-data.noarch 0:0.4.0-1.el6 will be installed
---> Package xml-common.noarch 0:0.6.3-32.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                  Arch   Version                    Repository     Size
================================================================================
Installing:
 hadoop                   x86_64 2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6
                                                            cloudera-cdh5  19 M
Installing for dependencies:
 alsa-lib                 x86_64 1.0.22-3.el6               base          370 k
 at                       x86_64 3.1.10-43.el6_2.1          base           60 k
 atk                      x86_64 1.30.0-1.el6               base          195 k
 avahi-libs               x86_64 0.6.25-12.el6              base           54 k
 avro-libs                noarch 1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37.el6
                                                            cloudera-cdh5  12 M
 bc                       x86_64 1.06.95-1.el6              base          110 k
 bigtop-utils             noarch 0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36.el6
                                                            cloudera-cdh5 8.8 k
 cairo                    x86_64 1.8.8-3.1.el6              base          309 k
 cdparanoia-libs          x86_64 10.2-5.1.el6               base           47 k
 
 ...
 
 zookeeper                x86_64 3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6
                                                            cloudera-cdh5 3.7 M

Transaction Summary
================================================================================
Install     106 Package(s)

Total download size: 137 M
Installed size: 325 M
Is this ok [y/N]: y
Downloading Packages:
(1/106): alsa-lib-1.0.22-3.el6.x86_64.rpm                | 370 kB     00:00     
(2/106): at-3.1.10-43.el6_2.1.x86_64.rpm                 |  60 kB     00:00     
(3/106): atk-1.30.0-1.el6.x86_64.rpm                     | 195 kB     00:00     
(4/106): avahi-libs-0.6.25-12.el6.x86_64.rpm             |  54 kB     00:00     
(5/106): avro-libs-1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37.el |  12 MB     00:09     
(6/106): bc-1.06.95-1.el6.x86_64.rpm                     | 110 kB     00:00     
(7/106): bigtop-utils-0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36. | 8.8 kB     00:00     

...
  
(106/106): zookeeper-3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36. | 3.7 MB     00:02     
--------------------------------------------------------------------------------
Total                                           1.3 MB/s | 137 MB     01:47     
warning: rpmts_HdrFromFdno: Header V4 DSA/SHA1 Signature, key ID e8f86acd: NOKEY
Retrieving key from http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/RPM-GPG-KEY-cloudera
Importing GPG key 0xE8F86ACD:
 Userid: "Yum Maintainer <webmaster@cloudera.com>"
 From  : http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/RPM-GPG-KEY-cloudera
Is this ok [y/N]: y
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : freetype-2.3.11-14.el6_3.1.x86_64                          1/106 
  Installing : fontconfig-2.8.0-3.el6.x86_64                              2/106 
  Installing : libjpeg-turbo-1.2.1-3.el6_5.x86_64                         3/106 
  Installing : 2:libpng-1.2.49-1.el6_2.x86_64                             4/106 
  Installing : libICE-1.0.6-1.el6.x86_64                                  5/106 
  Installing : libSM-1.2.1-2.el6.x86_64                                   6/106 
  Installing : 1:qt-4.6.2-28.el6_5.x86_64                                 7/106 

  ...

  Installing : hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64    105/106 
  Installing : parquet-format-1.0.0+cdh5.0.0+3-1.cdh5.0.0.p0.39.el6.n   106/106 
  Verifying  : libXdamage-1.1.3-4.el6.x86_64                              1/106 
  Verifying  : libSM-1.2.1-2.el6.x86_64                                   2/106 
  Verifying  : at-3.1.10-43.el6_2.1.x86_64                                3/106 
  Verifying  : hadoop-2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6.x86_64      4/106 

  ...
  
  Verifying  : libthai-0.1.12-3.el6.x86_64                              105/106 
  Verifying  : libXv-1.0.7-2.el6.x86_64                                 106/106 

Installed:
  hadoop.x86_64 0:2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69.el6                       

Dependency Installed:
  alsa-lib.x86_64 0:1.0.22-3.el6                                                
  at.x86_64 0:3.1.10-43.el6_2.1                                                 
  atk.x86_64 0:1.30.0-1.el6                                                     

  ...
                                          
  xorg-x11-font-utils.x86_64 1:7.2-11.el6                                       
  zookeeper.x86_64 0:3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36.el6                     

Complete!
[vagrant@centos ~]$ 

```

- ** Ubuntu/Debian **

安装任何软件前，应该先执行升级任务。

```bash
sudo apt-get update && sudo apt-get upgrade
```

然后是安装 Hadoop

```bash
vagrant@ubuntu:~$ sudo apt-get install hadoop-hdfs hadoop-mapreduce
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  avro-libs bigtop-utils parquet parquet-format zookeeper
The following NEW packages will be installed:
  avro-libs bigtop-utils hadoop parquet parquet-format zookeeper
0 upgraded, 6 newly installed, 0 to remove and 162 not upgraded.
Need to get 48.1 MB of archives.
After this operation, 56.6 MB of additional disk space will be used.
Do you want to continue [Y/n]? y
Get:1 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib avro-libs all 1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37~precise-cdh5.0.0 [12.6 MB]
Get:2 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib bigtop-utils all 0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36~precise-cdh5.0.0 [33.0 kB]
Get:3 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib zookeeper all 3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36~precise-cdh5.0.0 [4,159 kB]
Get:4 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib parquet-format all 1.0.0+cdh5.0.0+3-1.cdh5.0.0.p0.39~precise-cdh5.0.0 [440 kB]
Get:5 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib parquet all 1.2.5+cdh5.0.0+91-1.cdh5.0.0.p0.31~precise-cdh5.0.0 [10.6 MB]
Get:6 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/ precise-cdh5/contrib hadoop all 2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69~precise-cdh5.0.0 [20.3 MB]
Fetched 48.1 MB in 2min 44s (293 kB/s)                                                                                             
Selecting previously unselected package avro-libs.
(Reading database ... 51141 files and directories currently installed.)
Unpacking avro-libs (from .../avro-libs_1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37~precise-cdh5.0.0_all.deb) ...
Selecting previously unselected package bigtop-utils.
Unpacking bigtop-utils (from .../bigtop-utils_0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36~precise-cdh5.0.0_all.deb) ...
Selecting previously unselected package zookeeper.
Unpacking zookeeper (from .../zookeeper_3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36~precise-cdh5.0.0_all.deb) ...
Selecting previously unselected package parquet-format.
Unpacking parquet-format (from .../parquet-format_1.0.0+cdh5.0.0+3-1.cdh5.0.0.p0.39~precise-cdh5.0.0_all.deb) ...
Selecting previously unselected package parquet.
Unpacking parquet (from .../parquet_1.2.5+cdh5.0.0+91-1.cdh5.0.0.p0.31~precise-cdh5.0.0_all.deb) ...
Selecting previously unselected package hadoop.
Unpacking hadoop (from .../hadoop_2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69~precise-cdh5.0.0_all.deb) ...
Processing triggers for man-db ...
Setting up avro-libs (1.7.5+cdh5.0.0+16-1.cdh5.0.0.p0.37~precise-cdh5.0.0) ...
Setting up bigtop-utils (0.7.0+cdh5.0.0+0-1.cdh5.0.0.p0.36~precise-cdh5.0.0) ...
Setting up zookeeper (3.4.5+cdh5.0.0+28-1.cdh5.0.0.p0.36~precise-cdh5.0.0) ...
update-alternatives: using /etc/zookeeper/conf.dist to provide /etc/zookeeper/conf (zookeeper-conf) in auto mode.
Setting up parquet (1.2.5+cdh5.0.0+91-1.cdh5.0.0.p0.31~precise-cdh5.0.0) ...
Setting up parquet-format (1.0.0+cdh5.0.0+3-1.cdh5.0.0.p0.39~precise-cdh5.0.0) ...
Setting up hadoop (2.3.0+cdh5.0.0+548-1.cdh5.0.0.p0.69~precise-cdh5.0.0) ...
update-alternatives: using /etc/hadoop/conf.empty to provide /etc/hadoop/conf (hadoop-conf) in auto mode.
Processing triggers for libc-bin ...
ldconfig deferred processing now taking place
vagrant@ubuntu:~$ 

```

### 同 Apache Hadoop 的差异

- 使用 `alternatives` 系统： CDH 使用了 Linux 中的 `alternatives` 系统。这个系统可以使一个计算机中对某个软件安装多个版本，可以通过系统的 `alternatives` 命令随时切换不同版本。对于 RedHat/CentOS 类用户可以通过 `man 8 alternatives` 查看进一步信息，对于 Debian/Ubuntu 用户可以通过 `man 8 update-alternatives`  查看进一步信息；
- CDH 会添加、修改 `/etc/security/limits.d` 中的文件，用以对用户 `hdfs` 以及 `mapred` 进行配置合适的 `nofile` 和 `nproc` 限额；
- Hadoop 主目录使用 `/usr/lib/hadoop*` 而不是使用 `/usr/share/hadoop`。

### 建立软件源镜像

对于集群而言，经常的会在局域网内部建立一个 Yum/Apt 软件源的镜像。这样做有两个好处

- 可以相当大的程度上减少安装、升级时对外部带宽、流量的占用；
- 很多时候，集群中不是所有的机器都可以访问互联网，内部的软件源镜像可以解决这一问题；

创建源：

- ** CentOS/RedHat **

```bash
sudo yum install httpd yum-utils createrepo
sudo mkdir -p /var/www/html/repo
sudo chmod a+w /var/www/html/repo
cd /var/www/html/repo
sudo reposync -r cloudera-cdh5
sudo createrepo cloudera-cdh5
```

不要忘记了在其它服务器配置 cloudera 源的时候，修改为该镜像源

```
baseurl=http://mirror-server/cloudera-cdh5
```

注意将其中的 `mirror-server` 换成实际的域名或者IP。


- ** Ubuntu/Debian **

需要注意的是由于 `apt-mirror` 不支持默认的 cloudera-cdh5.list 中的一些格式。需要先行修改一下其格式。

```bash
sudo nano /etc/apt/source.list.d/cloudera-cdh5.list
```

注意其中的 `[arch=amd64]` 行，修改为以下格式：

```
deb-amd64 http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib
deb-src http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib
```

然后安装镜像工具：

```bash
sudo apt-get install apt-mirror apache2
```

下载：

```bash
vagrant@ubuntu:~$ sudo apt-mirror /etc/apt/sources.list.d/cloudera-cdh5.list 
Downloading 10 index files using 10 threads...
Begin time: Mon Apr 14 17:14:25 2014
[10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]... 
End time: Mon Apr 14 17:14:27 2014

Proceed indexes: [SP]

3.0 GiB will be downloaded into archive.
Downloading 193 archive files using 20 threads...
Begin time: Mon Apr 14 17:14:27 2014
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]... 
End time: Mon Apr 14 18:15:31 2014

0.0 bytes in 0 files and 0 directories can be freed.
Run /var/spool/apt-mirror/var/clean.sh for this purpose.

Running the Post Mirror script ...
(/var/spool/apt-mirror/var/postmirror.sh)


Post Mirror script has completed. See above output for any possible errors.

vagrant@ubuntu:~$
```

下载好的文件位于 `/var/spool/apt-mirror/mirror/` 目录下。我们可以将其链接到 apache 的www目录下：

```bash
sudo ln -s /var/spool/apt-mirror/mirror/archive.cloudera.com /var/www/cloudera
```

然后，不要忘记在其他服务器上设置下载 cloudera 的源时，使用当前机器的服务器：

```
deb-amd64 http://mirror-server/cloudera/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib
deb-src http://mirror-server/cloudera/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib
```

注意将其中的 `mirror-server` 换成实际服务器的 IP 或者域名。

### 建立软件源缓存代理

进行全源镜像是比较大型网络解决流量损耗问题的方案，但是它有些很明显的缺陷。

**全源镜像很大。**大约需要下载几十GB，如果包括了不同版本不同构架的系统的源，大小更是成倍增加，甚至需要几百个GB。这将是第一次初始化镜像时所需要下载的量。这除了意味着将占用大量的硬盘空间外，还意味着巨大的下载流量，以及长达几天甚至更久的初始化时间。

其实绝大多数软件包我们不会用到，而且对于集群来说，机器又非常相似。所以，也可以选择另一种方式，既缓存/代理的方式。所有的计算机通过这个代理下载软件包，凡是别人下载过的，就不用再从公网下载，而只需从代理服务器下载即可。

以 ubuntu 为例，安装配置非常简单。

首先在将要做缓存的服务器上安装代理软件：

```bash
sudo apt-get install squid-deb-proxy
```

然后再在所有计算机上安装代理客户端软件：

```bash
sudo apt-get install squid-deb-proxy-client
```

然后就好了，当需要 `apt-get` 升级的时候，会自动寻找到代理服务器，并通过代理服务器下载。无需额外的配置。

需要注意的是，**默认的配置只是针对 Ubuntu 官方的源的**，对于安装 CDH 而言，还需要配置 CDH 的源，否则会导致软件无法安装成功。添加一个文件 `/etc/squid-deb-proxy/mirror-dstdomain.acl.d/10-cloudera`，其内容为：

```
archive.cloudera.com
```

**在修改了该配置后，不要忘记重新启动代理服务器**

```bash
sudo restart squid-deb-proxy
```




配置：概述
===========

Hadoop 是高度可配置的，其拥有几百项配置项目。了解这些配置，是维护大规模集群，确保系统正常运行所必需的。

> 随着 Hadoop 的版本升级，很多之前使用的配置名称会发生改变，其目的是为了使其名称可以更准确的反应其功能。本书将使用 1.x 的配置名称。NameNode High Availability 以及 Fedoration 必须使用新版本的配置，因此该部分将使用新名称。

Hadoop 的配置可以分为四大类：

- 集群 (Cluster)
- 服务 (Daemon)
- 任务 (Job)
- 操作 (Individual Operation)

运维管理员将主要关注于前两者，而程序开发人员主要关注于后两者。

要使集群和服务级别的参数变更生效，往往需要重新启动集群，因此部分参数变更可能会导致服务中断。

许多参数可以由管理员设置默认值，但是执行期间可以由命令行、程序代码覆盖这些配置（如果允许的话）。如 `hadoop fs` 命令，可以控制该文件的副本数，甚至跨集群复制文件。

配置文件列表：

- `hadoop-env.sh`： Hadoop、JDK等环境变量，如需修改则在该文件中写入其值；
- `core-site.xml`： 核心配置文件，和所有的 Hadoop 服务和客户端都有关系；
- `hdfs-site.xml`： HDFS 相关的配置文件。HDFS 服务以及其客户端都需要该文件；
- `mapred-site.xml`： MapReduce 相关的配置文件。MapReduce 服务和其客户端都需要该文件；
- `log4j.properties`： log4j 的 Java 日志配置文件；
- `masters` (可选)： 行分割文件，指定 SecondaryNameNode 的文件。该文件在 2.x 系列中已不需要；
- `slaves` (可选)： 行分割文件。NameNode 将从该文件中读取 DataNode 列表，JobTracker 将从该文件中读取 TaskTracker 列表；
- `fair-scheduler.xml`(可选)： 当使用 Fair Scheduler 调度器时，该文件用于配置；
- `capacity-scheduler.xml`(可选)： 当使用 Capacity Scheduler 调度器时，该文件用于配置；
- `dfs.include`(可选，常用名)： 行分割文件，包含允许加入HDFS集群的节点列表。该文件名需要通过配置指定，一般常用该名称；
- `dfs.exclude`(可选，常用名)： 行分割文件，包含拒绝加入HDFS集群的节点列表。该文件名需要通过配置指定，一般常用该名称；
- `hadoop-policy.xml`： 用于指定哪个用户或组允许访问某个RPC的配置文件；
- `mapred-queue-acls.xml`： 指定哪个用户或组允许提交任务给 MapReduce 的哪一个队列；
- `taskcontroller.cfg`： 在安全模式下，`task-controller` 所使用的属性配置文件；

这些文件大多是使用 Java 的 [ClassLoader](http://docs.oracle.com/javase/6/docs/api/java/lang/ClassLoader.html) 资源加载机制从 classpath 中加载的，因此 Hadoop 的启动脚本将会确保配置文件目录位于 classpath 的头部。

Hadoop XML 配置文件
--------------------

Hadoop 主要的配置文件是三个 XML 文件 `core-site.xml`、`hdfs-site.xml` 以及 `mapred-site.xml`。这三个文件中所配置的项，将覆盖默认值。

当 MapReduce 运行的时候，部分参数可以有程序开发人员覆盖其值。管理员可以通过设置 `final` 属性，来阻止该配置被 MapReduce 程序所覆盖。

如：

```xml
<property>
  <name>foo.bar.baz</name>
  <value>42</value>
  <final>true</final>
</property>
```

后续章节描述配置是将直接使用名称和其值描述，而写入配置文件时，将使用上述格式，并依据需要决定是否需要 `<final>true</final>` 项。

环境变量和 Shell 脚本
======================

日志配置
==========

HDFS
=====

NameNode High Availability
============================

Namenode Federation
=====================


MapReduce
=============


机架拓扑
===========


安全
======




