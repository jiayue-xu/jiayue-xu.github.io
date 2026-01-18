---
title: 实现一个内存动态分配器
date: 2024-02-26 18:55:14
---

选自于CSAPP课程的malloc lab，实现malloc、free、realloc函数。

## malloc概述

malloc分配出来的地址有对齐要求，在32-bit平台上需要8字节对齐，在64-bit平台上16字节对齐。##

## new与malloc

- new和delete是C++的关键字，malloc和free是C/C++的标准函数。

- 在使用上，malloc需要显式地指定分配的内存大小，new不需要。

- new操作符从自由存储区上为对象动态分配内存空间，而malloc函数从堆上动态分配内存。

- new分配成功时，返回对象类型，无需进行类型转换，故new是类型安全的；malloc返回void*，需要通过强制类型转换将void*指针转换成我们需要的类型。

- new分配失败时，抛出bad_alloc异常，malloc内存分配失败时返回NULL。

- new操作符在分配内存的同时，会调用对象的构造函数完成初始化，malloc仅仅分配空间。

- malloc分配空间后，可以通过realloc扩张内存，new不能再次扩张内存。

- new相对于malloc效率要低，因此new底层封装了malloc。

- 使用`new[]`分配一个数组时，实际上会多分配4个字节，并将元素个数填入一开始的4个字节中，将第5个字节的地址返回；在`delete[]`时，会把`new[]`返回的地址前移4个字节，并交给free去释放。


