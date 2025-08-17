描述 docker 使用及原理

# 概述

## 架构

docker 是个典型的 C/S 架构，C 就是我们的 docker 客户端工具，S 就是 dockerd 守护进程，dockerd 又负责跟 registory 交互，拉取/上传 image 等。<br>

docker 跟 dockerd 可能在同一机器上，也可能不在同一机器上，真正负责容器管理的是 dockerd，而不是 docker。这点对于后续理解 dockerfile 中的 **上下文环境** 特别重要。

## 解决了什么问题

应用的打包、隔离、资源限制。

- 应用的打包

    通过 docker image 能力，将应用运行时依赖的上下文都打包到镜像中，保证了开发环境与运行环境的高度一致。

- 应用隔离

    通过 namespace 机制，隔离容器进程的视图。

- 应用的资源限制

    通过 cgroup 机制，限制容器的资源使用。

## 与 virtual machine 的区别

### virtual machine

通过软件模拟硬件并安装真正的操作系统，所以这个环境与宿主机隔离更加彻底。

- 优点

  与宿主机隔离更加彻底。

- 缺点

  比较重量级，性能比较差。

### docker

本质就是通过 namespace、cgroup 创建出来的沙箱环境，依赖宿主机环境。

- 优点

  更轻量、性能更好。

- 缺点

  与宿主机环境隔离的不彻底，比如 windows 的 exe 即使通过容器无论如何都不可能在 linux 宿主机中执行。

# 原理

本质
三板斧

## namespace

- 都有哪些 namespace

### network namespace

- 虚拟网桥是如何创建的
- iptables 有什么用

## cgroup

## union fs