---
title: "Docker Daemon 介绍"
date: 2022-03-07T20:28:10+08:00
tags: ["docker"]
categories: ["docker"]
featuredImagePreview: "https://ypy.7bao.fun/img/20220307203314.png"
author: "gaffey"
---

# Docker Deamon 介绍
> 摘抄自《深入浅出docker》
## Docker 引擎
Docker 引擎是用来运行和管理容器的核心软件。
基于开放容器计划OCI相关标准，Docker引擎采用了模块化的设计原则，其组件是可替换的。
![](https://ypy.7bao.fun/img/20220305114844.png)

## Docker引擎详解
### 旧版 Docker
Docker 引擎首次发布的时候，由两个核心组件构成：LXC 和 Docker daemon
+ Docker deamon 是一个单一的二进制文件，由 Docker Client 、Docker API、容器运行时、构建镜像等组成
+ LXC 由命名空间 Namespace 和 控制组 CGroup 等基础工具组成，由 Linux 内核的容器虚拟化技术提供。
![](https://ypy.7bao.fun/img/20220305115253.png)

### 摆脱 LXC
LXC 是基于 Linux 的， 这对于跨平台来说是一个致命的问题，其次，如此核心的组件依赖外部提供，这会给项目带来巨大风险，甚至会影响项目的发展。
因此 Docker 自研了 Libcontainer 代替 LXC，Libcontainer 与平台无关，可基于不同的内核为 Docker 上次提供必要的的容器交互能力。
+ Docker 0.9 版本中，Libcontainer 已经取代 LXC 成为默认的执行驱动

### 摒弃大而全的 Docker daemon
随着时间的推移，Docker daemon 越来越大， 导致难于变更，运行越来越慢，于是准备拆解 Docker daemon 进程，使其模块化，并且这些子模块是可替换的，也可以被第三方使用。这一计划遵循了 Unix 中的软件设计哲学： 小而专的工具可以组装为大型工具。
目前 Docker 引擎的架构图如图所示
![](https://ypy.7bao.fun/img/20220305120022.png)

## runc 和 containerd
根据 OCI 容器运行时规范，针对 Libcontainer 进行包装了一个轻量级命令行工具 runc
runc 只有一个作用，就是创建容器，所以它非常快

containerd 的主要任务是容器的生命周期管理，start、stop、pause、rm 等
containerd最初被设计为轻量级的小型工具，仅用于容器的生命周期管理。然而，随着时间的推移，它被赋予了更多的功能，比如镜像管理
> 其原因之一在于，这样便于在其他项目中使用它。比如，在Kubernetes中，containerd就是一个很受欢迎的容器运行时。然而在Kubernetes这样的项目中，如果containerd能够完成一些诸如push和pull镜像这样的操作就更好了。因此，如今containerd还能够完成一些除容器生命周期管理之外的操作。不过，所有的额外功能都是模块化的、可选的，便于自行选择所需功能。所以，Kubernetes这样的项目在使用containerd时，可以仅包含所需的功能

## 启动一个容器的流程
```
$ docker run --name ctrl -it alpine:latest sh
```
1. 当使用 Docker 命令行工具执行如上命令时，Docker 客户端会将其转换为合适的 REST API 发送到 Docker deamon
2. daemon 接受到创建新容器的请求，通过 gRPC 向 containerd 发起调用
3. containerd 并不负责创建容器，containerd 将 Docker 镜像转换为 OCI bundle 然后让 runc 基于此创建一个新的容器
4. runc 与操作系统API进行通信，基于 Namespace CGroup 等创建容器，容器进程作为 runc 的子进程启动，启动完毕后， runc 会退出
![](https://ypy.7bao.fun/img/20220305121238.png)

## shim
图片中展示了 shim， 它是什么作用呢？
shim 是实现无 daemon 的容器不可或缺的工具

前面提到，containerd 指挥 runc来创建新容器。事实上，每次创建容器时它都会fork一个新的runc实例。不过，一旦容器创建完毕，对应的runc进程就会退出。因此，即使运行上百个容器，也无须保持上百个运行中的runc实例。

一旦容器进程的父进程runc退出，相关联的containerd-shim进程就会成为容器的父进程。作为容器的父进程，shim的部分职责如下：
+ 保持所有STDIN和STDOUT流是开启状态，从而当daemon重启的时候，容器不会因为管道（pipe）的关闭而终止。
+ 将容器的退出状态反馈给daemon。