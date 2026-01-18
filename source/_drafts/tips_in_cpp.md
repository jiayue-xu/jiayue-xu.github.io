---
title: Tips and Tricks in C/C++ Language
date: 2024-01-10 16:05:34
tags:
  - C/C++
---

## 字符串与整数的转换

### atoi

在C语言中，`atoi` (ASCII to Integer) 函数用于将字符串转换为整数，它的原型定义在 `<stdlib.h>` 头文件中:

```c
int atoi(const char *str);
```

## 宏定义

## 内建函数

### __builtin_constant_p

`__builtin_constant_ps`是一个内建函数，用于在编译时判断一个表达式是否为常量，它返回一个整数常量，如果表达式在编译时可以确定为常量，则返回1，否则返回0。

## pthread_mutex_lock


## C++ STL

### list

C++中，list是一个双向链表容器。

```cpp

# include <list>

std::list<T> myList;

myList.push_back(element);
myList.push_front(element);

myList.front();  // 返回列表的第一个元素的引用
myList.back();

// 迭代访问列表的元素
for (auto it = myList.begin(); it != myList.end(); ++it) {
    // 使用 *it 访问当前迭代器指向的元素
}

// 在指定位置插入元素
auto it = std::next(myList.begin(), index); // 获取指定位置的迭代器
myList.insert(it, element); // 在指定位置插入一个元素
myList.erase(it); // 删除指定位置的元素

myList.size();
myList.empty();
```
