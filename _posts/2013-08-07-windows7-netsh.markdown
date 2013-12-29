---
title: Windows 7系统中使用netsh建立无线局域网
layout: post
guid: urn:uuid:d21c2a10-01d3-40a9-9803-26bc34d00d57
tags: daily
comments: no
---

**建立过程**

- 打开笔记本无线网卡开关

- “开始”-->“搜索程序和文件”框中输入cmd-->右键单击“cmd.exe”-->选择“以管理员身份运行”

- 输入以下命令。若“支持的承载网络”为是，则继续；若为否，则此方法作废。

```python
	netsh wlan show drivers
```
- 输入以下命令。其中ssid为该无线网络名称（英文），key为密码（至少8位）。

```python
	netsh wlan set hostednetwork mode=allow ssid=*** key=********
```
- 在“控制面板”-->“网络和Internet”-->“网络连接”中，可以看到多出了一个“无线网络连接 X”(X 为某个数字)。即，我们新建立了“无线网络连接 X”这个无线局域网。

（或者，点击右下角网络标志-->“打开网络和共享中心”-->“更改适配器设置”）

- 右键单击已经连接到Internet的网络连接-->“属性”-->“共享”标签-->“允许其他网络用户通过此计算机的Internet连接来连接(N)”-->“请选一个专用的网络连接”-->“无线网络连接 X”-->“确定”。即将已经连接到Internet的网络连接共享给我们新建立的局域网络连接——“无线网络连接 X”。

- 在cmd的命令行中输入以下命令，该无线网**即可使用**。

```python
	netsh wlan start hostednetwork
```
--- 
**使用过程**

- 开始”-->“搜索程序和文件”框中输入cmd-->右键单击“cmd.exe”-->选择“以管理员身份运行”

- 在cmd的命令行中输入以下命令。

```python
	netsh wlan start hostednetwork
```

--- 
**注意**

- 关闭该无线网的命令如下。如果该无线网不稳定，关闭它并重新开启常常会有效果。

```python
	netsh wlan stop hostednetwork
```
- 如果已经连接到Internet的网络连接有改变，则需要将建立过程中的第6步根据新的网络连接重新设置。
