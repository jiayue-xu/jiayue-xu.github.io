---
title: Architecture of openmpi
date: 2024-01-11 19:47:15
---

本文对open-mpi(ompi)项目的整体架构进行总结，ompi作为一个庞大的项目，在设计之初就有以下几点设计理念：

1. 将类似的功能放到到各自互相独立的抽象层里
2. 通过加载动态库、设置参数的方式，在运行时为相同的行为选择具体的实现方式
3. 对各个模块的抽象设计不能影响程序的性能

<!--more-->

## ompi的抽象层

ompi主要有三大抽象层，自下而上地有：OPAL、ORTE、OMPI

### OPAL

Open, Portable Access Layer，作为ompi最底下的抽象层，直接面向操作系统，提供了基础性功能以及用于适配不同操作系统的兼容性代码，包括且不限于：通用链表、字符串操作、调试控制、IP接口、共享内存、进程与内存亲近性、高精度定时器等。

### ORTE

Open MPI Run-Time Environment，作为一个运行时系统，负责启动、监视和终止并行任务。在ompi中，并行任务由一个或多个进程组成，这些进程可能不在同一个操作系统中，需要调度器和资源管理器在这些进程之间公平地分配共享计算资源。

### OMPI

Open MPI，实现MPI标准定义的各种通信语义，并为应用程序提供了MPI API，**需要兼容各种网络类型和底层协议**。

大体上各个抽象层之间是互相独立的，每个抽象层都会被编译成一个独立的库，并且上层的库依赖于下层的库。

但这种划分也不是死板的，比如说在OMPI层中，为了获得更好的网络性能，可能会采用RDMA网络方案，OPMI层可以调用网卡的用户态驱动，从而越过ORTE和OPAL甚至操作系统，直接向网卡硬件下发RDMA Write/Read等网络请求。

![open-mpi-layers](open-mpi-layers.jpg)

## 抽象层中的框架

## 框架中的组件

## 运行时参数



参考文献：

【1】[The Architecture of Open Source Applications (Volume 2)
Open MPI Jeffrey M. Squyres](https://aosabook.org/en/v2/openmpi.html)