---
title: CUDA中的共享内存和常量内存
date: 2023-11-12 21:01:55
---

进行CUDA编程时，需要掌握如何使用GPU的共享内存、常量内存加速计算，在此总结一下这两种内存的使用方法。

第一个遇到的问题是：在使用CUDA计算向量内积时，通用的做法是定义多个block，每个block由多个thread组成，为什么要这么做呢，对于长度为N的vector，能不能分配N个block，将核函数拷贝N份进行并行计算。

## 硬件限制block数量

由于GPU的硬件结构限制，对于一个核函数分配的block个数是有限的，当vector长度很大时，不能满足需求，因此需要使用block中的thread。

## 线程间共享内存

thread的另一个好处是可以使用共享内存进行block内的通信。

与`__device__`和`__global__`类似，CUDA C使用`__shared__`关键字扩展了C语法，在声明一个变量时，可以加上这个关键字将其存放在共享内存中。

共享内存变量最大的特点是GPU驱动为每一个block分配了一个副本，block内的threads共享这个变量，并且无法访问其他block中的同名变量，由于共享变量在GPU中，访问速度比DRAM中的变量快得多。

我们马上就想到了block中的thread可以通过共享内存进行通信协作，但是随之而来的问题是共享变量存在竞争条件，thread之间需要同步，可以使用`__syncthreads`保证线程的阻塞同步。

## 常量内存

GPU上有很多很多的计算单元，非常适合拿来做通用计算任务，与CPU的不同之处在于，在计算任务中GPU的内存带宽是性能瓶颈。甚至一些计算单元因为得不到足够多的输入，处于空闲状态。

使用常量内存而不是全局内存，是减少内存带宽需求的方式之一。顾名思义，在一个核函数的执行期间，常用内存用来存放常量值，例如一些GPU提供了64KB的常量内存大小。

书中使用了光线追踪的例子介绍了常量内存的使用场景，就光线追踪而言，代码示例如下：

```cpp
#define INF 2e10f

struct Sphere {
    float r, b, g;
    float radius;
    float x, y, z;
    // 像素坐标：(ox,oy) 返回像素点到球体交点的距离
    __device__ float hit(float ox, float oy, float *n) {
        float dx = ox - x;
        float dy = oy - y;
        if (dx*dx + dy*dy < )
    }
};
```

cherry-pick

缺页异常

UDP头

coredump
ulimit

nullptr->类的虚函数和函数

gdb
gcc -O0 -g
