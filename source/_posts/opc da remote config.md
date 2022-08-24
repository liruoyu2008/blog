title: OPC DA局域网远程连接精简配置实践
tags: [opc,工控,DCOM]
categories: [工控]
date: 2022-08-22 14:24:00
---

opc da局域网远程连接的配置进行过许多次，由于DCOM配置本身相对偏僻，前后使用的Windows开发与调试环境也不尽相同，所以每次都是参照教程摸索着走，以至于成不成功全看天意，且很多教程是需要关防火墙的，多少会影响到服务端的安全策略。最近刚好项目需要，遂于午后与吾友`@bool`使用两台机器一一尝试总结，得出一套精简方案，帮助广大工控人以尽量少的设置来完成opc da局域网通讯。

<!--more-->

## 服务端设置

### 1. 添加专用的用户和用户组

- 在`运行`中键入命令`lusrmgr.msc`打开`本地用户和组`
- 在用户中新建一个用户`OPCUser`，输入适当的密码（密码不可为空），并作如下设置：
![图 1](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnywhOyU_zIrDaoV-3/root/content)
- 在用户组中新建一个用户组`OPCGroup`，并将`OPCUser`添加到该组
![图 2](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnywkP_iTvqV3MDApC/root/content)
- **注:如果打算直接使用当前已存在的用户连接OPC DA服务，则该步骤也可以省略，但有个前提：该用户密码不可为空。**

### 2. DCOM配置

- 在`运行`中键入命令`dcomcnfg`打开组件服务，进行如下配置：
- OPCEnum配置
  - 在`组件服务`|`计算机`|`我的电脑`|`DCOM配置`中，找到`OPCEnum`
  - 右键`属性`，并切换到`安全`选项卡
  - 在`启动与激活权限`中，选择`自定义`，并点击`编辑`
  - 添加`OPCGroup`用户组，并赋予全部权限如下：
  ![图 3](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnywoAWawqrevi7f68/root/content)
  - 在访问权限中，也作类似配置
-  OPC DA服务运行时配置（以开普华Kepsware的opc da为例）
  - 在`组件服务`|`计算机`|`我的电脑`|`DCOM配置`中，找到`KEPSErverEX 6.5`
  - 右键`属性`，并切换到`安全`选项卡
  - 在`启动与激活权限`中，选择`自定义`，并点击`编辑`
  - 添加`OPCGroup`用户组，并赋予全部权限如下：
  ![图 4](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnywsNy-rqmLq6KD4U/root/content)
  - 在`访问权限`中，也作类似配置

### 3. 防火墙设置
- 在`运行`中键入命令`firewall.cpl`打开防火墙控制面板，进入`高级设置`
![图 5](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnywyfPGvaNJWg12Aj/root/content)
- 在`入站规则`上右键`新建规则`，新建一个`端口规则`，开放`135`端口访问权限；
- 在`入站规则`上右键`新建规则`，新建一个`程序规则`，开放`C:\Windows\SysWow64\opcenum`访问权限；
- 在`入站规则`上右键`新建规则`，新建一个`程序规则`，开放`C:\Program Files (x86)\Kepware\KEPServerEX 6\server_runtime.exe`OPC DA服务运行时访问权限；
- **注：服务端并不要求登录该OPCUser作为当前用户**

## 客户端设置

### 1.添加专用用户
- 在`运行`中键入命令`lusrmgr.msc`打开`本地用户和组`
- 在用户中新建一个用户`OPCUser`，输入与上面所创建的用户相同的用户名和密码，并作如下设置：
![图 6](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEnyw3rWyW8cV0zOY2c/root/content)

### 2. 访问服务端
- 登录该`OPCUser`用户，即可在该电脑上访问服务端电脑上的OPC DA服务

## 后记

本文参照了Kepware官方的详细配置文档，该文档步骤不可谓不繁琐，但还是具有参考意义的，为方便大家阅读，我已翻译此文档，有兴趣的转我的幕布文章 [远程OPC DA快速上手指南（DCOM）（kepware官方文档直译）](https://www.mubucm.com/doc/4s3Knbv1FTw) 自行阅读吧。
**官方原文档可上kepware官网下载：[点此下载](https://www.kepware.com/getattachment/04042e47-c690-467c-a931-a1ca126575db/Remote-OPC-DA-Quick-Start-Guide-DCOM.pdf)**
