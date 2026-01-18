---
title: 使用perf进行网络性能分析及调优
date: 2024-01-05 20:57:43
---

在实现一个系统之后，我们总希望能够优化它的性能，系统性能优化通常可以分为两个阶段：性能分析和性能优化。其中，性能分析的目的是查找性能瓶颈、热点代码，分析引发性能问题的原因。perf这个性能分析工具在分析网卡延时性能时非常方便，在此对perf进行学习与总结。

## perf简介

perf是Linux的一款性能分析工具，能够进行函数级和指令级的热点查找，可以用来分析程序中热点函数的CPU占用率，从而定位性能瓶颈。

就函数级分析而言，perf的基本工作方式是每隔一段固定时间，在CPU上产生一个中断，查看当前CPU上运行的是哪个进程、哪个函数，然后给对应的进程和函数的计数加一，这样就知道CPU有多少时间在某个进程或某个函数上，perf的缺省采样频率是4000/s。

<!-- more -->

安装perf：

```sh
sudo apt install linux-tools-common linux-tools-generic linux-tools-`uname -r`
```

## perf的基础使用

使用perf进行性能分析，主要使用下面两个命令：

- perf record: 保存perf追踪的内容，文件名为perf.data
- perf report: 解析perf.data的内容

比如要分析进程xxx，启动该进程后，首先启动使用下面命令：

```sh
sudo perf record -a --call-graph dwarf -p `ps aux | grep "xxx" | grep -v grep | cut -c 9-15` -d 1 -b
```

其中，

- -a: 表示对所有CPU采样
- --call-graph dwarf: 表示分析调用栈的关系
- -p: 表示分析指定的进程

通过Ctrl+C结束后，会生成perf.data文件，然后通过report导出报告，即可查看main函数和子函数的CPU平均占用率。

```sh
sudo perf report -i perf.data > perf.txt
```

## perf生成火焰图

### 火焰图

火焰图（Flame Graph）是性能优化大师Bredan Gregg创建的一种性能分析图表，因为它的样子近似火焰而得名。使用火焰图能够非常快速地定位到代码中的瓶颈。

火焰图的上下表示调用关系，下层的函数调用上层的函数，每个函数对应一个等高的长条，这个长条对应值该函数的某些指标，如cpu time、real time、memory allocate等。

根据不同的指标，火焰图可以分为：

- CPU火焰图，显示代码中函数的cpu耗时，函数的宽度与cpu时间成正比，主要用于分析进程中的cpu瓶颈。
- real火焰图，显示代码中函数的duration耗时，函数宽度与real time成正比，与cpu火焰图结合可以用于分析进程中的IO瓶颈。
- mem火焰图，显示代码中函数申请内存的大小，函数宽度与memory allocate成正比，主要用于优化进程的内存占用。

### perf + FlameGrpah

下载火焰图绘制工具：

```sh
git clone https://github.com/brendangregg/FlameGraph.git
export PATH=$PATH:/home/software/FlameGraph/
```

生成perf.data文件后，执行：

```sh
perf script > perf.script
```

生成火焰图，并使用浏览器打开：

```sh
stackcollapse-perf.pl perf.script | flamegraph.pl > workrun.svg
```