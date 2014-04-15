---
layout: post
category: tutorial
title: 准备 Hadoop 运行环境
tags : [hadoop]
tags : [hadoop, hadoop install]
date: 2014/02/18
---

一、安装所必须的软件
====================


首先，Hadoop 需要 Java 运行环境。

| Base |  |
| --- | --- |
| `openjdk-7-jdk` | 这里，我们使用系统自带的 OpenJDK 7。关于是否应该使用 Oracle 的 JDK 的问题，在这里完全没有必要。在 Java 6 的年代，由于 Java 的开放源代码进程问题，OpenJDK 6 和 Oracle JDK 6 还是有一些不一致的，所以在 JAVA 6 的年代，推荐使用 Oracle JDK 以避免不必要的麻烦。[在进入到 Java 7后](https://blogs.oracle.com/henrik/entry/moving_to_openjdk_as_the)，OpenJDK 和 Oracle JDK [已经一致了](https://jdk7.java.net/java-se-7-ri/)，而且，这一点 Hadoop 官方已经[明确给出确认](http://wiki.apache.org/hadoop/HadoopJavaVersions)，所以可以放心的使用 OpenJDK 7。 |
| `openssh-server` | Hadoop 需要使用 SSH 连接各个主机。即使是伪分布模式，也需要连接本机。所以需要安装 SSH 服务器。 |
| `nano` | 我们编辑配置文件的时候，我将使用 `nano` 编辑器。我知道许多人会选择使用 `vi`，但是从易用性上，`nano` 要比 `vi` 简单。使用 `nano` 不需要去背那些 `vi` 命令和快捷键，很适合新手。`nano` 在许多 ubuntu 系统里都是默认安装的，不需要额外的安装的。在此写进安装命令只是以防万一 （比如在使用 `vmbuilder` 构建的精简版的 Ubuntu 中，很多命令都不会被安装）。 |

和 Windows 不同，在 Ubuntu 里，安装软件非常简单，只需要一条命令即可，系统会从网上找到需要的软件包进行下载并安装。而且，你不需要担心版本之间的依赖关系、是否互相有冲突、安装位置等等。

安装上面的软件，只需执行：

```bash
sudo apt-get install openjdk-7-jdk openssh-server nano
```


除了上面所必需的软件外，为了方便调试，我们还会安装如下软件：

| Extra |  |
| --- | --- |
| `unzip` | 我们将来会用它对下载的 `.zip` 数据文件进行解压缩。 |
| `htop` | 使用 `htop` 对节点负载进行分析与观察是非常有帮助的。 |
| `tree` | 使用 `tree` 命令可以很方便的列出目录内部的树形结构。 |
| `dnsutils` | 其中包括两个在调试 DNS 时很常用的命令 `nslookup` 和 `dig`。 |
| `tcptrack` | 监视 TCP 连接的变化情况，方便调试。 |
| `command-not-found` | 这是一个帮助类的软件包，当你你使用的命令未安装的时候，它会告诉你命令在哪个软件里，以及如何安装。 |


```bash
sudo apt-get install htop tree dnsutils tcptrack command-not-found
```

当询问是否安装的时候，按 `y` 回车

> 如果虚拟机是从另一个虚拟机克隆出来的，其网卡的MAC地址应该会被重置。重置MAC地址之后会导致 Linux 安全固化部分的机制启动，会让新的MAC地址使用下一个eth编号。因此，当使用 `ifconfig -a` 的时候，会发现原来的 `eth0` 没有了，多了一个新的 `eth1`。而且会因此导致默认的 `/etc/network/interfaces` 针对 `eth0` 的配置失效，引导的时候会出现无谓的等待120秒网络配置的情况，并且最关键的是，网络无法访问了。
>
> 解决办法很简单，直接删除掉规则文件 `sudo rm /etc/udev/rules.d/70-persistent-net.rules` ，重启一下虚拟机，就会自动生成新的对应于 `eth0` 的规则文件。重启后，网络就可以使用了。



二、配置 SSH 无密码登陆
=======================


Hadoop 的 master 在控制 slaves 的时候，比如 NameNode 启动、停止 DataNode 等，需要使用 SSH 连接相应的节点。如果不配置无密码登陆，那么在启动停止的时候，都需要人工手动输入各个节点的密码。这是不现实的。

那么既保证安全，又能够避免麻烦的办法有两种，一种是使用 `ssh-agent`，在生产环境中可以考虑使用，配置较复杂；另一种则是使用 SSH 密钥登陆，这个操作非常简单，很适合实验环境配置，也是绝大多数网上教程使用的方法。

其原理很简单，只是简单的非对称加密，大学学过应用密码学课程的人都应该很熟悉，记不清的可以搜索关键字，非对称加密、RSA 等。在这里不赘述了。

在 Ubuntu 上配置无密码登陆非常容易。只需要两条命令。

首先，我们需要生成一对儿密钥：公钥和私钥。只需要使用命令：

```bash
ssh-keygen
```

一路回车即可，不需要输入任何东西，就生成好了。这条命令会将生成的密钥对放入 `~/.ssh` 目录下，因此，可以执行 `ls ~/.ssh` 去确认该文件夹里面是否已经成功存在所生成的密钥。

然后，我们需要将公钥添加到目标主机的授权列表里。比如，在这里我们将先配置 Hadoop 伪分布模式，因此我们需要保证 SSH 登录本机的时候，不需要密码。如果之前没有配置过使用密钥登录的，`ssh localhost` 会提示用户输入密码。这是我们不希望出现的。于是我们用下面的这一条命令来将公钥添加到本地的授权列表。

```bash
ssh-copy-id -i localhost
```

提示输入本机密码，输入密码后回车，这样就把公钥添加到本机的授权列表了。可以执行 `ls ~/.ssh`，会发现，目录里面多了一个 `authorized_keys` 文件，这个文件就是授权公钥列表文件。这里面包含的就是我们刚刚提交的公钥。有了这个，我们登录本机就不需要密码了。

在命令行里再登录一下试试看：

```bash
ssh localhost
```

如果配置都是正确的，会发现，没有刚才的密码提示了。

> 注意到网上许多教程在配置无密码登录的时候，都是用的是命令 `cat ~/.ssh/id_rsa.pub >> ~/.ssh/authroized_keys` 这将直接手动修改授权文件 `authroized_keys`。这是比较古老的方法，可以工作，但是已不推荐使用了，因为如果命令敲错，比如少打了一个大于号，就会导致认证密钥列表被覆盖，从而导致其它的一些应用登录可能出现问题，又或者授权文件的文件名敲错，而不会有任何提示，结果还是无法无密码登录等。而且，这个方法在跨主机的时候就更复杂了，还需要把文件先复制到目标主机上。因此，除非有什么特殊的原因，推荐使用刚才所描述的 `ssh-copy-id`。


三、配置主机名及静态IP地址
==========================


对于单节点的伪分布模式，本步骤可以忽略，只要在 Hadoop 配置文件中使用 `localhost` 就可以启动。完全不会有问题。但是，在多节点全分布模式下，这一步就是必须的了。在这里配置的原因是考虑到将来还会用这个节点做多节点实验。

此外，还有一个原因是，如果你的 Ubuntu 是运行于虚拟机中的，那么配置好了静态IP和主机名，就会很方便外部主机通过浏览器访问 Hadoop 的界面，比如查看 Hadoop 状态、浏览 HDFS 之类的。如果没有配置那些，也可以访问，但是会有一些麻烦。

在配置之前，我们先定义一下将要配置网络环境。

我们的 Ubuntu 是安装在 VirtualBox 虚拟机中的，网卡连接方式的是 VirtualBox 虚拟机的桥接网卡，桥接表明虚拟机和实际的物理机处于同级的位置，也就是说从外部看来，虚拟机和主机是局域网中两台机器，因此同属局域网的网段。

该网络的网段是： `10.0.1.0/24`

网关是： `10.0.1.1`

该网关同时也有 DNS 转发的功能。

我们今天配置的伪分布的单节点，以及将来配置的全分布的各个节点将都在这个网络下，为了方便起见，我们先定义一下各个节点的 IP 和主机名：

```bash
10.0.1.110 hadoop-master
10.0.1.111 hadoop-secondary
10.0.1.121 hadoop-data1
10.0.1.122 hadoop-data2
10.0.1.123 hadoop-data3
```

前面是该节点应该使用的IP，后面是该节点的主机名。今天我们将在 `hadoop-master` 上面配置单节点伪分布模式。

下面开始配置。



3.1 修改本机主机名
--------------------


```bash
sudo nano /etc/hostname
```


这是一个一行内容的文本文件，将其修改为 `hadoop-master`。

然后按 `Ctrl+x`，会提示已修改，是否保存，输入 `y` 回车，还会再确认一下文件名，直接回车即可，文件就保存了。



3.2 配置静态 IP
----------------

```bash
sudo nano /etc/network/interfaces
```

将其中的

```bash
auto eth0
iface eth0 inet dhcp
```

替换为

```bash
auto eth0
iface eth0 inet static
    address 10.0.1.110
    netmask 255.255.255.0
    gateway 10.0.1.1
    dns-nameservers 10.0.1.1 8.8.8.8
```

和刚才一样，`Ctrl+x`，保存退出。

> 我们此处配置的都是 Ubuntu 服务器版，如果配置的是 Ubuntu 桌面版，则很有可能 `/etc/network/interfaces` 中没有 `auto eth0` 项。这是由于 `eth0` 由图形界面配置工具 `NetworkManager` 所接管。我们要取代 `NetworkManager` 管理的办法很简单，就是添加上面要替换的内容即可。


3.3 配置主机名和 IP 对应关系
----------------------------


```bash
sudo nano /etc/hosts
```


在文件尾部添加：

```bash
10.0.1.110 hadoop-master
10.0.1.111 hadoop-secondary
10.0.1.121 hadoop-data1
10.0.1.122 hadoop-data2
10.0.1.123 hadoop-data3
```
 
保存退出。



3.4 使配置生效
----------------


```bash
sudo ifdown eth0 && sudo ifup eth0
```

可以用 `ifconfig` 来检查自己的网络配置，或者直接 `ssh hadoop-master` 试一下。如果碰到错误，检查一下配置文件。

从这里开始，我们就可以不用在虚拟机窗口中配置了，我们可以使用 SSH 客户端（即终端），如[Putty](http://www.putty.org/)、[Bitvise SSH Client](http://www.bitvise.com/ssh-client-download)、[WinSCP](http://winscp.net/eng/docs/free_ssh_client_for_windows)、[Xshell](http://www.netsarang.com/products/xsh_overview.html)等工具来连接 `hadoop-master`。

使用终端的好处是显而易见的，比如我们可以方便的复制、粘贴文字到终端窗口，也可以将里面的内容复制、粘贴出来。此外，上面的许多 SSH 客户端支持 SFTP，通过 SFTP 我们可以很轻松的将文件上传到 Linux 主机上。

**需要注意的是，在主机上也需要配置上述主机名和IP地址对应关系的，对于 Linux 来说，和第三步一样，配置 `/etc/hosts` 文件；对于 Windows 而言，是编辑 `C:\Windows\System32\drivers\etc\hosts` 文件，都是将第3步的内容追加到文件末尾。**



四、禁用 IPv6
==============


由于 Hadoop 只在 IPv4 开发，因此对 IPv6 的兼容性目前还存在问题。因此，最好禁用系统的 IPv6，防止不必要的麻烦。

```bash
sudo nano /etc/sysctl.d/10-disable-ipv6.conf
```

在文件中添加如下内容：

```bash
net.ipv6.conf.all.disable_ipv6=1
```

使配置文件生效：

```bash
sudo sysctl -p /etc/sysctl.d/10-disable-ipv6.conf
```

可以使用命令 `ifconfig | grep inet6` 检查是否已经禁用了 IPv6。如果没有任何结果返回，则说明 IPv6 已经成功禁用了；否则，则说明禁用 IPv6 没有成功。



五、配置 `JAVA_HOME` 环境变量
==============================


Hadoop 需要在使用 `JAVA_HOME` 环境变量。由于该环境变量将在多处使用，并会在 `ssh + 主机名 + 命令` 的形式使用，因此，我们将在全局环境变量中配置：

```bash
sudo nano /etc/environment
```

在文件最后添加：

```bash
JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/
```

`Ctrl+x`，保存退出。

这里虽然也有 PATH 环境变量，但是不要在这里修改它，这里是全局变量。


>有些教程建议修改 `/etc/profile` 文件，可是修改该文件后，`JAVA_HOME` 依旧是无法出现在 `ssh` 命令模式下的环境中，就导致了还必须同时设置 `conf/hadoop-env.sh` 文件中的 `JAVA_HOME`，否则，将会出现 `Error: JAVA_HOME is not set.` 的故障。可能是写教程的人不熟悉 `/etc/profile` 和 `/etc/environment` 的加载时间所致。这里我们只需修改 `/etc/environment`文件即可。

然后，`sudo reboot`，重新启动虚拟机，以让全局环境变量生效。

> 可以使用命令 `ssh localhost env | grep JAVA_HOME` 来判断是否设置正确，如果设置正确会看到 `JAVA_HOME` 的配置，如果返回为空，则说明设置依旧有错。

