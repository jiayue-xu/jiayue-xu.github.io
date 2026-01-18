---
title: DPDK无锁队列
date: 2024-05-05 18:42:31
---

## 创建ring


```c
struct rte_ring *rte_ring_create(const char *name, unsigned int count, int socket_id, unsigned int flags);
```

使用示例：

```c
const char *NAME = "ring_name";
struct rte_ring *r;
r = rte_ring_create(NAME, 64, socket_id, 0);
```

