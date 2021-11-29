---
title: Lab03 Ceph存储集群实践
weight: 3
---

# Lab03 Ceph存储集群实践

{{< hint info >}}

Ceph是一个分布式的存储集群。大多数生产可用的分布式集群的部署安装都不简单，Ceph也不例外。很多时候，需要对Linux Kernel和Linux网络管理有基本的了解。

实验文档不可能将整个Ceph的文档都搬过来，因此，在实验的整个过程中，请务必将[Ceph官方文档](https://docs.ceph.com/en/pacific/)也作为重要的参考依据。实验文档中有描述不到位的地方，可以参考[Ceph官方文档](https://docs.ceph.com/en/pacific/)的相关讨论。

另外，因为Ceph是Red Hat主导开发的产品，因此，Red Hat也有关于Ceph完整的技术介绍，[它的文档](https://access.Red Hat.com/documentation/en-us/red_hat_ceph_storage/5)同样值得参考，比如，你在这里可以找到它的[Ceph部署指南](https://access.Red Hat.com/documentation/en-us/red_hat_ceph_storage/5/html/installation_guide/index)，甚至在很多方面比Ceph官方更加详实。

{{< /hint >}}

{{< hint danger >}}

在中文互联网上，你很容易找到[Ceph的中文文档](http://docs.ceph.org.cn/start/intro/)。但它有很多**过时的内容**，比如它还在使用`ceph-deploy`这个官方已经不再维护的工具（这个工具同样不支持新版的Ceph）来部署集群。因此，它的内容充其量也只能作为部分参考。

除了中文文档，你还会找到很多良莠不齐的Blog，这些文章很多也是过时的，在使用时同样务必加以甄别。

在实验过程中，**在执行每一条命令之前请务必搞清楚它是用来干啥的，执行后会有什么结果**。当然，勇敢试错是值得鼓励的，但请做好从头开始的准备。

{{< /hint >}}

## 概述

## 重要概念

## Ceph部署

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

1. 保证系统已经安装好Python3

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
cephadm bootstrap --image harbor.scs.buaa.edu.cn/ceph/ceph:v16 --mon-ip 10.251.253.10 --log-to-file
```

上面这条命令中，`--image`制定了Cephadm启动容器时使用的镜像名称，`--mon-ip`指定了Cephadm要在哪个机器上启动一个mini集群，`--log-to-file`指定Cephadm将log输出到文件中，这样可以方便我们在日后排查问题。

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

#### 添加其他节点

#### 创建OSD进程