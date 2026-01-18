---
title: RDMA性能
date: 2024-04-22 20:30:05
---

RDMA的应用场景广泛，有远程存储：GDS，深度学习：GDR。我们希望这些应用能够获得预期的RDMA网络性能（足够的带宽和延迟），以及写软件代码时知道怎么写能获得最大可用性能。

RDMA测试工具：

perftest，和TCP/IP中的iperf非常相似的

实际测试程序，GDR、GDS、MPI、NCCL、Pytorch


## Ref

- Collie: Finding Performance Anomalies in RDMA Subsystems. (NSDI 2022.)
- Husky: Understanding RDMA Microarchitecture Resources for Performance Isolation. (NSDI 2023.)