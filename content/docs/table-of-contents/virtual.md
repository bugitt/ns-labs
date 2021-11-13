---
title: Lab02 虚拟化实验
weight: 2
---

# Lab02 虚拟化实验

## 实验目标

1. 实施计算虚拟化，安装配置环境，熟悉计算虚拟化的概念，理解基本操作，掌握基础知识。
2. 理解集中管理对于虚化的作用，通过部署集中vCenter体验集群的设置，分布式交换机的设置，了解主机从不同网络进行迁移的实际需求。

## 实验内容

1. 按照实验指南的指导，完成实验。
2. 按照实验报告模板，撰写实验报告，将重要的实验步骤截图，填入实验报告中，并回答相应问题。

实验可以在分配的虚拟机中进行，也可以在本地机器中进行。（如果本地机器内存大于12G，则建议在本地进行，以获得更流畅的实验体验）。在本地进行实验的同学，可以前往[这里](https://bhpan.buaa.edu.cn:443/link/59127115D38C8C1C4C1B16DDC16D91F5)下载需要的资源；使用云平台虚拟机的同学，相应的资源已经放到了虚拟机桌面上。

请在云平台作业提交截止时间之前，将作业提交到云平台，命名为：lab02-学号-姓名.pdf，如 lab02-18373722-朱英豪.pdf。

## 前情提要

推荐开始试验前，首先查看[资源文件夹](https://bhpan.buaa.edu.cn:443/link/59127115D38C8C1C4C1B16DDC16D91F5)中的”虚拟化实验.pptx“。

{{< hint danger >}}

有任何问题，请首先阅读 [FAQ](./faq.md)

如果还有问题，还可尝试阅读[官方文档](https://docs.vmware.com/cn/VMware-vSphere/index.html)（如下图所示，请注意选择正确的版本号）

![](https://cdn.loheagn.com/142844.png)

{{< /hint >}}

### Hypervisor

我们知道，一台计算机一般有以下的结构：

<img src="https://cdn.loheagn.com/image-20211107205517736.png" alt="image-20211107205517736" style="zoom:50%;" />

操作系统负责管理硬件资源（CPU，内存，硬盘等），并向上提供相应的系统调用，供具体的应用程序使用。

而我们平常提到的操作系统的虚拟化，本质上就是要模拟出一套硬件（包括虚拟CPU，虚拟内存，虚拟硬盘等），然后在这一套虚拟的硬件的基础上部署客户操作系统。客户操作系统完全不需要做任何修改，即可在这个“虚拟的机器”中顺利执行。但客户操作系统的运行结果（比如接收键盘输入，输出图像和声音等），最终都是要靠原始的“实实在在”的硬件（物理机）来完成的。也就是说，需要有那么一个结构，能够将这个“虚拟的机器”的行为翻译到物理机的行为（比如将虚拟CPU的指令翻译到物理机的CPU指令）。负责做这件事情的结构被称为Hypervisor，又称为虚拟机监控器（virtual machine monitor，缩写为 VMM）。

根据工作方式的不同，Hypervisor分为以下两种。

![image-20211107213246128](https://cdn.loheagn.com/image-20211107213246128.png)

第一种是我们比较熟悉的情况，本质上就是在主操作系统（Host Operating System）上安装了一个虚拟化软件，它来负责充当虚拟机的管理者，并通过主操作系统的系统调用来完成对物理机硬件的使用。VMware Workstation、Virtual Box、Qemu等都属这类虚拟化软件。除了这个虚拟化软件之外，主操作系统上还会运行其他“正常”的应用程序，比如，你在用VMware Workstation的同时还能听歌聊天等。

第二种Hypervisor则直接舍弃了主操作系统（因为毕竟隔着一层，性能会有损失），而是直接把Hypervisor部署在硬件上。在这种情况下，物理机变成了更纯粹的“为虚拟化而生”的机器。Hypervisor能够直接与硬件沟通，其实在某种程度上也承担了主操作系统的角色（管理硬件），因此，我们也可以把这种Hypervisor看作是一种为虚拟化特制的操作系统。这其中典型的就是VMware ESXi。

因为我们不可能要求每位同学都制备一套硬件来安装学习VMware ESXi，所以需要首先使用VMware Workstation来模拟出一套硬件。但VMware Workstation仅仅起一个前置作用，在实际的实验中并不会涉及到。请大家首先理清这层关系。

## 实验指南

### 0. 安装VMware Workstation

使用分配的虚拟机的桌面上的安装包安装即可。

安装完成后需重启机器。

打开VMware Workstation时， 选择试用即可。

![image-20211107120821035](https://cdn.loheagn.com/image-20211107120821035.png)

### 1. 安装VMware ESXi

使用桌面上的ESXi镜像创建虚拟机。

{{< hint info >}}

注意选择客户操作系统的类型。

{{< /hint >}}


![image-20211107121139109](https://cdn.loheagn.com/image-20211107121139109.png)

虚拟机创建完成后，直接打开电源即可启动ESXi操作系统的安装流程，这一过程可能需要等待较长时间。

安装流程中总是保持默认选项即可。

{{< hint info >}}

在此流程中，可能需要使用使用到某些快捷键，这些快捷键可能会首先被你本机的操作系统捕获，在本机的系统设置中暂时屏蔽该快捷键即可。

{{< /hint >}}


{{< hint danger >}}

请务必牢记设置的root密码。

{{< /hint >}}

![image-20211107122136324](https://cdn.loheagn.com/image-20211107122136324.png)

安装完成后，可以看到如下界面。

![image-20211107163915642](https://cdn.loheagn.com/image-20211107163915642.png)

可以看到，ESXi系统获得了一个IPv4地址`192.168.80.128`，并且这个地址是通过`DHCP`的方式获得的。这里用到的DHCP服务器其实是VMware Workstation内置的。也就是说，`192.168.80.128`这个地址只有在安装VMware Workstation的机器上才是有效的。

### 2. 访问ESXi

一般情况下，可以使用桌面客户端VMware vSphere Client作为“Client”和ESXi（作为“Server”）通信。

VMware vSphere Client使用如下图所示的安装包安装即可。

![image-20211107165049138](https://cdn.loheagn.com/image-20211107165049138.png)

安装完成后，打开vSphere Client，填入ESXi的IP地址、root的用户名和密码后即可建立起与ESXi的连接。

{{< hint info >}}

如果弹出安全证书警告，直接忽略即可。

{{< /hint >}}

![image-20211107164923128](https://cdn.loheagn.com/image-20211107164923128.png)

除了桌面客户端，也可以直接使用浏览器访问。访问的地址就是ESXi的地址，用户名和密码与vSphere Client的相同。

![image-20211107165815482](https://cdn.loheagn.com/image-20211107165815482.png)

如下图所示，可以为ESXi分配许可证。

可用的KEY：

```
4F092-DML5N-H8E20-4HC54-17K4D      
1G2J8-4K097-48DU1-HH3G2-C6H60      
4V4JR-40317-H88H1-83ANP-23H76
1G002-DRK5J-08429-K035K-0KK16
NY01K-AP343-M89H8-WJ9EK-2VK1F    
1F29K-4W14M-H8099-PA8X2-9KUHD
```



![image-20211107170758629](https://cdn.loheagn.com/image-20211107170758629.png)

后续实验内容可以在Web端进行，也可以在vSphere Client桌面客户端进行。

### 3. 观察和体验vSphere Client提供的功能

Client侧界面主要包含导航器、主体内容和任务事件这三部分。

![image-20211107172432364](https://cdn.loheagn.com/image-20211107172432364.png)

请浏览左侧导航栏的不同模块和不同模块下不同选项卡的内容，对ESXi提供的功能有个大致的了解。

其中，存储部分可以查看ESXi虚拟机可访问的数据内容。

![image-20211107173326427](https://cdn.loheagn.com/image-20211107173326427.png)

可以通过使用“数据存储浏览器”查看、下载、上传、下载“存储”中的文件。

### 4. 资源池

资源池可以实现对CPU和内存的池化管理。

尝试在vSphere Client中新建资源池。

![2020-11-03-17-11-57](https://cdn.loheagn.com/2020-11-03-17-11-57.png)

理解资源池“预留”和“限制”的含义。

![image-20211107180212183](https://cdn.loheagn.com/image-20211107180212183.png)

### 5. 新建和安装虚拟机

ESXi最主要的功能就是对虚拟机的管理。

vSphere Client和Web端都有显著的入口供用户创建虚拟机。

![image-20211107180409949](https://cdn.loheagn.com/image-20211107180409949.png)

![image-20211107180541776](https://cdn.loheagn.com/image-20211107180541776.png)

创建虚拟机的流程与使用VMware Workstation创建虚拟机没有太大区别，按照创建向导进行即可。

在创建过程的“自定义设置”阶段，需要手动配置CD/DVD驱动器，插入桌面上的CentOS或Tiny Core Linux的安装镜像，以使虚拟机在开机时，能自动进入安装镜像的安装引导界面。

![image-20211107181242690](https://cdn.loheagn.com/image-20211107181242690.png)

当然，你也可以在虚拟机创建完成后，打开电源之前，手动编辑虚拟机配置，添加对应的镜像文件。

![image-20211107181633965](https://cdn.loheagn.com/image-20211107181633965.png)

完成后，打开虚拟机电源，即可进入操作系统的安装引导流程。

### 6. 虚拟机文件系统格式和种类

添加虚拟机后，可以在“存储”中看到每台虚拟机中包含的文件内容。

![image-20211107181915857](https://cdn.loheagn.com/image-20211107181915857.png)

了解这些不同格式的文件和含义和作用。

### 7. 虚拟机导出

关闭虚拟机电源后，即可将虚拟机导出。

![image-20211107182140360](https://cdn.loheagn.com/image-20211107182140360.png)

观察导出文件格式与虚拟机正常文件的区别。

### 8. VMware Tools（选做）

可以在如下图所示的虚拟机控制台的“操作”选项中找到“安装VMware Tools”的选项。

![image-20211107183923365](https://cdn.loheagn.com/image-20211107183923365.png)

点击“安装VMware Tools”后，ESXi会给虚拟机挂载一个包含了安装脚本的光盘。根据安装的操作系统的不同，具体的安装方式也有区别。对于Linux系统，可以在`/dev`目录下找到该光盘，并将其挂载到文件系统中，然后进入其中，执行安装脚本。

### 9. 角色和用户（选做）

可以在Web端创建不同的角色和用户，根据角色的不同，用户将获得不同的权限。尝试使用不同的用户账号登录Client，以体验权限限制带来的差异。

![image-20211107184717658](https://cdn.loheagn.com/image-20211107184717658.png)

### 10. 创建与配置集群（选做）

后面这两部分实验内容比较复杂，实验环境也不是特别理想，有兴趣的同学可以选择尝试。

在上面的实验中，我们进行了ESXi的部署，并使用ESXi创建了一些虚拟机。在实际应用中，往往需要多个ESXi主机组成集群，来提供更多的资源，或者提高可用性。在接下来的实验中，我们将使用vCenter Server管理多个ESXi主机，来管理所有的虚拟机和ESXi“物理机”集群。

1. 在VMware Workstation中新建一个新的ESXi虚拟机。该虚拟机内存可以给大一些，比如14G，磁盘大小也可以大一些，比如100G。以方便后续在其上安装vCenter Server。

    {{< hint danger >}}

可以在VMware Workstation中给机器开大内存和大容量磁盘。

因为内存和磁盘是虚拟的，所以即使本机物理内存和磁盘容量不够，也把握得住。

如果后续实验中，确实出现了内存或磁盘容量不够的情况，请联系助教扩容。

    {{< /hint >}}

2. 安装vCenter Server

{{< tabs "vCenter" >}}

{{< tab "VMware-VIM-all-6.5.0-4602587.iso" >}}

1. 打开vCenter Server安装包[`VMware-VIM-all-6.5.0-4602587.iso`](https://bhpan.buaa.edu.cn:443/link/C7C16536FDE46C8F9210C8CAB35E5848)，双击`autorun`启动安装程序。**之后的安装步骤，如无特殊说明，无需改动默认选项，直接下一步即可**
    ![](https://cdn.loheagn.com/060106.png)
    ![](https://cdn.loheagn.com/install_vcenter.png)

2. 点击[安装]按钮，（等待几十秒），会弹出另一个安装页面。选择默认选项“vCenter Server 和嵌入式Platform Services Controller”。

3. 在`系统网络名称`页面，要填入虚拟机的本机IP。点击确认后会弹出两次提示，确认并无视即可。**本实验中应该是10.251开头的IP**
    ![](https://cdn.loheagn.com/fqdn.png)

4. 在`vCenter Single Sing-On 配置`页面，需要设置登录vCenter的密码。域名建议不改动，密码需要符合下面的要求，站点名称可以随意更改。
    ![](https://cdn.loheagn.com/vcenter_pwd.png)

5. vCenter Server所需的端口。如果点击下一步时提示某个端口被占用，在cmd中执行`netstat -ano | findstr "{port_num}"`即可找到占用端口的进程（把`{port_num}`替换为相应的端口号，注意两侧有引号），找到对应的进程号后，使用`tskill ${pid}`kill对应的进程即可。其中，80端口的进程号为`4`的服务可以使用`NET stop HTTP`杀掉。**不推荐**更改默认端口号。
    ![](https://cdn.loheagn.com/vcenter_ports.png)

6. 安装完成后，浏览器访问虚拟机地址或者使用vSphere Client均可访问到vCenter。账号为`administrator@vsphere.local`，密码为第4步中设置的密码。

7. 创建数据中心。

8. 添加主机。把本次实验创建的两个ESXi主机都添加进去。在添加主机时会弹出安全警示，选择“是”继续添加即可。
    ![](https://cdn.loheagn.com/vcenter_addhosts.png)

{{< /tab >}}

{{< tab "VMware-VCSA-all-6.5.0-4602587.iso" >}}

1. 安装打开vCenter Server安装包[`VMware-VCSA-all-6.5.0-4602587.iso`](https://bhpan.buaa.edu.cn:443/link/73FCD6D1FB5716CD0FA5A93B338C1C4A)，在本机（或虚拟机）中装载该ISO文件，可以看到README文件，其中包含着具体的安装指引。**之后的安装步骤，如无特殊说明，无需改动默认选项，直接下一步即可**
    ![](https://cdn.loheagn.com/125244.png)

2. 我们刚刚启动的只是一个安装器程序，vCenter Server本质上还是要安装在一个特定的ESXi主机中。所以，在选择安装目标时，需要填入刚才新建的那个内存和磁盘容量都比较大的ESXi虚拟机的相关信息。
    ![](https://cdn.loheagn.com/131933.png)

3. 在最后的网络配置这里，选择`DHCP`
    ![](https://cdn.loheagn.com/132706.png)

4. 安装完成后，浏览器访问虚拟机地址或者使用vSphere Client均可访问到vCenter。

5. 创建数据中心。

6. 添加主机。把本次实验创建的两个ESXi主机都添加进去。在添加主机时会弹出安全警示，选择“是”继续添加即可。
    ![](https://cdn.loheagn.com/vcenter_addhosts.png)

{{< /tab >}}

{{< /tabs >}}

### 11. 分布式交换机（选做）

默认情况下，两台主机中的虚拟机之间的网络是不通的。这很显然不符合我们的需求——搭建集群的主要目的就是为了整合不同主机的资源，虚拟机对其在哪个主机上应该是无感的。

因此，就需要使用分布式交换机，将两个或多个主机的网络连通起来。

感兴趣的同学，可以查找相关资料，在配置好的集群上，创建分布式虚拟交换机。并分别在不同的主机上部署虚拟机，实现它们之间的网络互通。


## 实验报告模板

```markdown
# Lab02 虚拟化实验

> 班级：
> 学号：
> 姓名：

---

## 实验内容

### 1. 安装VMware ESXi



### 2. 访问ESXi



#### ESXi分配许可证



### 3. 观察和体验vSphere Client提供的功能



### 4. 资源池



#### “预留”和“限制”




### 5. 新建和安装虚拟机



### 6. 虚拟机文件系统格式和种类



#### 不同格式的文件和含义和作用



### 7. 虚拟机导出



#### 导出文件格式与虚拟机正常文件的区别



### 8. VMware Tools（选做）



### 9. 角色和用户（选做）



### 10. 创建与配置集群（选做）



### 11. 分布式虚拟交换机（选做）



## 实验总结与心得



```
