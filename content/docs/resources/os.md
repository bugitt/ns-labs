---
title: 操作系统
weight: 1
---

# 操作系统

常见操作系统 ISO 镜像可至官网、[MSDN，我告诉你](https://msdn.itellyou.cn/)、[Next, ITELLYOU](https://next.itellyou.cn/) 等网站上去下载。

## Windows Server 2019

ED2K 下载：`ed2k://|file|cn_windows_server_2019_updated_july_2020_x64_dvd_2c9b67da.iso|5675251712|19AE348F0D321785007D95B7D2FAF320|/`

使用 KMS 激活方式如下：

```sh
# 使用管理员身份打开PowerShell
DISM /online /Set-Edition:ServerDatacenter /ProductKey:WMDGN-G9PQG-XVVXX-R3X43-63DFG /AcceptEula
# 打开 CMD (管理员)
slmgr.vbs /ipk WMDGN-G9PQG-XVVXX-R3X43-63DFG
slmgr.vbs /skms kms.teevee.asia
slmgr.vbs /ato
```
