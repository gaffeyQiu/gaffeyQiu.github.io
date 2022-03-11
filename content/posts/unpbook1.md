---
title: "[笔记]《Unix网络编程卷一》第一章"
date: 2022-03-11T23:14:18+08:00
tags: ["TCP/IP", "network", "socket"]
categories: ["计算机网络"]
author: "gaffey"
---

## 环境搭建

只考虑 Linux 环境的话

1. 获取源代码 `git clone https://github.com/unpbook/unpv13e.git`
2. `cd unpv13e`
3. `./configure`
4. `cd lib`
5. `make`
6. `cd ../libfree`
7. `make`

验证环境是否成功

```sh
cd ../intro
make all
// 先在一个终端启动服务端
./daytimetcpsrv
// 另一个终端执行客户端，能正常返回当前时间
./daytimetcpcli 127.0.0.1
Fri Mar 11 22:44:07 2022
```



## OSI 模型

OSI 全称计算机通信开放互联系统（open systems interconnection），这是一个七层模型，非常经典

![](https://ypy.7bao.fun/img/202203112248946.png)

其中最底下两层是硬件提供，通常情况下只需要记住 MTU 是 1500 字节就够了，中间这幅图，TCP 和 UPD 之间有一个空隙，表明了上层的网络应用绕过传输层直接使用 IPv4 和 IPv6 协议是有可能的，比如***原始套接字（raw socket）***

OSI 最上面的三层可以合并成应用层，就是 web 客户端、 web 服务器、FTP 服务器以及绝大多数使用网络的应用所在层

最右边这幅图，区分了用户进程和内核的分界，TCP、UDP 和他的下层是由内核提供，应用程序不需要处理相关通信细节：例如发送数据，等待确认，给无序数据排序，计算校验和等，内核提供 socket api 让用户程序访问



## 总结

整个第一章开头讲了 c 语言写的 tcp 时间服务器和客户端的例子，这个没什么说的，代码直接在源码里看就行，难的就是 c 语言一些网络库的函数不会用，死记硬背不行，只有边学边看文档了，然后了解 OSI 七层模型都是干嘛的，但是重点只用关注传输层和网络层即可，因为我们需要在这两层上面构建网络应用，最后就是网络 debug 能力，简单介绍了 netstat 、ifconfig 命令，能看懂网络拓扑图

![](https://ypy.7bao.fun/img/202203112310042.png)


