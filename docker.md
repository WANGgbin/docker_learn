描述 docker 使用及原理

# 概述

## 与 virtual machine 的区别

## 架构

docker 是个典型的 C/S 架构，C 就是我们的 docker 客户端工具，S 就是 dockerd 守护进程，dockerd 又负责跟 registory 交互，拉取/上传 image 等。<br>

docker 跟 dockerd 可能在同一机器上，也可能不在同一机器上，真正负责容器管理的是 dockerd，而不是 docker。这点对于后续理解 dockerfile 中的 **上下文环境** 特别重要。

## 解决了什么问题

# 使用


# 原理

本质
三板斧

## namespace

- 都有哪些 namespace

### network namespace

- 虚拟网桥是如何创建的
- iptables 有什么用
- 
## cgroup

## union fs