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

Hadoop Shell 脚本分两类：

- 一类是定义 Hadoop 运行所必需的参数。如 JDK 的位置、选项、Hadoop 的位置参数等；
- 另一类是为服务运行时提供配置环境变量的。

`haoop-env.sh` 中需要配置 `JAVA_HOME`，虽然 `hadoop` 会自己寻找 `JAVA_HOME`，如果没有配置好的 `JAVA_HOME` 依旧会导致访问时使用了错误的 JDK 版本，或者干脆找不到。

`HADOOP_daemon_OPTS` 的选项将会传给 对应的 daemon(服务)。

新的 Hadoop 版本开始使用 [Apache Bigtop](http://bigtop.apache.org/) 项目，该项目将会帮助 Hadoop 生态圈的软件寻找到合适的JDK以及其它所需软件，从而不必每个组件都实现一套这个需求。CDH 的发布版本中就使用了 Bigtop。

有时，用户会要求管理员将他们所用的 jar 文件加入到 `$HADOOP_CLASSPATH` 中去。**要尽一切可能不去把用户所需要的 jar 加入 `$HADOOP_CLASSPATH` 中去，要推荐用户使用 Hadoop Distributed Cache。** 要知道，`$HADOOP_CLASSPATH` 的设计目的是为了给 Hadoop 服务程序提供所需的库的，而不是为用户的 MapReduce 程序提供库的。虽然把 MapReduce 程序所依赖的库放到 `$HADOOP_CLASSPATH` 中去可以让该 MapReduce 程序运行，但是却破坏了 Hadoop 服务的稳定性。此外，**每次修改 `$HADOOP_CLASSPATH`，需要重启所有的 TaskTracker 以使之生效。**相信没有人希望每次用户所依赖的库升级后都要重新启动一次集群。

`$HADOOP_HOME` 已经不推荐使用了，可以使用 `$HADOOP_PREFIX`代替。

日志配置
==========

几乎所有的 Hadoop 的服务都是用 log4j 来控制日志输出，因此，必须合理的配置 `log4j.properties` 才可以看到日志输出。Hadoop 默认的 `log4j.properties` 文件位于 `conf` 目录下，和其它的配置文件放在一起。

自己撰写 MapReduce 程序的时候，特别是本机调试的时候，需要让 `log4j.properties` 文件位于 CLASSPATH 内，不然调试的时候会无法输出日志信息。


HDFS
=====

接下来说一下 `hdfs-site.xml` 配置。

标识和定位
------------

### fs.default.name (core-site.xml)

该参数配置客户端默认的文件系统的位置。默认使用的是 `file:///`，既本地文件系统。一般开发环境会使用这个配置，而生产环境则会配置为真正的集群地址，其形式为： `hdfs://hostname:port`。其中 `hostname` 和 `port` 都是指 NameNode 的地址。

这个地址有两方面用处，一方面供客户端读取以定位 NameNode 位置，另一方面供 NameNode 自己读取自己应该监听的端口。默认情况下，NameNode 监听端口为 `8020`

- 使用该配置的有：NN、DN、SNN、JT、TT、客户端

### dfs.name.dir

**配置 HDFS 中 NameNode 数据存储位置，这是非常重要的一个配置。** 该值为逗号分割的无空格字符串，可以包含一个到多个路径。如： `/data/1/dfs/nn,/data/2/dfs/nn,/data/3/dfs/nn`，如果是多个路径，每个路径将存放相同的内容，目的为冗余备份。一般情况下，在这里配置至少两个本地路径，分别指向不同的物理硬盘。此外，还会再有一个 NFS 的路径，目的为避免物理机火灾类的严重故障时的 HDFS 的核心信息不至于丢失。由于这种冗余能力，所以不需要再额外的使用RAID作为存储，不过一些管理员还是会选择 RAID，为了再加一层保险。这个目录中的数据量不是海量的，是 GB 级别，而不会到 TB 级。

> **需要注意**： 由于很多部署采用同质化部署，既所有机器使用相同的配置。而由于 NameNode 不需要很多硬盘空间，导致 NameNode 上的硬盘空间很大部分被闲置。于是一些管理员总觉得应该充分利用磁盘，从而**在 NameNode 上还跑一些其它的服务，这是非常错误的。**

该配置的默认值是 `${hadoop.tmp.dir}/dfs/name`，而默认情况下，`hadoop.tmp.dir`是 `/tmp/hadoop-${user.name}`。`/tmp` 重新启动后，内容会被清空。于是，如果没有修改 `dfs.name.dir` 或者 `hadoop.tmp.dir` 的话，NameNode 的所有信息实际上是存储于空中，重启后，所有信息都会丢失。因此一定要检查 `dfs.name.dir` 配置。

- 使用该配置的有：NN

### dfs.data.dir

`dfs.data.dir` 指定了 DataNode 存储 HDFS block 的位置。其内容也是逗号分割的字符串，来表示多个目录位置。一般来说是对应于多个物理硬盘，换句话说，硬盘只需要 JBOD 形式，而不需要用RAID，这样的性能较高。**Hadoop 会将 block 轮训存入各个位置**，这样将来读取时才可能会并发读入。HDFS 会将 block 副本存于多个 DataNode，因此不必担心磁盘故障会导致数据丢失的问题。

- 使用该配置的有：DN

### fs.checkpoint.dir

`fs.checkpoint.dir` 指定了 SecondaryNameNode 所需要进行 checkpoint 合并的目录。也是逗号分割的字符串，与 `dfs.name.dir` 一样，不同的位置一般代表着不同的物理硬盘，并且所有的位置会入完全一样的信息，以达到冗余的目的。这个目录一般是作为恢复 NameNode 元数据的最后的解决办法。**如果没有任何其它办法可以恢复元数据**，这里可以起码有一个可以用的、虽然较老的元数据。

- 使用该配置的有：SNN

### dfs.permissions.supergroup

HDFS 中存在一个超级用户的概念，凡是在 `dfs.permissions.supergroup` 中指定的用户组，将享有超级用户的身份。**拥有该身份的用户可以执行 HDFS 上的任何操作。**放入该组的用户需要被格外关注。

- 使用该配置的有：NN、客户端

优化调整
----------

### io.file.buffer.size (core-site.xml)

Hadoop 的代码中很多会使用文件IO操作，`io.file.buffer.size` 指定了通用的缓存大小。越大的缓存意味着更好地数据传输率，但也意味着更大的延迟和更大的内存占用。**其值为内存页大小(默认为4KB)的倍数**。比如可以调整为 64KB。

- 使用该配置的有：客户端、所有服务

### dfs.balance.bandwidthPerSec

HDFS balancer 是负责 HDFS DataNode 的存储负载均衡的，它会调整数据在 DN 之间复制。如果没有对其流量进行限制，则可能会造成 balancer 占用了全部的网络带宽。该配置就是对其限定的。需要注意的是，其单位为字节每秒，而非网络中常用的位每秒。

**该配置不能够动态调整**，只有在 DN 启动时，方会读取一次该值。

- 使用该配置的有：DN

### dfs.block.size

`dfs.block.size` 顾名思义，是 HDFS block 大小。但是需要注意的是，这里有一个常见的**误解**，该值的含义严格来说是**默认**的HDFS block 大小。因为设定该值并不会改变已有文件的存储方式，也就是说**不会变动已有文件的块大小**；此外，该值也可以在创建文件的时候被覆盖掉，**客户端可以为该文件指定另外的块大小**。

默认值因Hadopp发行版本不同而不同，常见的是64MB或128MB。**MapReduce 的 Job 会为每个块分配一个 map，因此 Block 的大小会明显的影响 Job 的性能。** (实际情况会稍有差异，请查阅 《Hadoop 权威指南》中关于 `FileInputFormat` 的章节。)

- 使用该配置的有：客户端

### dfs.datanode.du.reserved

DataNode 会将其本地可供 HDFS 使用的磁盘空间(`dfs.data.dir`)大小通报 NameNode。而通常情况下，`mapred.local.dir` 使用同样的硬盘，因此它们的空间是共享的。一般为了确保 MapReduce 的任务执行，需要为其保留一定的空间供其使用。`dfs.datanode.du.reserved` 就是保留一部分空间，使 HDFS 不能够使用全部的硬盘空间。**默认情况下该值为0，也就是说允许 HDFS 使用全部的硬盘空间。**可以考虑给每个物理硬盘预留 10GB 空间供 MapReduce 使用，不过要观察实际情况，如果 MapReduce 会产生大量的中间结果，则应该适度增加该值。

- 使用该配置的有：DN

### dfs.namenode.handler.count

NameNode 维护一个线程池，用以响应来自包括客户端和服务端的并发服务请求。`dfs.namenode.handler.count` 配置了线程池的规模，其默认值是10。越大的集群则需要越高的并发池。**一般会设置为 `20 * log(n)`。** 比如一个200节点的集群，该值应该设置为 `20 * log(200) = 106`。

该数值过低会导致 DN 经常因为超时而被 NN 所拒绝链接，NN 的 RPC 的队列很长，以及 RPC 的高延迟。

- 使用该配置的有：NN

### dfs.datanode.failed.volumes.tolerated

默认情况下，DN上的一个硬盘坏掉了就会标记为该 DN 坏掉了。这就会触发 NN 开始进行不满足副本数的 block 复制。对于中型和大型规模的集群来说，硬盘坏掉是很经常的事情，没必要只要有一块硬盘坏了就宣布这个DN挂了。`dfs.datanode.failed.volumes.tolerated` 就是设置，有几个硬盘坏掉后，则认为 DN 挂了。其默认值为 0，既无容忍，一块坏了就挂。

- 使用该配置的有：DN

### dfs.hosts

当 DN 第一次联系 NN 的时候，在获得其 Namespace ID 后，立即就可以接收数据块了。**默认情况下，任意服务器都可以这样联系 NameNode 而加入集群。**在更高安全性的需求下，可以将允许加入集群的主机明确写入以行分割形式写入一个文件，并通过 `dfs.hosts` 指定其文件路径。这样，凡是不在该列表中的主机，都会被拒绝加入集群。

- 使用该配置的有：NN

### dfs.host.exclude

和 `dfs.hosts` 正好相反，该配置是指定**哪些节点需要排除在集群范围之外。** 该文件常被用于在停止某 DN 前通知 NN 之用，这样 DN 可以平滑的退出集群。

- 使用该配置的有：NN

### fs.trash.interval (core-site.xml)

为了防止误删除，HDFS 可以启用回收站功能。同桌面系统一样，所删除的东西会先移入回收站一段时间，然后再被删除。`fs.trash.interval`所指定的就是这“一段时间”长短（以分钟为单位）。用户可以显式的通过命令 `hadoop fs -expunge` 清空回收站；用户也可以通过为 `hadoop fs -rm` 命令添加 `-skipTrash` 来直接删除。** 回收站功能只支持命令行文件操作。**

- 使用该配置的有：NN，客户端

格式化 NameNode
-----------------

- **在第一次启动 HDFS 前，必须格式化 NameNode。**
- **格式化必须使用拥有 Hadoop 超级用户身份的用户格式化。**

格式化 NameNode 会在 `dfs.name.dir` 目录中创建一个 `fsimage` 文件、edit log，以及生成一个随机的 Storage ID。DN 第一次连接到 NN 的时候会获得该 ID，并在以后拒绝连入其它 Storage ID 的NN。

** 如果要格式化 NameNode，也一定要删除 DN 中的所有数据。**

** 如果格式化 NameNode，所有的元数据将丢失，即使 DN 中依旧保留着那些文件对应的 block 内容，也将无法再使用这些 block了。**

```bash
sudo -u hdfs hadoop namenode -format
```

创建 /tmp 目录
-----------------

很多应用要求 HDFS 中存在 `/tmp` 目录，用以存放临时文件。该目录应该是所有人都可写的。

```bash
hadoop fs -ls /
sudo -u hdfs hadoop fs -mkdir /tmp
sudo -u hdfs hadoop fs -chmod 1777 /tmp
hadoop fs -ls /
```

NameNode High Availability
============================

Hadoop 1.x 以来，最引起关注的就是 NameNode 的可用性问题。如其构架所述，NameNode 一旦挂掉，HDFS 就等于挂掉。在基础构架上经常采用一些高可用性的措施，如 RAID、双电源热备、双网卡热备等，来保证 NameNode 不会轻易挂掉。不过这只是一种折衷的办法。最好从机制上 NameNode 就具有高可用性。

从 Hadoop 0.23 以来，NameNode High Availability 就成为一个很受关注的特性。CDH 4 第一次引入了 Hadoop 2.0，以尝试支持该特性，并且在 CDH 5 中，开始将 Hadoop 2.3 作为默认的版本。

> 由于 NN HA 是 2.0 的特性，因此本节将使用 2.x 的新配置名称。

使用 NFS 建立 NameNode High Availability
-----------------------------------------

和之前一样，需要配置一个高可用的 NFS 服务器，并且挂载到两个 NN 上。假定其挂载点为 `/mnt/filer/namenode-shared`，称其为**共享编辑目录**。通常情况，挂载时需要加入参数 `tcp,hard,intr`，以及至少版本为 `nfsvers=3`。此外需要配置合适的 `timeo` 和 `retrans` 值。可以通过 `man 5 nfs` 来查看文档。

如果共享编辑目录不可写入或者不可用，则 NN 会退出。这与 `dfs.name.dir` 不同，对于 `dfs.name.dir`来说，如果提供的某目录不可用，那么只会忽略，而不会导致 NN 退出。由于需要确保两个 NN 都可以读写共享编辑目录，很重要的一件事是，**需要保证同样用户在不同服务器中的 `uid` 要一致**。

接下来的配置将集中于 `core-site.xml` 与 `hdfs-site.xml` 中。

由于存在多个 NN 来表示一个 NameService，因此需要一个逻辑的名称来代替之前的 NameNode Id，这个名称被称为 `nameservice-id`。

隔离(Fencing)方法
-------------------

当活跃的 NN 变得不再响应时，需要有些机制可以确保不健康的 NN 放弃其操作，并有等候的 NN 变为活跃。因为在某些情况下，虽然 NN 停止响应了，但是其进程并没有意识到自己已出故障，而继续向共享编辑目录写入信息，这会导致多个 NN 写入一个数据，而导致数据损坏。**这种设法让不健康 NN 终止其行为的办法称为隔离(Fencing)。**

Fencing的办法有很多种。比如，通过 RPC 或者 `kill -15` 来平和的请求 NN 进程退出；或者通过 `kill -9` 强制进程退出；或者通过电源管理的组件直接切断该 NN 主机的电源（通常称为一枪爆头 STONITH）；或者告之共享存储的媒介(如NFS)禁止该失效的 NN 写入。

在实际中，应该使用多种办法，以防止一种办法失效的情况。**Fencing 的办法应该按照顺序执行，其中最平和的办法放到最前面，最严厉的办法放到后面。**

Hadoop 内置了一个标准的 **RPC** fencing和两个用户可配置的 **sshfence** 与 **shell** fencing 机制。Hadoop 会先尝试其内置的 RPC 机制来请求 NN 退出，如果不成功，才会执行用户定义在 `dfs.ha.fencing.methods` 配置下的方法。

- `sshfence`： 利用 ssh 连入要终止的 NN 主机，利用 `fuser` 来确定所听端口的进程，并终止该进程。其格式为 `sshfence(username:port)`。需要注意的是，应该配置好 SSH 的密钥，这样可以无密码登录。由于无法判断目标主机是真的已经关闭了，还是 ssh 无响应，因此如果当对方主机已关闭的时候，`sshfence`不会返回成功，而会返回失败。因此必须准备一个非 ssh 的fencing方式，来在其失败时进一步的操作。

- `shell`： 顾名思义，允许指定一个 shell 脚本或命令，因此这个选项非常灵活。其中需要注意的是 Hadoop 的配置属性会位于环境变量，只不过 `.` 变成了 `_`， 如：

  - `$target_host`: 目标主机；
  - `$target_port`: 目标主机的 RPC 端口；
  - `$target_address`: `$target_host:$target_port`；
  - `$target_nameserviceid`: NameService ID；
  - `$target_namenodeid`: NameNode ID；

`shell` 的形式为：

```bash
shell(/path/to/script.py --namenode=$target_host --nameservice=$target_nameserviceid)
```

脚本如果成功，应返回0；如果失败返回非0的值。

**如果所有的 fencing 方法都失败的话，那么故障切换(failover)的行为就会中止，需要人为的介入来解决问题，以防止盲目的切换导致数据损毁。**

需要注意的是，脚本没有超时机制，因此如果所写脚本未能退出，则切换控制行为将永远不会发生。

基本配置
----------

如未说明，下面的配置都是在 `hdfs-site.xml` 中的：

### dfs.nameservices

指定 NameService ID，既一对 NN 所表示的逻辑名。HA 和 Federation 都将使用该配置。

- 使用该配置的有：NN、客户端

### dfs.ha.namenodes.<nameservice-id>

指定 NameService ID 所代表的 NN 列表，以逗号分割，如：`nn1,nn2`。其中 `<nameservice-id>` 表示的是 `dfs.nameservices` 中指定的 id。

- 使用该配置的有：NN、客户端

### dfs.namenode.rpc-address.<nameservice-id>.<namenode-id>

指定对应 `<nameservice-id>` 中的 `<namenode-id>` 的 RPC 主机名和端口号，由冒号分割。如 `hadoop01.mycompany.com:8020`。

- 使用该配置的有：NN、客户端


### dfs.namenode.http-address.<nameservice-id>.<namenode-id>

指定对应 `<nameservice-id>` 中的 `<namenode-id>` 的 HTTP的 主机名和端口号，由冒号分割。如 `hadoop01.mycompany.com:50070`。

- 使用该配置的有：NN


### dfs.namenode.shared.edits.dir

NameNode HA 均可以访问到的共享文件系统。形式应为 `file://` 的URL，如：`file:///mnt/namenode/prod-analytics-edits`。

- 使用该配置的有：NN

### dfs.client.failover.proxy.provider.<nameservice-id>

当使用 HA 的时候，客户端需要知道如何判断那个 NN 是活动状态。这个选项指定了一个类，该类将会被用于定位活动状态的 NN。当前 Hadoop 只提供了一个类，既：`org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider`。

- 使用该配置的有：客户端

### dfs.ha.fencing.methods

行分割的隔离(fencing)方式列表，内容如前所述。如：

```
sshfence(hdfs:22)
shell(/path/to/fence-nfs-filer.sh --host=$target_host)
shell(/path/to/pdu-controller.sh --disable-power --host=$target_host)
```

- 使用该配置的有：NN、ZKFC

自动故障切换配置
-----------------

现在默认配置下，NN HA 是需要手动故障切换。如果需要可以设置自动切换，自动切换需要额外的添加两个组件：

- Apache ZooKeeper： 分布式锁定、协作、配置服务。需要额外安装。

- The ZooKeeper Failover Controller (ZKFC)： 这个服务将随 NN 运行，用于监控 NN 的健康状态、维护 ZooKeeper 的会话信息、在必要的时候发起状态转换和隔离。这部分包含在 Hadoop 中，不需要额外安装，但需额外配置。

### 配置

需要进行下列设置，才可以实现自动故障切换：

#### dfs.ha.automatic-failover.enabled （hdfs-site.xml)

当该项为 `true` 时，会额外启动 ZKFC 来监控 NN 状态，以实施自动故障切换。并且，如果启用该选项，则必须合理配置 `ha.zookeeper.quorum` 指向用于控制故障切换的 ZooKeeper quorum。

- 默认值：`false`
- 示例：`true`
- 使用该配置的有：NN、ZKFC、客户端

#### ha.zookeeper.quorum (core-site.xml)

要使用自动故障切换功能，这里必须指定 ZooKeeper 的成员。格式为逗号分割的成员信息，成员则有主机名和端口号组成。至少要有三个 ZooKeeper。

- 示例：zk-node1:2181,zk-node2:2181,zk-node3:2181
- 使用该配置的有：ZKFC

### 初始化 ZooKeeper 状态

在自动故障切换可以使用之前，需要先对 ZooKeeper 进行必要的初始化。命令为 `hdfs zkfc -formatZK`，该命令需要使用 HDFS 的 super user (既格式化 HDFS 的用户) 的用户来执行。当然，执行该命令前，需要都已经配置好，并且 ZooKeeper 已然在运行。

```bash
hdfs zkfc -formatZK
```

格式化并启动(bootstrap) NameNode
----------------------

随便选择一个 NN 作为主(primary) NN，这只是初始状态，所以选择任何一个都可以。选择后，另一个称为后备(standby) NN。

使用 `hdfs namenode -format` 在主NN上格式化 NameNode。这将会格式化本地目录以及共享编辑目录。

```bash
sudo -u hdfs hdfs namenode -format
```

在standby NN 上，需要从 primary NN 上取回一份元数据信息并准备，可以使用命令 `hdfs namenode -bootstrapStandby`。

不过在此之前，需要先让 primary NN 启动并成为 active 状态。

启动 primary NN 很简单，只需要在 primary NN 上执行正常的启动指令即可。

```bash
sudo service hadoop-hdfs-namenode start
sudo service hadoop-hdfs-zkfc start
```

如果**没有配置自动故障切换，则需要手动激活。** 通过 `-failover` 参数加两个 NN 的主机名，其顺序为，A ⇨ B，A 为 StandBy，B 为 Active。

```bash
hdfs haadmin -failover hadoop-ha02 hadoop-ha01
```

准备好后，就可以在 Standby NN 上执行 `-bootstrapStandby` 了。

```bash
sudo -u hdfs hdfs namenode -bootstrapStandby
```

启动 standby NN 时需注意，如果配置了自动故障切换，同 primary NN 上一样，也需要启动 ZKFC。

```bash
sudo service hadoop-hdfs-namenode start
sudo service hadoop-hdfs-zkfc start
```

测试自动故障切换是否成功，可以直接将 primary NN 上的 NN 进程杀掉，然后观察 FC 的日志，正常情况会看到健康状况的状态切换，以及 ZooKeeper 锁的状态切换。

Namenode Federation
=====================


MapReduce
=============


机架拓扑
===========


安全
======




