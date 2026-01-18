---
title: RPC分布式训练
date: 2024-06-14 21:10:49
---

## Overview

分布式训练将模型训练负载分配到多个工作节点上，从而提升训练速度和模型准确率。尽管分布式训练可以被用于任何类型的机器学习模型，但对计算密集型的大模型优化更明显，比如深度学习。
 
教程给了两个使用`torch.distributed.rpc`的分布式训练示例，源码可以在[示例代码](https://github.com/pytorch/examples)找到。

有时候，你的训练模型比较特别，常见的分布式数据并行不太适用，比如：

1. 在增强学习中，模型本身比较小，并且从环境中获取训练数据代价也比较大。在这种情况下，让多个观察者并行运行同时共享单个代理进程会比较合适，代理进程负责在本地进行训练，但是也需要在训练者和观察者之间进行数据通信。
2. 你的模型可能很大，单台机器的GPU卡无法适配，此时可能需要将模型拆开到不同的机器上运行。或者你可能需要实现一个参数服务器训练框架，模型参数和训练者在不同的机器上。

`torch.distributed.rpc`适用于以上场景，在示例1中RPC和RRef允许从一个worker发送数据到另一个worker，并且轻松地引用远程数据对象。在示例2中，分布式自动评分和自动优化器让反向传播和优化步骤就像在本地执行一样。接下来给出对应的案例。

## 使用RPC和RRef进行分布式增强学习

使用RPC的分布式增强学习模型解决CartPole-v1，训练的策略可以借鉴以下的单线程代码：定义了一个神经网络模型，名为Policy，接收一个4维的输入x，init方法定义了神经网络的层，forward定义了神经网络的前向传播过程。总的来说，策略函数将4维的输入映射到两个可能动作的2为概率分布，通常用于强化学习算法，来在环境中作出决策并最大化奖励信号。

```python
import torch.nn as nn
import torch.nn.functional as F

class Policy(nn.Module):

    def __init__(self):
        super(Policy, self).__init__()
        self.affine1 = nn.Linear(4, 128) # 全连接线性层，输入4维，输出128维
        self.dropout = nn.Dropout(p=0.6) # dropout层，概率为0.6，防止过拟合
        self.affine2 = nn.Linear(128, 2) # 另一个全连接线性层，输入128维，输出2维 

    def forward(self, x):
        x = self.affine1(x)
        x = self.dropout(x)
        x = F.relu(x)   # 应用ReLU激活函数
        action_scores = self.affine2(x)
        return F.softmax(action_scores, dim=1) # 将2维输出归一化为概率分布
```

## Ref

- [Pytorch RPC distributed training](https://pytorch.org/tutorials/intermediate/rpc_tutorial.html?utm_source=distr_landing&utm_medium=rpc_getting_started)
