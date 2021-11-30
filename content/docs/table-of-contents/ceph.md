---
title: Lab03 Ceph存储集群实践
weight: 3
---

# Lab03 Ceph存储集群实践

## 实验目的

1. 了解Ceph存储的基本工作原理

2. 建立对分布式存储的初步认识

## 实验说明

本次实验需要使用至少三台虚拟机，而每位同学只有一台机器。因此可以三名同学选择合作实验。这三名同学的实验报告内容（除“实验总结与心得”外）可以全部相同。请在实验报告中写明提交者是谁，另外的合作者是谁。

当然，你也可以选择自己完成实验。但这要求你在本地的虚拟机管理工具中，新建3台以上的机器进行实验（或者使用其他公有云产品中的机器）。

对于使用自己的机器的同学，建议采用CentOS系统（这样Kernel版本不至于太新，减少出错的可能），每台机器配置为2核2G即可，并且每台机器至少附带一个全新的大于5G的虚拟磁盘。

请在云平台作业提交截止时间之前，将作业提交到云平台，命名为：lab03-学号-姓名.pdf，如 lab03-18373722-朱英豪.pdf。

{{< hint info >}}

Ceph是一个分布式的存储集群。大多数生产可用的分布式集群的部署安装都不简单，Ceph也不例外。很多时候，需要对Linux Kernel和Linux网络管理有基本的了解。

**实验中遇到的困难请及时在课程微信群中抛出。**

实验文档不可能将整个Ceph的文档都搬过来，因此，在实验的整个过程中，请务必将[Ceph官方文档](https://docs.ceph.com/en/pacific/)也作为重要的参考依据。实验文档中有描述不到位的地方，可以参考[Ceph官方文档](https://docs.ceph.com/en/pacific/)的相关讨论。

另外，因为Ceph是Red Hat主导开发的产品，因此，Red Hat也有关于Ceph完整的技术介绍，[它的文档](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/5)同样值得参考，比如，你在这里可以找到它的[Ceph部署指南](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/5/html/installation_guide/index)，甚至在很多方面比Ceph官方更加详实。

{{< /hint >}}

{{< hint danger >}}

在中文互联网上，你很容易找到[Ceph的中文文档](http://docs.ceph.org.cn/start/intro/)。但它有很多**过时的内容**，比如它还在使用`ceph-deploy`这个官方已经不再维护的工具（这个工具同样不支持新版的Ceph）来部署集群。因此，它的内容充其量也只能作为部分参考。

除了中文文档，你还会找到很多良莠不齐的Blog，这些文章很多也是过时的，在使用时同样务必加以甄别。

在实验过程中，**在执行每一条命令之前请务必搞清楚它是用来干啥的，执行后会有什么结果**。当然，勇敢试错是值得鼓励的，但请做好从头开始的准备。

{{< /hint >}}

## 概述

Ceph(读音 /ˈsɛf/) 是一个分布式的存储集群。什么是分布式存储？我们为什么需要它？

试想，你在搭建了一个网站对外提供服务。用户在使用网站的过程中会存储大量的数据，网站运行过程中也会产生大量的日志信息。

最初，你将网站部署在一个装有500G硬盘的服务器上。随着时间的流逝，500G的硬盘逐渐被填满。现在你有两种选择。

1. 纵向拓展。在服务器上加装硬盘，甚至你可以使用LVM将硬盘无缝拓展到原来的文件系统中，上层应用和用户根本看不出来有任何差别。但随着数据量的进一步积累，加装的硬盘还会被填满。即使你将服务器的硬盘槽位都插满，最终还是无法解决数据量逐渐增大的问题。数据是无限的，一台机器能承受的数据量总是有限的，氪金也无法解决这个问题。

2. 横向拓展。买一台新的服务器，用网线把它和原来的服务器连起来，把原来的服务器存不下的数据存储到这台新的服务器上。当需要使用到这些数据时，再从新的服务器上取出来。当第二台服务器被填满后，再添加新的服务器。

第二种看起来是最可行的方法：随着业务的扩展，继续加机器就可以了。这种由多台网络互通的机器组成的存储系统即可被理解为“分布式存储系统”。

但随着机器数量的增加，整个系统的复杂度也在上升。新的多机器系统会表现出与原来的单机系统很多不同的特性，会带来更多的问题，比如：

- 如何划分数据？也就是说，如何决定网站接收的某份数据该存储到哪台机器上？每台机器的存储容量可能不同，存储性能也可能不同，如何平衡每台机器的存储容量？

- 如何获取数据？我们将数据保存在不同的机器上时，通常保存的不是一个完整的文件，而是经过一个个切分后的数据块，每个数据块可能保存在不同的机器上。当获取数据时，我们需要知道要获取的文件包含哪些数据块，每个数据块存放在哪台机器的哪个位置。随着机器数量和数据量的增加，这不是一个简单的任务。

- 随着机器数量的增加，系统发生故障的概率也在增加。仅对硬盘而言，我们假设每块硬盘在一年中发生故障的概率是1%，对于普通消费者而言，这似乎不是什么问题，这种故障可能在硬盘的整个使用周期内都不会发生；但对于一个包含几百块硬盘的存储系统来说，这意味着几乎每天都会有若干块硬盘发生故障，而每块硬盘的故障都有可能造成系统的宕机和数据损失。因此，分布式存储系统必须有较强的容错能力，能够在一定数量的机器崩溃时，仍能对外提供服务。

- ……

上面这些问题，正是Ceph这类分布式存储系统所要解决的问题。简单来说，Ceph是一个能将大量廉价的存储设备统一组织起来，并对外提供统一的服务接口的，提供**分布式**、**横向拓展**、**高度可靠性**的存储系统。

{{< hint info >}}

对分布式系统感兴趣的同学，可以趁下学期或大四空闲的时候听一下[MIT 6.824](https://pdos.csail.mit.edu/6.824/)的课程，并尽量完成它的全部实验。

在互联网上搜索“MIT 6.824”能得到大量的资料，比如，B站上有[翻译好的熟肉](https://www.bilibili.com/video/BV1R7411t71W)。

{{< /hint >}}

除此之外，Ceph的独特之处还在于，它在一个存储系统上，对外提供了三种类型的访问接口：

- 文件存储。简单来说，你可以将Ceph的存储池抽象为一个文件系统，并挂载到某个目录上，然后像读写本地文件一样，在这个新的目录上创建、读写、删除文件。并且该文件系统可以同时被多台机器同时挂载，并被同时读写。从而实现多台机器间的存储共享。

- 对象存储。Ceph提供了对象存储网关，并同时提供了S3和Swift风格的API接口。你可以使用这些接口上传和下载文件。

- 块存储。Ceph还能提供块存储的抽象。即客户端（集群外的机器）通过块存储接口访问的“所有数据按照固定的大小分块，每一块赋予一个用于寻址的编号。”客户端可以像使用硬盘这种块设备一样，使用这些块存储的接口进行数据的读写。（一般这种块设备的读写都是由操作系统代劳的。操作系统会对块设备进行分区等操作，并在其上部署文件系统，应用程序和用户看到直接看到的是文件系统的接口（也就是文件存储））。

![](https://cdn.loheagn.com/123326.jpg)

需要注意的是，虽然Ceph对外提供了上面这三种不同类型的存储接口，但其底层会使用相同的逻辑对接收的数据进行分块和存储。

## 重要概念

一个Ceph集群必须包含三种类型的进程：Monitor、OSD和Manager。其中，Monitor和OSD是最核心的两类进程。

#### Monitor 和 OSD

Monitor进程负责维护整个系统的状态信息，这些状态信息包括当前的Ceph集群的拓扑结构等，这些信息对Ceph集群中各个进程的通信来说非常关键。除此之外，Monitor进程还负责充当外界（也就是官方文档中总是提到的“Client”）与OSD进程交流的媒介。

OSD进程则负责进行真正的数据存储。如下图所示，外界传送给Ceph集群的数据（不管是通过文件存储、对象存储还是块存储的接口）都将被转化为一个个对象（object）。这些object将经由OSD进程存储到磁盘中。

![](https://cdn.loheagn.com/134513.jpg)

简单来说，当一个Client试图向Ceph集群读写数据时，将发生以下步骤：

1. Client向Monitor进程请求一个token校验信息

2. Monitor生成token校验信息，并将其返回给Client

3. Monitor同时会将token校验信息同步OSD进程

4. Client携带着Monitor返回的token校验信息向对应的token发送数据读写请求

5. OSD进程将数据存储到合适的位置，或从合适的位置读出数据

6. OSD进程向Client返回数据

{{< hint info >}}

以上读写数据的流程是经过极致简化的，主要是为了帮助大家建立对Monitor进程和OSD进程所起的作用的感性认识。

想要了解详情，请阅读[Ceph的文档 - Architecture](https://docs.ceph.com/en/pacific/architecture/)。

{{< /hint >}}

#### Manager

Manager进程主要负责跟踪当前集群的运行时状况，包括当前集群的存储利用率、存储性能等等。同时，它还负责提供Ceph Dashboard、RESTful接口的外部服务。

{{< hint info >}}

需要注意的是，Ceph集群中有**三类**这样的进程，但不是每个进程只有**一个**。

我们之前提到过，Ceph是一个有很高容错性的分布式系统，而达到高容错性的一个很重要的方式就是“**冗余**”。

比如，对于Monitor进程来讲，集群中仅有一个就够用了。但如果运行这一个Monitor进程的机器挂了，那么整个集群就会瘫痪（Client将不知道该跟谁通信来拿到校验信息和集群状态信息等）。因此，一个高可用的Ceph集群中会包含多个执行几乎相同任务的运行在不同机器上的Monitor进程；这样挂了一个，Client还可以跟剩下的通信，整个集群依旧可以正常对外提供服务。同样的道理，OSD进程和Manager进程也有多个副本。

另外，Ceph为了保证数据的可靠性（也就是说Client存储进来的数据不能丢失）——注意区分其与整个系统可靠性的区别——在默认情况下，会将每份数据存储**3份**，每份都会存储在不同的OSD上（鸡蛋不能放到同一个篮子里）。这样，即使有部分OSD挂掉，也能保证大部分数据不会丢失。因此，一个健康的Ceph集群要求至少同时存在三个健康的OSD进程（当然，这个默认的数值可以更改）。

{{< /hint >}}

#### Pool 与 Placement Group（PG）

请查阅Ceph的相关文档，阐述 Pool、Placement Group 与 OSD 之间的关系。

## Ceph部署

本节内容的目标是创建一个可用的Ceph集群。其中包括，至少一个Monitor进程、至少一个Manager进程、至少三个OSD进程。

{{< hint info >}}

使用云平台提供的机器进行实验的同学，请直接使用root用户登录，密码是`&shieshuyuan21`。

登录后，请首先使用`passwd`命令修改密码。

{{< /hint >}}

Ceph官方提供了[多种部署方式](https://docs.ceph.com/en/pacific/install/)。

在本实验文档中，我们采用[Cephadm](https://docs.ceph.com/en/pacific/cephadm/#cephadm)作为部署工具。Cephadm也是官方推荐的部署和管理Ceph集群的工具，它不仅可以用来部署Ceph，还可以在安装完成后，用来管理集群（添加和移除节点、开启rgw等）。

当然，如果你对其他部署方式感兴趣的话，也可以尝试；比如，对Kubernetes比较感兴趣的同学，可以尝试使用[Rook](https://rook.io/)进行部署。

### Cephadm

Cephadm是基于“容器技术（Container）”进行工作的，每个Ceph的工作进程都运行在相互隔离的容器中。Cephadm支持使用Docker和Podman作为容器运行时。在部署时，Cephadm将首先检查本机中安装的容器运行时类型，当Docker与Podman并存时，将首先使用Podman（毕竟Podman也是Red Hat的产品）。

在部署时，Cephadm会首先在本地启动一个mini的Ceph集群，其中包括Ceph集群最基本的Monitor进程和Manager进程（当然，这两个进程都是通过容器形式运行起来的）。这个mini集群在某种程度上来说也是合法的，只不过其基本不能对外提供任何功能。随后，我们将继续使用Cephadm提供的工具，将其他机器（后文中也会称之为“节点”）加入到集群中（也就是在其他节点中启动Ceph的Monitor、Manager、OSD等进程），从而构建一个完整可用的集群。

下面是Red Hat给出使用Cephadm构建的集群的架构图。

![](https://cdn.loheagn.com/090719.jpg)

图中的“Container”指的就是我们上面所说的“容器”。

图中最左边的Bootstrap Host就是我们执行Cephadm相关命令的机器（事实上，在整个集群的构建过程中，除了修改IP和联网等操作外，我们都只会在这个Bootstrap Host上进行操作）。我们在Bootstrap Host执行的针对其他节点的操作，都是Cephadm通过ssh的方式发送给对应节点的。

这个图目前大家心里有个印象就好，在后续的实验操作中，可以反复回来对照检查。

### 部署前操作

#### 联网

部署过程中的操作尽量都联网进行。云平台的Linux机器联网操作请参考：

> 推荐使用[Dr-Bluemond/srun](https://github.com/Dr-Bluemond/srun)提供的工具。云平台提供已经编译好的Linux64位版本。可以这样获取：
> 
> ```bash
> wget https://scs.buaa.edu.cn/scsos/tools/linux/buaalogin
> chmod +x ./buaalogin
> ```
> 
> 使用前请使用`config`命令配置一下校园网用户名和密码（注意，如果用户名中有英文的话，请大小写都尝试一下）：
> 
> ```bash
> ./buaalogin config
> ```
> 
> 配置完成后，使用`login`命令登录即可：
> 
> ```bash
> ./buaalogin login
> ```
> 
> 或，直接：
> 
> ```bash
> ./buaalogin
> ```
> 
> 如果想作为系统命令使用的话（注意替换合适的安装路径）：
> 
> ```bash
> sudo install ./buaalogin /usr/local/bin
> ```

#### 安装Cephadm

{{< hint info >}}

对于云平台提供的虚拟机，Cephadm相关的工具已经安装好，本步骤可以略过。

{{< /hint >}}

{{< hint info >}}

本步骤需要在各台机器上都要完成

{{< /hint >}}

Cephadm也是个命令行工具，在使用前也需要额外安装这个工具。

1. 保证系统已经安装好curl、Python3、Podman/Docker(推荐使用Podman，都是自家RedHat的产品)

2. 下载Cephadm：

    ```bash
    curl --silent --remote-name --location https://scs.buaa.edu.cn/scsos/ceph/cephadm
    ```

    赋予可执行权限；

    ```bash
    chmod +x ./cephadm
    ```

3. 安装相关工具（请务必保持联网状态，可能会耗费比较久的时间，请耐心等待）：

    ```bash
    ./cephadm add-repo --release octopus
    ./cephadm install
    cephadm install ceph-common
    ```

#### 配置静态IP，修改Hostname

{{< hint info >}}

本步骤需要在各台机器上都要完成

{{< /hint >}}

Ceph集群的各个机器之间是通过IP地址进行通信的，而云平台的机器在重启后通过DHCP获取的IP可能会发生变化，这将给实验带来一定的干扰。

因此，建议在正式开始实验前，为机器配置静态IP（即，将你此时的虚拟机的IP固定下来，即禁用DHCP，改为手动指定IP、网关、DNS服务器等）。对于使用个人机器进行实验的同学，同样也建议配置一下静态IP。

{{< hint info >}}

使用云平台提供的机器的同学，请参考下方给出的CentOS机器的配置方法。

{{< /hint >}}

{{< tabs "Static IP" >}}

{{< tab "CentOS" >}}

对于CentOS的机器，配置IP可以手动编辑`/etc/sysconfig/network-scripts/`下的配置文件。

也可以采用NetworkManager提供的下面这种图形化的终端配置方法。

登录到虚拟机后，使用`nmtui`编辑需要配置静态IP的网卡：

```bash
nmtui edit ens150
```
{{< hint danger >}}

# 指定正确的网卡

上面这条命令中的`ens150`指的是你要指定静态IP的网卡名称。请根据自己的实际情况指定网卡名称。

如何查看自己的网卡名称？

在终端中输入`ip a`，将得到大致如下的输出：

![](https://cdn.loheagn.com/150722.png)

可以看到，这里列出了当前机器上的所有网卡及其上的IP等信息，并且每个都有编号。

找到那个标有你的IP地址的网卡信息，即可找到它的网卡网卡名称。

比如，在上图中，你要固定的IP是`10.251.253.10`，找到它对应的那一组网卡信息，即可看到网卡名称是`ens160`。

{{< /hint >}}

在打开的如下图所示的界面中，编辑`IPv4 CONFIGURATION`下面的几项。

![](https://cdn.loheagn.com/064525.png)

首先，将`Automatic`切换为`Manual`。意思是，不希望通过DHCP自动获取IP，而是手动指定IP。

然后，编辑`Address`，填入你当前的IP地址，并给出子网掩码的位数。对于云平台的机器子网掩码位数是`22`；使用本地机器的同学，请根据实际情况配置。

然后，编辑`Gateway`，即网关地址。对于云平台的机器，这一地址是`10.251.252.1`；使用本地机器的同学，请根据实际情况配置。

然后，编辑`DNS servers`，即DNS服务器地址。对于校园网内的机器，都可以将DNS地址指定为`202.112.128.50`。

之后，选择最下面的`OK`退出即可。

总而言之，如果使用云平台的机器，最终配置的结果与下图所示应该一直（仅IP地址不同）。

![](https://cdn.loheagn.com/150020.png)

{{< /tab >}}

{{< tab "Debian 及 Ubuntu 16.04" >}}

直接编辑 `/etc/network/interfaces` 文件即可。具体可以参考[这篇文章](https://michael.mckinnon.id.au/2016/05/05/configuring-ubuntu-16-04-static-ip-address/)。

其中，对于云平台的机器，需要指定`gateway`为`10.251.253.10`，`netmask`为`255.255.252.0`，`dns-nameservers`为`202.112.128.50`。

{{< /tab >}}

{{< tab "Ubuntu 18.04 及以上" >}}

Ubuntu 18.04开始，Ubuntu开始使用Netplan来管理网络连接。因此，配置静态IP需要编辑 `/etc/netplan/` 下的配置文件。具体可以参考[这篇文章](https://linuxize.com/post/how-to-configure-static-ip-address-on-ubuntu-18-04/)。

其中，对于云平台的机器，需要指定`gateway`为`10.251.253.10`，`netmask`为`255.255.252.0`，`nameservers`为`202.112.128.50`。

{{< /tab >}}

{{< /tabs >}}

{{< hint danger >}}

对于使用云平台的机器的同学，配置静态IP时，请**务必务必**保证使用现有的IP。

也就是说，配置静态IP前后，你的机器的IP应该始终相同。

{{< /hint >}}

配置完静态IP后，还需要修改一下机器的hostname，以区分集群内的各台机器。

```bash
echo '你的机器的Hostname' > /etc/hostname
```

机器的hostname其实可以任意指定，但最好有意义，并携带上你的学号，以防止云平台上几十台实验虚拟机的hostname发生冲突。

比如，如果你的学号是`15213304`，可以将第一台机器的hostname指定为`ceph-01-15213304`，第二台机器的hostname指定为`ceph-02-15213304`。

配置完静态IP和hostname后，请使用 `reboot` 命令重启机器。

### 部署

{{< hint info >}}

基于Cephadm的便捷性，部署部分的操作只需要在**选定的一台**机器（我们把这台机器称为Bootstrap Host上执行即可。

{{< /hint >}}

{{< hint info >}}

在进行部署操作前，请务必保证所有机器处于联网状态。

{{< /hint >}}

#### BootStrap

我们首先需要在一台选定的机器上，使用`cephadm`启动一个mini集群。

```bash
cephadm bootstrap --image harbor.scs.buaa.edu.cn/ceph/ceph:v16 --mon-ip *<mon-ip>* --log-to-file
```

请将`*<mon-ip>*`替换为你执行这命令的机器的IP。

如：

```bash
cephadm --image harbor.scs.buaa.edu.cn/ceph/ceph:v16 bootstrap --mon-ip 10.251.219.248
```

上面这条命令中，`--image`制定了Cephadm启动容器时使用的镜像名称，`--mon-ip`指定了Cephadm要在哪个机器上启动一个mini集群。

更详细地，这条命令将会做如下事情：

> - Create a monitor and manager daemon for the new cluster on the local host.
>
> - Generate a new SSH key for the Ceph cluster and add it to the root user's /root/.ssh/authorized_keys file.
> 
> - Write a copy of the public key to /etc/ceph/ceph.pub.
> 
> - Write a minimal configuration file to /etc/ceph/ceph.conf. This file is needed to communicate with the new cluster.
> 
> - Write a copy of the client.admin administrative (privileged!) secret key to /etc/ceph/ceph.client.admin.keyring.
> 
> - Add the _admin label to the bootstrap host. By default, any host with this label will (also) get a copy of /etc/ceph/ceph.conf and /etc/ceph/ceph.client.admin.keyring.

命令执行完成后，我们可以通过`ceph -s`查看当前集群的状态。

![](https://cdn.loheagn.com/074508.png)

可以看到，确实启动了一个Monitor进程和一个Manager进程。

另外，我们还注意到，当前集群的健康状态是`HEALTH_WARN`，原因下面也列出来了：`OSD count 0 < osd_pool_default_size 3`。这是因为当前Ceph集群默认的每个Pool的副本数应该是3（即，Ceph中存储的每份数据必须复制3份，放在3个不同的OSD中），但我们OSD的进程数是0。不用着急，马上我们就会创建足够的OSD进程。

![](https://cdn.loheagn.com/074750.png)

mini集群启动完成后，可以使用`podman ps`查看当前启动的容器的状态：

![](https://cdn.loheagn.com/091703.png)

可以看到，最上面那两个容器，对应的就是ceph集群的Monitor进程和Manager进程。

#### Ceph Dashboard（可选）

注意看`bootstrap`指令的输出，你可以看到一段这样的内容：

![](https://cdn.loheagn.com/074911.png)

显然，这是在告诉我们，cephadm同样启动了一个`Ceph Dashboard`，这是一个Ceph的管理前端。通过访问这个页面，我们就可以以可视化的方式观察到当前集群的状态。

这段信息中的`ceph-01`代指的就是你当前这台机器的IP（当然，如果你的Hostname不是`ceph-01`的话，得到的将是其他的结果）。用你的机器的IP替换这个`ceph-01`，尝试在浏览器访问它。

是不是访问失败了？这是因为Ceph Dashboard默认使用的是HTTPS协议，而且它用的HTTPS证书还是自己签发的。。。为了能正常访问，我们可以手动禁用SSL：

```bash
ceph config set mgr mgr/dashboard/ssl false
```

但禁用SSL后，Dashboard服务将默认监听8080端口。但在CentOS中，8080端口默认是被防火墙屏蔽的。

你可以选择手动打开防火墙的8080端口；也可以像下面这样，将Dashboard服务的监听端口手动改为8443（因为这个端口就是使用HTTPS时Dashboard的监听端口，在刚才的Bootstrap时已经在防火墙中打开了）：

```bash
ceph config set mgr mgr/dashboard/server_port 8443
```

然后，重启该Dashboard服务，使刚才的配置生效。

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

然后，查看当前的服务状态（如果输出为空的话，耐心多等一会儿）：

```bash
ceph mgr services
```

![](https://cdn.loheagn.com/080831.png)

现在，你应该可以正常访问Dashboard服务了。注意，用户名和密码是我们前面提到的Bootstrap命令输出的那堆信息中提到的。

![](https://cdn.loheagn.com/081143.png)

#### 添加其他节点

{{< hint danger >}}

添加其他节点前，请保证所有节点都处于联网状态

{{< /hint >}}

接下来，我们将把其他机器添加到现有的这个mini集群中来。

前面提到过，cephadm是通过ssh协议与其他机器通信的。所以，这里需要首先把Bootstrap Host机器的公钥copy到其他的所有机器：

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>*
```

例如，如果你的有一台机器的IP是`10.252.252.40`，那么这条命令应该是：

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.252.252.40
```

处理完所有的机器后，就可以正式将它们加入到mini集群中来了：

```bash
ceph orch host add *<newhost>* [*<ip>*] [*<label1> ...*]
```

例如，你有一台机器的Hostname是`ceph-02`，IP是`10.252.252.40`，那么这条命令应该是：

```bash
ceph orch host add ceph-02 10.252.252.40
```

添加完成后，你可以通过`ceph -s`查看当前集群状态的变化。也可以通过Ceph Dashboard看到变化。

#### 创建OSD进程

我们知道，OSD进程是真正用来做数据读写的进程。我们可以用一块专门的磁盘交给OSD进程来读写数据，ceph集群所存储的数据就将保存在这些磁盘中。

这些被用来交给OSD进程管理的磁盘，应该满足以下条件：

> - The device must have no partitions.
> 
> - The device must not have any LVM state.
> 
> - The device must not be mounted.
> 
> - The device must not contain a file system.
> 
> - The device must not contain a Ceph BlueStore OSD.
> 
> - The device must be larger than 5 GB.

简单来说，就是将一块干净的磁盘插入机器后，什么都不用做就好。

在云平台分配的虚拟机中，每台机器都额外插入了一块这样干净的磁盘。可以通过`fdisk -l`来查看：

![](https://cdn.loheagn.com/090352.png)

注意看上面这两块磁盘：

- 第一个名称是`/dev/sda`，容量是16G，有两个分区：`/dev/sda1`，`/dev/sda2`。这就是我们现在在使用的这个系统所用的磁盘，系统数据都存储在这个磁盘中。

- 第二个名称是`/dev/sdb`，容量是10G，没有任何分区。这就是我们即将交给OSD管理的磁盘。

使用下面的命令来创建OSD进程：

```bash
ceph orch daemon add osd *<hostname>*:*<device-name>*
```

比如，你要在主机`ceph-01`的名称为`/dev/sdb`的磁盘上创建OSD进程，那么命令应该是：


```bash
ceph orch daemon add osd ceph-01:/dev/sdb
```

{{< hint info >}}

这条命令默认不会有输出创建OSD进程的详细信息，也就是说，如果该命令很耗时的话，那么你将在什么输出都没有的情况下等待较长时间，这可能令人发慌。你可以加上`--verbose`参数（缩写为`-v`），来让它输出详细信息。

比如：

```bash
ceph orch daemon add osd ceph-01:/dev/sdb --verbose
```

{{< /hint >}}

执行完成后，可以使用`ceph -s`查看当前的集群状态，可以发现已经有一个OSD进程加入了进来。

![](https://cdn.loheagn.com/105544.png)

使用同样的方法，将所有节点的附加硬盘都加入进来。

## 实验报告模板


```markdown
# Lab03 Ceph存储实践

> 班级：
> 学号：
> 姓名：
> 合作者们：['学号-姓名', '学号-姓名']

---

## 实验内容

### 基本概念思考

回答下列问题：

#### Pool、Placement Group 与 OSD 之间的关系

### Ceph部署

<!-- 本部分内容可以根据部署方式的不同进行不同的改动，这里是使用Cephadm部署的模板 -->

#### 实验前置准备

<!-- 可写内容包括： -->
<!-- 云平台/本机，操作系统的具体版本，各台机器的Hostname -->
<!-- 一些前置工具的安装、配静态IP、改Hostname的记录 -->

#### Bootstrap

#### 添加 Host

#### 创建 OSD 

## 实验总结与心得

```
