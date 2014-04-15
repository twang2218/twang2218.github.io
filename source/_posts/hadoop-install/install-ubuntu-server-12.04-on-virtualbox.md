---
layout: post
category: tutorial
title: 安装 Ubuntu Server 12.04 64位虚拟机
tags : [hadoop, hadoop install]
date: 2014/02/17
---

一、选择搭建环境
==============


1.1 下载安装 VirtualBox
-----------------------

我们使用免费的 [VirtualBox](https://www.virtualbox.org/) 虚拟机，这是一款出色的跨平台的虚拟机软件，可以在一个真实系统中虚拟出一个计算机，我们将在虚拟计算机中安装 Ubuntu。这样做的好处是最小化干扰当前的系统，我们可以在虚拟机中做许多事情，对主机都不会有影响。而且，在将来的过程中，我们会实现多节点的全分布模式部署 Hadoop，有了虚拟机，我们仅通过1-2台计算机就可以架设10多台虚拟机以完成实验。这在没有虚拟机的情况下，实验将非常麻烦。

从 [VirtualBox 官方网站](https://www.virtualbox.org/wiki/Downloads)下载最新版的 VirtualBox。VirtualBox 支持跨平台，包括 [Mac OS X](http://download.virtualbox.org/virtualbox/4.3.8/VirtualBox-4.3.8-92456-OSX.dmg)、[Linux](https://www.virtualbox.org/wiki/Linux_Downloads) 以及 [Windows](http://download.virtualbox.org/virtualbox/4.3.8/VirtualBox-4.3.8-92456-Win.exe)。根据自己的需要可以选择相应的安装包下载安装。VirtualBox 的安装过程这里就不赘述，非常简单，一路下一步即可。不清楚的可以到网上找教程，这类教程非常多。


1.2 下载 Ubuntu Server ISO
---------------------------


在这个实验中，我选择 [Ubuntu 服务器版](http://www.ubuntu.com/server/)。Ubuntu 从2004年诞生以来，发展非常迅速，在 Web 服务器市场上，[仅次于](http://w3techs.com/technologies/history_details/os-linux/all/y) [Debian](https://www.debian.org/)，而在当前云计算中，甚至成为服务器的[首选](http://thecloudmarket.com/stats#/by_platform_definition)。Ubuntu 服务器版本不同于桌面版本，它是为服务器定制的，默认是没有图形操作界面的，这正是出于服务器所需要的高稳定性、高安全性和低系统资源占用的考虑。对于我们使用虚拟机架设实验环境，这是非常合适的。Ubuntu 服务器版本提供了专门针对虚拟机的最优化配置，安装的时候可以选择，这种配置可以保证内核最精简，启动后系统仅占不到60MB的内存。这可以让我们在一台计算机中虚拟更多的虚拟机。

访问 http://www.ubuntu.org.cn/download/server 点击`“获取 Ubuntu 12.04 LTS”`。保存到一个指定的位置。

1.3 网络环境
-------------


关于网络环境，我们假设物理计算机存在一个可连接的局域网环境，局域网中有默认的`DHCP`和`DNS`以及网关，局域网中的计算机，可以不经过配置，直接通过`DHCP`上网。这一般是常见的台式机环境。


二、添加虚拟机
=============


虚拟机装好之后，我们可以点击运行 VirtualBox，接下来，我们开始添加虚拟机。点击新建。

我们输入电脑名称为 `hadoop-master`，类型为 `Linux`，版本为 `Ubuntu (64 bit)`。生产环境由于内存原因，往往都是64位的系统，所以我们实验也以64位系统为主。

![vbox-01](/pic/vbox/vbox-01.png)

虽然 Ubuntu 系统很省内存，但是 Hadoop 需要一定的空间。这里内存最小也需要 1GB，调整为 2GB 性能会好一些。

![vbox-02](/pic/vbox/vbox-02.png)

这里需要创建新的虚拟硬盘。这个虚拟硬盘将是在物理机上生成的一个或多个文件组成，在虚拟机里面改动分区、格式化等只是针对那个文件操作而已，不会影响主系统。

![vbox-03](/pic/vbox/vbox-03.png)

VirtualBox 支持多种虚拟机硬盘格式，在这里我们选择默认的 `VDI`。

![vbox-04](/pic/vbox/vbox-04.png)

刚才说过虚拟硬盘实际上是一个物理机上面的文件。那么这个文件的生成就有两种方式，一种是简单的固定大小，也就是说，如果虚拟硬盘设置的是20GB，那么就直接创建一个20GB的大文件。这可能对于硬盘比较紧张、或者可能虚拟机系统内部不需要那么多硬盘空间的情况是一种浪费，因此这里我们选择另一种方式，动态分配。也就是用多少，分配多少空间。这样的话性能可能有些影响，但是很节约磁盘空间。

![vbox-05](/pic/vbox/vbox-05.png)

如果默认位置的硬盘空间不足，这里可以选择磁盘文件的位置到另一块硬盘。默认的8GB大小其实已经足够。但是这里我分配了30GB，仅仅是防止意外情况出现。反正动态分配，只要不真用那么多空间，无所谓的。

![vbox-06](/pic/vbox/vbox-06.png)

通过上面的新建虚拟机向导我们就添加了一个虚拟机。不过这里我们还需要进行更细节的一些配置。右键点击新建的虚拟机，选择`"设置"` ⇨ `"系统"` ⇨ `"处理器"`。这里我们勾选上`"启用 PAE/NX"`。

![vbox-07](/pic/vbox/vbox-07.png)

然后是修改网络设置，这里我们可以为虚拟机添加多块儿网卡，这在做 OpenStack 实验的时候很有用处。不过这里我们只需要有一块儿网卡就足够了。连接方式默认是`NAT`，这里我们选择`"桥接"`。桥接模式是指将虚拟机放置在与物理机同级的位置，从网络中看过来，会认为虚拟机和物理机是两台独立计算机。这对于我们配置东西很方便。这里不要选择 `NAT` 模式，`NAT`随然可以上网，但是会导致同在 `NAT` 的主机无法互相通讯。这里如果选择`“仅主机”`会导致虚拟机互相之间可以通讯，却无法访问外部网络。`“界面名称”`也就是网卡名称，我这里选择了 `eth0`是我的物理网卡。

![vbox-08](/pic/vbox/vbox-08.png)

我们虚拟机中的Ubuntu系统安装将使用光盘安装方式，当然，由于是虚拟机，我们并不需要刻录一张光盘，可以用虚拟光盘即可。依照图中点击右侧光盘按钮 ⇨ `"选择一个虚拟光盘..."`

![vbox-09](/pic/vbox/vbox-09.png)

然后选取我们之前下载的 Ubuntu Server 12.04 LTS 的 ISO 文件。然后点击打开即可。

![vbox-10](/pic/vbox/vbox-10.png)

点击`“确定”`后，我们的虚拟机就建立完毕了。

![vbox-11](/pic/vbox/vbox-11.png)


三、安装 Ubuntu Server
======================



点击启动后，虚拟机就启动了。经过虚拟BIOS界面后，就会由刚才所选择的光盘进行引导，进入 Ubuntu Server 安装画面。

这里我们选择 `English`。这对于我们排障很有帮助，有些时候命令行里面将无法显示中文文字，如果我们选择了中文，可能会乱码，根本无法看清错误原因，非常不利于排障。而且，由于报错是中文的，在 Google 上搜索报错的时候，也会带来困难，只能搜索到很少的资料。所以这里一定要选择英文。

![install-1](/pic/vbox/install-1.png)

在开始安装之前，我们可以按 `F4` 来选择 `Install a minimal virtual machine`。这是我们之前提到过的 Ubuntu Server 为虚拟机量身定做的精简系统。当然，使用默认的 `Normal` 也可以完成实验，但是出于对系统资源的占用考虑，我建议使用这个优化选项。

![install-2](/pic/vbox/install-2.png)

回车选定后，我们就可以一路回车安装了。

**请注意，在接下来的安装过程中，凡是下面提到的安装界面，都是需要变动的；所有不需要变动的，只需要一路回车即可，这里就不赘述了。**

这里的位置可能需要改动，选择 `other` ⇨ `Asia` ⇨ `China`

![install-3](/pic/vbox/install-3.png)

![install-4](/pic/vbox/install-4.png)

主机名我们可以保持默认，反正后续的 Hadoop 配置过程我们也会改动它。

![install-5](/pic/vbox/install-5.png)

输入用户全名，可以是任意名字，我这里用的是 [`Wombat`](http://www.australiazoo.com.au/our-animals/animal-encounters/wombat-encounter.html)。系统会自动生成一个用户名 `wombat`。用户名要小写、不包含空格。

![install-6](/pic/vbox/install-6.png)

这里输入密码，重复一次确认无误即可。

![install-7](/pic/vbox/install-7.png)

如果输入的密码过于简单，会有这样的提示。这是安全提示，在生产环境中，我们应该 `Go Back` 选择一个安全的密码。但是，在我们的实验环境中，可以忽略这个提示，选择 `Yes`。

![install-8](/pic/vbox/install-8.png)

当选择时区的时候请注意，中国标准时间是北京时间，可是列表中没有北京。这是历史原因导致的。在1949年之前，民国时期，中国曾经出现过[昆仑时区、回藏时区、陇蜀时区、中原时区，以及长白时区](http://zh.wikipedia.org/wiki/%E4%B8%AD%E5%9C%8B%E6%99%82%E5%8D%80)。其中，中原时区是GMT+8时区，也就是今天的北京时间，而当年，是以上海租界内的[徐家汇观象台所](http://baike.baidu.com/view/1949604.htm)提供。所以当时建立时，以上海为名。建国后，政府以上海的中原时区统一了全国时间，但是改以北京为名。因此，这里我们应当选择的是`“Shanghai”`。

![install-9](/pic/vbox/install-9.png)

一路回车，当询问是否写入分区时，选择 `Yes`，写入分区。

![install-10](/pic/vbox/install-10.png)

再一路回车后，其中可能会有的环节需要从网络下载安装包。之前的位置和时区选择会帮助 Ubuntu 自动选择中国镜像，所以不必担心速度。下载后一路回车，就会重启，进入登录界面了。

![install-11](/pic/vbox/install-11.png)


至此，安装 Ubuntu Server 12.04 LTS 64 位版的工作就做完了。我们可以继续下面的 Hadoop 安装准备工作了。
