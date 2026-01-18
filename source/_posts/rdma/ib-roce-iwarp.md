---
title: IB、RoCE、iWARP
date: 2024-04-20 10:51:08
---

RDMA的特点是提供了远程访问远端虚拟内存的特性，以及网卡到外设的直接访问路径。三种支持RDMA的技术：InfiniBand、RoCE和iWARP都实现了同一套用户API——ibverbs，但是物理层和链路层实现不同。

![协议比较](cmp_ib_roce_iwarp.jpg)

## InfiniBand

IB链路层提供了基于credit的流控机制，用于支持缓存到缓存的拥塞解决方案。

IB支持虚拟通道（Virtual Lanes），简化了上层协议，提供了QoS。

IB的传输层提供了可靠传输。

成本较高，因为需要使用IB专用交换机

## RoCE

支持以太网方案的有RoCE和iWARP两种方案，从带宽、延迟、CPU占用率三方面看，RoCE的性能都是更优的。

和IB相比成本更低，可以使用以太网交换机

## iWarp

使用TCP作为网络层保证可靠性，性能上有损失

## Ref

[Basic Knowledge and Differences of RoCE, IB, and TCP Networks](https://support.huawei.com/enterprise/en/doc/EDOC1100203339)