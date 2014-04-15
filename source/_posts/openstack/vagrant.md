---
layout: post
category: tutorial
title: Vagrant 简介
tags : [vagrant]
date: 2014/03/01
---

一、什么是 Vagrant
=================

开发过程中，我们经常碰到一个问题，总有成员会抱怨说：“我这里编译出错啊”，或者，“这个页面在我机器上运行不了啊”。这类问题层出不穷，大多是由于开发环境搭建过程中的不一致所产生。而我们往往无法苛求开发人员所使用的桌面环境必须保持完全一致，毕竟每个人都有自己的喜好。而在云计算实验环境搭建过程中，这个问题就更明显了。经常有人说跟着某教程最后得不出一样的结果，或许教程确实有错，但也有很多的时候，是实验者由于疏忽，遗漏了某些环节所致。

怎么解决这个问题呢？我们都知道 [VirtualBox](https://www.virtualbox.org/) 是一个虚拟机，我们可以在上面虚拟一台或多台完整的计算机系统。那么也许可以由团队创建一个用于开发的虚拟机，确保这个虚拟机是可以完成工作的，然后将其分发给各个成员，让他们按照指示去搭建环境。

嗯，这在一定程度上可以解决问题。但是，每次建立虚拟机的时候，总有许多参数需要设置，总是或多或少遗漏了些什么，最后导致虚拟机和需求不一样。特别是在实验云计算的时候，需要跑多个虚拟机，配置也非常复杂，尤其是网络配置，稍有不慎，就会导致环境搭建出现问题。而且，开发过程中，很可能会由于人为的错误，导致虚拟环境出现故障，需要重新搭建环境。这就会将这种问题放大很多。

而 [Vagrant](http://www.vagrantup.com/) 的出现，则很好的解决了上面的问题。Vagrant 可以很好的结合虚拟机（如VirtualBox、VMWare），根据配置文件，轻松的创建多台虚拟机实验环境。我们可以在配置文件中指定，包括从哪里去下载这个虚拟机、网络该如何联通、主机名、IP地址，甚至可以指定开机后自动配置的脚本。

我们可以利用 Vagrant 为开发环境手动配置一台虚拟机，将其放到指定位置后，把配置文件分发给各个成员。在成员的计算机上使用 `vagrant` 的各种命令，就可以轻松的做到一条指令就创建好多台虚拟机配置。而且，不必担心虚拟机环境在操作中被破坏掉，因为只需简单的几条命令，这几台虚拟机就可以抛弃掉，重新创建一个完好如初的虚拟环境。也正是由于这种便利性，[《OpenStack Cloud Computing Cookbook》](http://www.packtpub.com/openstack-cloud-computing-cookbook-second-edition/book)选择使用 Vagrant 作为实验环境搭建的工具。


二、安装
========


Vagrant 可以运行在 Mac OS X、Linux，以及 Windows 上，和 VirtualBox 一样，都是免费的[开源软件](https://github.com/mitchellh/vagrant)。

既然是虚拟机的配置工具，那么我们首先得需要一个虚拟机，这里我们使用 VirtualBox 虚拟机。如果没有安装，直接从官方网站：https://www.virtualbox.org/wiki/Downloads 下载安装即可。

然后，我们从 Vagrant 的官方网站( http://www.vagrantup.com/downloads )下载这个软件，下载后在各自的系统上安装。有的系统（如Windows）可能会需要重新启动。

之后，命令行里面就多了一个 `vagrant` 命令。我们可以通过 `vagrant -v` 来检查所安装的 Vagrant 的版本。


三、浅尝
========

好吧，我们来体验一下这个工具。

假设我们需要一个 Ubuntu 12.04 64位服务器。之前怎么做呢？下载对应的 ISO，然后新建虚拟机，将 ISO 绑定在光盘上，然后一步步的安装。加上下载，估计没有一个小时是搞不定的。从哪个角度说都不是一个轻松的事情。

现在有了 Vagrant，一切都变得简单了起来，我们只需要执行下面的几行命令，就可以做到了：

```bash
mkdir ~/myserver1 && cd ~/myserver1
vagrant init precise64 http://files.vagrantup.com/precise64.box
vagrant up
vagrant ssh
```

> 如果是第一次执行这个命令，需要等待其下载 `precise64.box` 文件，时间视网络情况而定。之后再执行则不必等待下载了。

执行上述命令后，会类似如下的结果：

```bash
tao@wombat:~/myserver1$ vagrant init precise64 http://files.vagrantup.com/precise64.box
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
tao@wombat:~/myserver1$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
[default] Importing base box 'precise64'...
[default] Matching MAC address for NAT networking...
[default] Setting the name of the VM...
[default] Clearing any previously set forwarded ports...
[default] Clearing any previously set network interfaces...
[default] Preparing network interfaces based on configuration...
[default] Forwarding ports...
[default] -- 22 => 2222 (adapter 1)
[default] Booting VM...
[default] Waiting for machine to boot. This may take a few minutes...
[default] Machine booted and ready!
[default] Mounting shared folders...
[default] -- /vagrant
tao@wombat:~/myserver1$ vagrant ssh
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Welcome to your Vagrant-built virtual machine.
Last login: Fri Sep 14 06:23:18 2012 from 10.0.2.2
vagrant@precise64:~$ 
```

可以看到，短暂的等待几分钟后，我们的服务器就准备好了，不但已经启动，而且已经登录进了服务器，我们可以开始做我们想做的事情了。

神奇吧？让我们看看，刚才都做了些什么。

为了测试方便，我们建立了一个新的目录 `~/myserver1`，其后的操作都在这个目录中进行。

而要使 Vagrant 开始工作，我们需要两个前提条件：

- 虚拟机镜像，也称之为 `Box`
- 配置文件 `Vagrantfile`

在初始化的命令中，同时帮我们满足了这两个条件。

```bash
vagrant init precise64 http://files.vagrantup.com/precise64.box
```

第一个参数 `precise64` 是指定`box`的名字，而第二个参数 `http://files.vagrantup.com/precise64.box` 这是一个 URL，指向了所需要的虚拟机镜像的位置。这个位置可以是来自 HTTP，也可以是本地文件。

首先，Vagrant 会检查缓存中是否存在名为 `precise64` 的`box`，如果有就拿来用，如果没有，则到第二个参数中的链接去下载。换句话说，如果我们已经执行过这条命令，缓存存在了 `precise64` 的 `box`，比如我们打算建立个 `myserver2`，那么我们是可以省略第二个参数的。

很多熟识的系统都已经有人准备好了虚拟机镜像，我们可以直接从官方网站：http://www.vagrantbox.es/ 中找到所需的`box`。这里我们根据需要，选择了 `precise64`。当然，你可以选择别的`box`，甚至可以自己定制一个`box`，放到自己的内部服务器上。

随后，Vagrant 会在当前目录创建一个默认的配置文件 `Vagrantfile`，里面默认会写上要使用 `precise64` 做为虚拟机的模板。嗯，至此，我们所需的虚拟机还依旧不存在呢。

此后，我们执行了

```bash
vagrant up
```

这是一个关键的命令。这个命令是告诉 Vagrant，请按照配置文件将所有的虚拟机启动起来。当然，我们现在就一个虚拟机。

Vagrant 会检查当前虚拟机是否已经存在，如果不存在，那么就从指定 `box` 中克隆一个虚拟机，然后，依据配置文件 `Vagrantfile` 中的配置进行各种所需的配置，并且启动该虚拟机。我们可以从上述日志中看到此次启动过程中，根据默认配置文件主要进行了必须的网络设置、主机名设置，以及共享目录的绑定。

最后一条命令：

```bash
vagrant ssh
```

这个命令是通过 `ssh` 连接我们已经启动的虚拟机。我们可以通过上述输出的主机名可以注意到，已经从我本机原来的主机名，变到了 `precise64` 也就是默认的那个主机名。

在虚拟机中，可以执行 `exit`，以退出 `ssh` 连接回到物理机。当然，这并不意味着虚拟机已关机。要关闭虚拟机，我们除了在虚拟机中执行 `sudo poweroff` 外，还可以在物理机执行 `vagrant halt`。

假如我们的虚拟机被我们搞坏了，想抛弃掉，重新来一个虚拟机。也非常简单。

```bash
vagrant destroy
```

这样虚拟机就被扔掉了，我们只需再次执行 `vagrant up`，新的虚拟机就会生成，不出一分钟，我们又可以继续工作了。使用 Vagrant 让这一切都变的轻松起来。

四、进阶
========

4.1 配置文件里都写了些啥？
-----------------------

打开 `Vagrantfile` 文件

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # All Vagrant configuration is done here. The most common configuration
  # options are documented and commented below. For a complete reference,
  # please see the online documentation at vagrantup.com.

  # Every Vagrant virtual environment requires a box to build off of.
  config.vm.box = "precise64"

  # The url from where the 'config.vm.box' box will be fetched if it
  # doesn't already exist on the user's system.
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network :forwarded_port, guest: 80, host: 8080
  
  ...
  
end
```

我们首先会注意到，这是一个 Ruby 文件。没错，Vagrant 就是使用 Ruby 写成的。这里的配置文件也需要满足 Ruby 语法。

其中大部分的代码都已被注释起来，但有两条配置没有被注释掉的：

```ruby
  config.vm.box = "precise64"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"
```

可以看到，这两行就是我们在 `vagrant init` 中后面所指定的参数。由此可以看出，`vagrant init`只是帮我们生成了配置文件而已，换句话说，如果我们写好了 `Vagrantfile`，就不需要 `vagrant init`，只需将准备好的配置文件放入到所需目录中，然后直接执行 `vagrant up` 即可。

还有很多注释掉的配置，那些都是一些常用的配置，包括网卡设置、IP地址、绑定目录，甚至可以指定内存大小、CPU个数、是否启动界面等等。如果需要，可以根据注释文本进行配置。

下面列出一些常用的配置：

- `config.vm.hostname`：配置虚拟机主机名
- `config.vm.network`：这是配置虚拟机网络，由于比较复杂，我们其后单独讨论
- `config.vm.synced_folder`：除了默认的目录绑定外，还可以手动指定绑定
- `config.ssh.username`：默认的用户是`vagrant`，从官方下载的`box`往往使用的是这个用户名。如果是自定制的`box`，所使用的用户名可能会有所不同，通过这个配置设定所用的用户名。
- `config.vm.provision`：我们可以通过这个配置在虚拟机第一次启动的时候进行一些安装配置，后面我们将专门介绍。

需要注意的是，`Vagrantfile` 文件只会在第一次执行 `vagrant up` 时调用执行，其后如果不明确使用 `vagrant reload`，则不会被强制重新加载。

4.2 虚拟机与主机间共享目录
------------------------

我们可以在虚拟机上进行开发了，不过很多人依旧希望使用物理系统上的熟悉的编辑器对文件进行编辑，而非常不喜欢使用 `vim` 在命令行里编辑文件。这没什么，是很正常的现象，也是图形界面演变的动力所在。

那么我们如何把主机上的文件放到虚拟机呢？

嗯，常用的多主机文件共享的办法有 SFTP、SAMBA文件共享、NFS。而对于虚拟机而言，则可以直接使用共享目录的形式，这样最方便而且速度最快。Vagrant 则充分利用了这一点，默认的情况下，会将 `Vagrantfile` 所在目录自动绑定到虚拟机的 `/vagrant/` 目录下。

就拿我们刚刚创建的虚拟机为例，我们本机的 `~/myserver1` 目录将会绑定到 `/vagrant` 下。我们可以很简单的验证一下：

```bash
vagrant@precise64:~$ echo wombat > /vagrant/my.txt
vagrant@precise64:~$ exit
logout
Connection to 127.0.0.1 closed.
tao@wombat:~/myserver1$ ll
总用量 72
drwxr-xr-x    3 tao tao  4096  3月  1 17:08 ./
drwx--x---+ 101 tao tao 32768  3月  1 16:43 ../
-rw-r--r--    1 tao tao     7  3月  1 17:08 my.txt
drwxr-xr-x    3 tao tao  4096  3月  1 15:22 .vagrant/
-rw-r--r--    1 tao tao  4619  3月  1 15:22 Vagrantfile
tao@wombat:~/myserver1$ cat my.txt 
wombat
tao@wombat:~/myserver1$
```

4.3 网络配置
------------

我们可以通过 `config.vm.network` 对虚拟机的网络进行定制。比如，我们可以用下面一行设置端口映射：

```ruby
  config.vm.network "forwarded_port", guest: 80, host: 8080
```

这将让物理机的`8080`端口映射到虚拟机的`80`端口。因此，可以在物理机上直接访问虚拟机所建立的网站。这在虚拟机使用不是`桥接`的时候尤为重要，因为很多时候默认会禁止物理机通过网络直接访问虚拟机，通过端口映射可以依旧能够访问所需端口。

我们还可以通过 `config.vm.network` 来指定所在网络以及IP地址。一般来说，分为`private_network`还是`public_network`。

- `private_network`： 私有网络

位于私有网络的主机之间可以互相访问，但是外界无法访问私有网络的主机，对于某些虚拟机而言，连物理机都无法通过网络直接访问虚拟机，此时如果需要使用物理机访问虚拟机，就需要前面所提及的端口映射。例如：

```ruby
  config.vm.network "private_network", ip: "192.168.50.4"
```

- `public_network`：公有网络

在绝大多数虚拟机上，这等同于`桥接`网络。如果虚拟机选择了桥接方式连接，也就是相当于虚拟机和物理机在网络上处于平等的位置，同样的接在了网卡所在的网络。因此从外界看来，会感觉这是两台独立的计算机。

配置方式和私有网络接近，只需将其中的`private_network`换成`public_network`。如要使用 DHCP：

```ruby
  config.vm.network "public_network"
```

如果物理机存在多块网卡，需要指定某一块作为桥接用，那么可以参考如下配置：

```ruby
config.vm.network "public_network", :bridge => 'en1: Wi-Fi (AirPort)'
```

如果是在启动了虚拟机之后，需要重新加载配置，只需要执行：

```bash
vagrant reload
```


4.4 自定义启动配置
-----------------

如前所例，我们很轻松的用两条命令就创建了一个 Ubuntu 12.04 的服务器，但是这离我们所希望的服务器能够配置成我们所需要的环境还相距甚远。

一种办法是我们自己[定制一个 `box`](http://docs.vagrantup.com/v2/virtualbox/boxes.html)，这样下载下来后，就可以满足我们的需求；而另一种办法，则是可以设定一个配置脚本，让虚拟机在第一次运行的时候就执行这个脚本，然后执行相应的配置工作。我们这里就利用 `config.vm.provision` 这个配置来做这件事情。

假设，我们希望第一次启动的时候做这么几件事情：

- 安装 Apache HTTP 服务器；
- 将Web服务器默认的目录指向 `Vagrantfile` 所在的目录，这样我们可以直接在物理机熟悉的环境编写代码；


我们首先需要先写一个脚本，这个脚本将完成我们所提到的配置工作，将会在第一次启动虚拟机的时候运行。在此我们将其命名为 `bootstrap.sh`，并且放到与 `Vagrantfile` 相同的目录下：

```bash
#!/usr/bin/env bash

apt-get update
apt-get install -y apache2
rm -rf /var/www
ln -fs /vagrant /var/www
```

然后，我们修改 `Vagrantfile` 文件，在配置中加入下面这一行：

```ruby
  config.vm.provision :shell, :path => "bootstrap.sh"
```

然后我们只需要 `vagrant up`（如果虚拟机已经启动，则执行 `vagrant reload --provision`），虚拟机启动之时，就会自动调用该脚本，进行所必需的配置。

进入虚拟机后，可以 `wget http://localhost/Vagrantfile` 来验证这一点。

vagrant 除了使用 `shell` 脚本外，还支持许多常用的管理工具来配置，比如`Puppet`、`Chef`等。进一步的信息，可以在官方网站： http://docs.vagrantup.com/v2/provisioning/index.html 中了解。
