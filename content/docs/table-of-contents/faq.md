---
title: FAQ
weight: 9
---

# FAQ

## 如何登录实验虚拟机
  
在云平台中查看虚拟机访问IP、账户以及密码

{{< tabs "uniqueid" >}}

{{< tab "Windows" >}}

win+R 打开“运行”，然后输入`mstsc`，然后在弹出窗口中输入IP、账号、密码即可
![mstsc](https://cdn.loheagn.com/mstsc.png)

{{< /tab >}}

{{< tab "MacOS（Linux？）" >}}

- [Parallels Client](https://bhpan.buaa.edu.cn:443/link/930A373912ACB5A1FC3CE603C3AF9520)
- Microsoft Remote Desktop （有Bug——或者说是Feature——无法使用macOS本地工具截图）
- VMware Fusion

{{< /tab >}}

{{< /tabs >}}

## 云平台上没有显示虚拟机IP或显示的IP无法登录怎么办

可以用来登录的IP位于10.251.252.1~10.251.255.254之间

- 如果未登录过实验虚拟机
  - 云平台上显示的IP是169或者无IP则暂时无法登录，需要重启虚拟机重新获取IP。需要注意的是，虚拟机的重启、以及DHCP IP的获取都需要时间，开机后应等待3分钟以上，请勿频繁开关机
 
- 如果之前登录过实验虚拟机
  - 云平台上显示的是192开头的IP或者无IP，可以尝试使用之前的可用IP进行登录。如果无法登录（包括密码错误），请联系助教解决。
 
### 重要提醒

由于分配的IP均为DHCP自动获取，有可能实验虚拟机的IP会发生变化，尤其是在长时间关机再开启后。因此请登录实验虚拟机后，及时更改虚拟机密码，以避免以下情况发生：

- 未意识到登录了他人的虚拟机， 帮助别人完成了实验工作
- 自己的虚拟机别别人登录，已经做完的实验被破坏

同时，虚拟机密码请设置非常用敏感密码。因为助教在帮助大家解决问题时往往需要你的密码登录到你的虚拟机。

## 用到的安装包、镜像文件如何获取

- Workstation安装包(`Mware-workstation-full-14.1.3-9474260.exe`)、ESXi镜像(`VMware-VMvisor-Installer-6.5.0-4564106.x86_64-Dell_Customized-A00.iso`)、TinyOS镜像(`Core-10.1.iso`)以及可选使用的CentOS镜像(`CentOS-7-x86_64-Minimal-1908.iso`)均放在了实验虚拟机桌面上，可以直接使用。
- vSphere Client安装包可以在[课程中心下载](http://course.buaa.edu.cn/access/content/group/b7656edd-6e82-4c48-80b3-c9397cf89f72/%E4%BA%91%E5%AE%9E%E9%AA%8C%E8%B5%84%E6%BA%90/VMware-viclient-all-6.0.0.exe)（推荐此方式）或在[临时的资源存放网站](http://10.251.254.150/VMware-viclient-all-6.0.0.exe)下载。

- 同时，所需的资源文件都在[临时的资源存放网站](http://10.251.254.150)存有备份

## Workstation与ESXi在本实验中的用处

Workstation是用来安装ESXi的。安装ESXi成功后，只要保证Workstation正常运转即可，后续的实验**都是通过vSphere (Web) Client在ESXi中完成的**

## 安装workstation时报错的解决方法（关于 .net framework的错误）

![dotneterror](https://cdn.loheagn.com/dotnet_error.png)
打开浏览器登录校园网验证，连接网络之后会自动安装补丁修复此错误。

## 如何在workstation中载入ESXi镜像

- 在workstation中，创建虚拟机时
![workstation_image](https://cdn.loheagn.com/vmware_workstation_run_4.png)

## 载入workstation后界面无反应/鼠标消失怎么办

workstation中的虚拟机是会占用鼠标的，按ctrl+alt即可将鼠标焦点从workstation虚拟机中移出。如果安装程序超过2分钟还没有自动启动，重启workstation中的ESXi虚拟机

## 安装ESXi时的IP如何配置

ESXi的IP不需要特殊配置，使用DHCP分配的动态IP即可。请注意，此IP只在虚拟机中可以有有效访问

## 如何登录vSphere Client

安装[vSpehre Client](http://course.buaa.edu.cn/access/content/group/b7656edd-6e82-4c48-80b3-c9397cf89f72/%E4%BA%91%E5%AE%9E%E9%AA%8C%E8%B5%84%E6%BA%90/VMware-viclient-all-6.0.0.exe)成功后，打开vSphere Client，输入ESXi显示的IP，以及安装ESXi过程中设置的账号密码即可登录。

## 使用web client还是传统的本地client

本次实验大部分操作要求在本地client中完成。如果遇到本地client无法完成的操作，则要登录web client完成

## 访问web client时报错的解决方法

![web_client_error](https://cdn.loheagn.com/web_client_error.png)

按ESC键忽略此错误即可

## 创建用户时提示“用户名或密码格式无效”的解决方法

密码要求包含数字、大小写字母以及符号，缺一不可

## 关于进行虚拟机设置时，使用vSphere Client提示“无法使用vSphere Client编辑版本为12或更高版本的虚拟机的设置”的解决方法

1、创建虚拟机时，第一个页面“配置”不选择“典型”，选择“自定义”，并在“虚拟机版本”处选择虚拟机版本11或以下版本。
2、使用Web客户端登录ESXi管理页面，在虚拟机浏览器中输入ESXi地址即可访问。
以上两种方法使用一种即可。

## 开启虚拟机电源后，打开本地vSphere Client的虚拟机控制台显示黑屏

换用Web客户端访问。

## 安装虚拟机操作系统时显示找不到OS

- 确保连接了正确的镜像文件
- 确保虚拟光驱的状态为“已连接”（可在设置中更改）

一般可以自动检测，无需进入BIOS调整启动顺序或手动选择启动选项，但如果排查过上面的情况仍然无法安装操作系统，可以手动进入BIOS更改一下引导选项。

## 虚拟机无法连接外网

- 确保设置了正确的DNS、已经通过gw认证登录
- 如果无法访问认证页面，可以在控制面板\网络和 Internet\网络连接中查看是否有多余的网卡（不包括VMware Network Adapter），查看这些网卡的IP，将多余的网卡禁用（注意不要禁用错了，要保留IP地址与远程登录地址相同的网卡）

