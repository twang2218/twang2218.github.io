---
layout: post
category: readings
title: 《Hadoop Operations》读书笔记 - 3
tags : [hadoop]
---
{% blockquote Eric Sammer http://shop.oreilly.com/product/0636920025085.do "Hadoop Operations" - O'Reilly (2012) ... (p41 ~ p73) %}{% endblockquote %}


第四章 Planning a Hadoop Cluster
==================================


选择 Hadoop 发布以及版本
--------------------------

计划部署 Hadoop 集群的第一件事情就是选择 Hadoop 的发布和版本。需要开发人员、分析师、以及BI类其他系统共同来决定。

一般提到 Hadoop 往往除了 Hadoop 核心外，还会需要其生态圈的其它组成部分。所有这些组成部分必须要考虑到兼容性的问题，包括二进制兼容和API兼容。

### Apache Hadoop

在 1.0 以前，Apache 很久才会发布一个版本，不过现在要频繁多了。从 1.0 之后， Apache 的发布除了原有的 tarball 外，还增加了 RPM 和 Deb 来满足 RedHat和Debian两大包管理体系的需要。

### CDH (Cloudera's Distribution Including Apache Hadoop)

Cloudera 是一个为 Hadoop 提供管理、咨询、技术支持的公司。他们有自己的发行，称为 CDH。而且像 Apache Hadoop 一样，CDH发布的软件都是开源的。

Cloudera 会基于一个 Hadoop 的稳定版本，然后积极维护一些补丁，而且负责处理一些与系统相关的版本问题。Cloudera 公司的很多雇员同时也是 Apache Hadoop 的 committer （committer 是指有权利向 Apache 提交源代码的人，这些人一般是这个项目的积极维护者）。

由于大多数人部署 Hadoop 都是部署很多生态圈相关的部分，所以 CDH 除了 Hadoop 本身外，还包含了 HBase, Hive, Pig, Sqoop, Flume, ZooKeeper, Oozie, Mahout, Hue, Impala 等。

CDH 基本每年一个主要的版本发布，大约每个季度会有以修复补丁为主的小版本发布。CDH4包含了 Hadoop 2.0 因此包含了 YARN。即将发布的 CDH 5 将基于 Hadoop 2.3。

CDH 同样提供了 tarball 安装，以及 RHEL 5/6 的RPM包，SuSE 的 RPM包，以及 Debian 的 Deb包。此外，还提供了包管理软件 yum, zypper, apt 的软件库的源，因此各个系统部署非常简单。

### 版本和功能

选择版本的考虑主要有以下几个方面：

- 所需要的稳定程度
- 所需要的功能

Hadoop 的版本号是很混乱的。

