---
title: 单例设计模式
date: 2024-03-11 20:18:09
---

## 单例模式概述

单例模式是保证一个类仅有一个实例，并提供一个访问它的全局访问点。

要做到以上要求，保证一个类只有一个实例并且不能被实例化，可行的办法是：

1. 让类自身负责保存它唯一的实例
2. 这个类可以截取他人创建新对象的请求，并阻止请求
3. 提供一个访问该实例的方法


## 简单实现

```cpp
class Singleton {
public:
    static Singleton *getInstance() {
        if (instance == nullptr) {
            instance = new Singleton;
        }
        return instance;
    }
protected:
    Singleton();
private:
    static Singleton *instance;
};

Singleton * Singleton::instance = nullptr;

```

## 单例模式的优点

1. 对唯一实例的受控访问，控制用户的访问方式
2. 相比于把方法定义成静态方法的方式，Singleton类的子类可以重写它的方法，可以用于扩展这个类

