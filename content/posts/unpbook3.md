---
title: "[笔记]《Unix网络编程卷一》第三章"
date: 2022-03-20T20:05:20+08:00
tags: ["TCP/IP", "network", "socket"]
categories: ["计算机网络"]
author: "gaffey"
---

# 套接字编程

前两张讲的是网络编程的理论知识，这一章就是告诉我们，如何在用户空间内利用内核提供的套接字接口使用我们的 TCP\IP 协议

## 主机字节序和网络字节序
数据在内存中的排列顺序影响着CPU解析出来的值，类似于你在纸上写123，习惯从坐往右读的人会理解成123， 从右往左的人会理解成321，计算机中就有大端序和小端序，大端序是指一个整数的高位字节存储在低地址处，小端序则是高位字节存储在高地址处，例如：0x0102， 小端序是 [0x01, 0x02] 这样排列， 大端序是 [0x02, 0x01] 这样排列。下面是检查机器的字节序代码：
```c
#include <stdio.h>
int main(int argc, char **argv)
{
	union {
		short value;
		char union_bytes[ sizeof(short) ];
	} test;

	test.value = 0x0102;
	if ( (test.union_bytes[0] == 1) && (test.union_bytes[1] == 2) )
	{
		printf("big endian\n");
	}
	else if ( (test.union_bytes[0] == 2) && (test.union_bytes[1] == 1) )
	{
		printf("little endian\n");
	}
	else
	{
		printf("unknown...\n");
	}
}
```
一般小端序叫主机字节序，大端序也叫网络字节序
> 即使同一个机器上的不同进程之间，也需要考虑字节序问题  
> 
Linux 提供了一套函数来完成主机字节序和网络字节序之间的转换
```c
#include <netinet/in.h>
unsigned long int htonl(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntonl(unsigned long int netlong);
unsigned short int ntons(unsigned long int netshort);
```
其中：htonl 代表 host to network long 即 长整型的主机字节序转网络字节序，其他以此类推。

## 通用 socket 地址
网络编程接口中定义了通用的 socket 地址 sockaddr 结构体， 但是很不好用， 一般会使用各个协议的专用socket地址，比如 UNIX 本地域协议族使用 sockaddr_un 结构体， TCP/IP 分别使用 sockaddr_in 结构体(IPv4)， sockaddr_in6 结构体(IPv6), 由于 socket 编程接口需要使用 通用socket地址，将专有socket地址强制转换一下即可

## socket 函数
整个 socket 连接需要用到函数的流程图如下  

![](https://ypy.7bao.fun/img/202203192338008.png)

+ `int socket(int family, int type, int protocol)`: 创建一个指定协议族(例如IPv4) 和 套接字类型(例如 字节流)的 socket 描述符
+ `int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)`: connect 函数用来建立与 TCP 服务器的连接。
+ `int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen)`: bind 函数将一个本地协议地址赋予一个套接字
+ `int listen(int sockfd, int backlog)`: listen 将一个未连接的套接字转换为被动套接字，指示内核应接受指向该套接字的连接请求，其中第二个参数规定了内核为相应套接字排队的最大连接个数。
+ `int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen)`: accept 函数由 TCP 服务端调用，用于从已完成连接队列中的队列头返回一个已完成连接，如果已完成连接队列为空，那么进程将进入睡眠(假设套接字为默认的阻塞方式)
+ `int close(int sockfd)`: 关闭套接字，并终止 TCP 连接。