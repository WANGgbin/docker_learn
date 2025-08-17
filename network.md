描述 docker network 相关内容。

# network namespace

每个容器都有一个自己的 network namespace(默认)，也可以指定 network namespace，k8s pod 中的容器就共享一个 network namespace。
怎么理解 network namespace 呢？

network namespace 包括：协议栈、网卡、回环设备、路由表、iptables 等。每个容器所在的 network namespace 都是不同的(默认)。


# 同一宿主机，容器间通信

dockerd 会默认在宿主机创建一个虚拟的网桥 "docker0"。每个容器都会创建一个 veth(Virtual Ethernet) pair，即一对虚拟网卡，一端连接在网桥上，另一端作为容器 network namespace 中的虚拟网卡。<br>

docker0 和 容器中的 veth 处于同一个网段，每个容器的路由表中，会增加一条记录，如果目的 ip 地址属于这个网段，则使用直连的方式发送数据包，即获取目标 ip 地址对应的 mac 地址，然后发送到网桥，再由网桥转发给对应的容器。<br>

所以网桥本质就是个**交换机，工作在二层**，负责转发同一网段(同一宿主机的不同容器)之间的请求。

# 容器跟宿主机通信

宿主机的 ip 跟容器的 ip 不再同一个网段，因此当容器查看自己的路由表的时候，匹配到的下一条地址是网桥的 ip，因此发送的以太网帧的目的 mac 地址就是网桥的 mac 地址。<br>

此网桥跟交换机不同的是，如果收到的数据包的目的 mac 地址是自己的 mac 地址，则会将数据包交给宿主机的协议栈处理。<br>

# 宿主机跟容器通信

宿主机会多一条路由表，凡是网桥这一网段的目标 ip，都由 docker0 通过直连的方式发送。docker0 收到数据包后，通过 arp 获取到目标 ip 对应的 mac 地址，
然后通过指定端口发送到目标容器的 veth 虚拟网卡，然后虚拟网卡把数据发送给容器自己的协议栈。

# 容器跟非宿主机通信

先发送到 docker0，然后上传到宿主机协议栈，然后根据目标 ip 地址过路由表，后续走正常的通信流程即可。

# 不同宿主机容器间通信

首先我们需要明确：

不同宿主机容器间的通信是靠宿主机之间的通信完成的，所谓依靠宿主机间的通信能力，就是把“容器到容器的 ip包/以太网帧(不同的方法对应的数据包类型不同)” 是作为应用层数据，
然后通过宿主机的协议栈发送出去，在接受宿主机的角度，也会通过协议栈解析数据获得这个数据包，然后会转发给宿主机中特殊的虚拟网卡，该虚拟网卡上传数据到宿主机的协议栈，
然后通过路由表，将数据包发送给 docker0 网卡，然后 docker0 将数据转发给目标容器。

跨主机的容器通信有好几种解决方案，比如：flannel/ˈflænl/ udp、flannel vxlan。但不同方案基本是类似的，不同的地方在于：

- 工作在几层
- 如果将“容器之间的数据包”通过宿主机的协议栈发出去
- 接受宿主机解析到“容器之间的数据包”后，如何将数据转发给对应的虚拟网卡

## flannel udp

工作在三层。每一个宿主机都有一个特殊的 flannel 进程以及一个特殊的 tun。当 docker0 接收到数据包之后，上传到宿主机的协议栈，然后查看路由表，
将该 ip 数据包直接发送给 tun 设备，tun 接着将数据包发送到字符设备，flannel 进程从字符设备读取数据包，然后通过宿主机的协议栈发送一个 udp 数据包。

当接受端宿主机上的 flannel 进程接收到该数据包后，然后通过写字符设备的方式，将数据包发送给宿主机的 tun 设备，tun 接着将 ip 数据包上传给宿主机的协议栈，
因为目标 ip 地址是容器的地址，所以接着查看路由表，将数据包发送给 docker0 网卡，然后 docker0 将数据包发送给目标容器。

该方案比较简单，但问题是涉及到内核/用户态之间数据包的拷贝，导致性能比较差。

## flannel vxlan(虚拟扩展局域网)

相比于 udp 的方案，vxlan 所有封包、拆包的行为都发生在内核态，因此性能更好。<br>

该方案下，每个宿主机有个特殊的设备：vtep，简单理解就是个虚拟的网卡，工作在二层。同 udp 方案一样，容器到容器的数据包会通过宿主机的协议栈发送给 vtep，
然后 vtep 会加上目标容器对应的 mac 地址(加 mac 地址是因为该设备工作在二层)，那么如何获取目标容器的 mac 地址呢？并不是通过 arp 获取的，因
为压根不在同一网段，而是目标宿主机的 flannel 进程会将它的 vtep 的 mac 地址发送给当前宿主机。接着，linux 内核会给此数据帧加一个特殊的 vxlan header，
然后内核会把该数据封装成一个 udp 发送出去，发送 udp 的时候，怎么知道目的 ip 地址呢？这也是 flannel 完成的，内核中会存储 vtep mac 地址到 ip 地址的映射。

接受宿主机接收到 udp 后，发现有个 vxlan header，内核就会直接将此数据包发送给对应的 vtep 设备，然后 vtep 将数据包上传给协议栈，
然后协议栈根据路由表将数据发送给 docker0，最后由 docker0 发送给对应的容器。<br>

可以看到整个流程中，内核态和用户态之间是没有数据包拷贝的，所有操作全由内核完成。

# netfilter

是个内核框架，允许用户态程序在协议栈处理数据包的各个流程中注册 hook，从而能够介入网络数据包的处理。

# iptables

iptables 是在 netfilter 基础上的再抽象。所谓 tables 按功能划分，常见的 table 包括：filter、nat。每个 table 又包含若干 chain。怎么理解 chain 呢？
chain 对应一个时机，比如 input、output、preroute 等，这个时机我们可以设置多个规则，这些规则按顺序组成一个链条，这就是 chain 的由来。

通过 iptables，用户可以更方便的控制协议栈处理数据包的行为。关于 iptables 的详细使用，可以参考《鸟叔的 linux 服务器架设篇》

## 自定义 chain

- 为什么要自定义 chain

一句话：就是方便管理 chain。<br>

比如对于 filter.INPUT chain，我们针对 web 服务定义了一系列规则，又针对其他服务定义了一系列规则，如果我们要修改 web 服务相关的规则，就需要遍历 INPUT chain，然后找到所有跟 web 相关的规则，进行修改，这不方便。<br>

自定义 chain 的思路是将某一类规则专门交给一个 chain 管理，原始 chain 只要引用这个自定义 chain 即可，这样当后续有维护需求的时候，只需要变更这个自定义 chain 即可。当然，自定义 chain 也可以再引用 chain。<br>

这种 **分级** 的思路大大方便了 chain 规则的管理.<br>


- 如何自定义 chain

    - 定义 chain

```sh
    iptables -t filter -N custom_chain
```

    - 自定义 chain 增加规则

```sh
    iptables -t filter -I custom_chain -s xxx -d xxx -j xxx
```

    - 原始 chain 引用自定义 chain
  
```sh
    iptables -t filter -I INPUT -s xxx -d xxx -j custom_chain
```
> -j 除了可以指定具体的行为，还可以引用自定义 chain

## iptables 规则匹配流程

- 第一级规则如果都没有匹配到，则使用默认规则，只有默认的 chain(第一级 chain) 才有默认策略
- 其他级别的 chain，如果没有匹配到规则，则返回到上一级 chain 继续匹配
- 各个级别的 chain 可以通过 -j RETURN 的方式提前返回，就不用匹配当前 chain 后续的规则了

## 参考资料

- https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/#1-iptables-%E5%92%8C-netfilter-%E6%98%AF%E4%BB%80%E4%B9%88

# 容器端口映射

本质是在宿主机的 iptable nat 的 chain pre_route 上增加了一条 dnat 规则。当容器退出后，该规则也删除。

# 虚拟网卡


怎么理解虚拟网卡呢？虚拟意思是不是物理网卡，但是功能上(内核视角看)跟物理网卡完全一致。容器之间就是通过虚拟网卡的技术来通信的。

## lo 网卡
就是个特殊的虚拟网卡。


# 资料

关于 docker 的网络可以参考下面的资料：
- https://zhuanlan.zhihu.com/p/684307529
- 