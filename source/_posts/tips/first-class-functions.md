---
title: 经常遇到的Lambda表达式
date: 2024-03-04 20:50:52
tags:
  - C++
---

在自定义比较函数的地方经常看到简洁的lambda表达式的写法，在此对C++的first-class function特性进行学习和总结。

如果一门编程语言将函数视为一种变量，那么称其支持first-class function，C++就是其中一个，由此有了函数指针、函数对象、lambda表达式的概念。

<!--more-->

## 函数对象

就像智能指针封装了裸指针，函数对象(function objects)或者说仿函数(functors)封装了裸函数指针。


将一个函数封装到一个类里很简单，只需要重载这个类的函数调用操作符即可，如下：

```cpp
#include <iostream>

using namespace std;
 
class Sum {
    public:
        int operator()(int a, int b) const {
            return a + b;
        }
};
 
int main()
{
    Sum sum;
    cout << sum(2, 3) << " " << sum.operator()(1, 2);
    return 0;
}
```

### 仿函数对象作为函数参数

### 参数化的仿函数对象


## Lambda表达式

