---
title: NCCL初始化
date: 2024-05-11 12:44:27
---

```c
ncclResult_t ncclGetUniqueId(ncclUniqueId* out) {
  NCCLCHECK(ncclInit());
  NCCLCHECK(PtrCheck(out, "GetUniqueId", "out"));
  ncclResult_t res = bootstrapGetUniqueId((struct ncclBootstrapHandle*)out);
  TRACE_CALL("ncclGetUniqueId(0x%llx)", (unsigned long long)hashUniqueId(*out));
  return res;
}
```

```c
pthread_mutex_t initLock = PTHREAD_MUTEX_INITIALIZER;
static bool initialized = false;

static ncclResult_t ncclInit() {
  if (__atomic_load_n(&initialized, __ATOMIC_ACQUIRE)) return ncclSuccess;
  pthread_mutex_lock(&initLock);
  if (!initialized) {
    // 初始化环境变量
    initEnv(); 

    // 初始化GRD copy
    initGdrCopy();
    
    // 初始化bootstrap网络，主要用于交换简单信息，比如ip端口，数据量小，使用的是TCP
    NCCLCHECK(bootstrapNetInit());

    // 初始化通信网络，实际数据的传输，优先使用RDMA
    NCCLCHECK(ncclNetPluginInit());

    initNvtxRegisteredEnums();
    __atomic_store_n(&initialized, true, __ATOMIC_RELEASE);
  }
  pthread_mutex_unlock(&initLock);
  return ncclSuccess;
}
```

## 网络设备抽象层

在`net_ib.h`文件中，定义了两个重要的结构体：`ncclNetProperties_t`和`ncclNet_t`

- `ncclNetProperties_t` - 网络属性
- `ncclNet_t` - 网络方法

## 网络设备实现层

`net_ib.cc`和`net_socket.cc`文件分别注册了各自的`nccLNet_t`对象，


