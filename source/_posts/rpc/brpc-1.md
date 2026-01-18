---
title: bRPC
date: 2024-06-03 19:25:29
---

## RDMA初始化

如果要使用RDMA作为传输层设备，先调用`GlobalRdmaInitializeOrDie`进行初始化，这里使用`pthread_once_t`保证只进行一次初始化工作。

```c
static pthread_once_t initialize_rdma_once = PTHREAD_ONCE_INIT;

void GlobalRdmaInitializeOrDie() {
    if (pthread_once(&initialize_rdma_once,
                     GlobalRdmaInitializeOrDieImpl) != 0) {
        LOG(FATAL) << "Fail to pthread_once GlobalRdmaInitializeOrDie";
        exit(1);
    }
}
```

在GlobalRdmaInitializeOrDieImpl函数中，首先调用ReadRdmaDynamicLib加载ibverbs动态链接库，将verbs API符号引入到当前进程中。目的是将ibverbs封装一下，在NCCL中也有类似的写法。

接着调用了`ibv_fork_init`函数，这个函数主要是保证了程序在进行fork系统调用时，MR的内存不会因为写时拷贝而引发DMA数据不一致性的问题。

很常规的一套初始化流程，调用`ibv_get_device_list`获取系统中探测到的RDMA设备列表，brpc使用gflags控制参数，用法如下：默认使用探测到的第一个RDMA设备，除非`FLAGS_rdma_device`指定了要使用的设备名称。

```c
DEFINE_string(rdma_device, "", "The name of the HCA device used "
                               "(Empty means using the first active device)");
DEFINE_int32(rdma_port, 1, "The port number to use. For RoCE, it is always 1.");
```

在`OpenDevice`中，使用到了`unique_ptr`，`ibv_context`是要创建的指针类型，`IbvContextDeleter`是自定义的删除器函数，在context对象销毁时会被调用。**但是在这里直接调用了release释放指针的所有权，后续需要手动delete指针，否则会引起内存泄露？**

```cpp
std::unique_ptr<ibv_context, IbvContextDeleter> context{
            IbvOpenDevice(g_devices[i]), IbvContextDeleter()};
```

获取RDMA设备对应的ibv_context后，获取对应port的gid，注意：IB的一个端口对应一个GID，RoCE的一个端口对应两个GID，其中第二个是v2的GID。

接着创建`comp_channel`，用于后续的CQE事件通知，以及用于创建MR的PD。

### MR内存池初始化

1. 创建了`g_user_mrs_lock`用于保护MR相关的临界区。

2. 初始化用户MR数据结构：创建一个brpc自定义的hashmap，key的类型为void*，value的类型为ibv_mr*，接着调用FlatMap的`init`函数，初始化bucket的数量为65535。

3. 初始化brpc内部MR数据结构：一个指向`ibv_mr`vector的指针。

4. max_sge取用户参数和设备属性的较小值

```cpp
g_user_mrs_lock = new (std::nothrow) butil::Mutex;
if (!g_user_mrs_lock) {
    PLOG(WARNING) << "Fail to construct g_user_mrs_lock";
    ExitWithError();
}

g_user_mrs = new (std::nothrow) butil::FlatMap<void*, ibv_mr*>();
if (!g_user_mrs) {
    PLOG(WARNING) << "Fail to construct g_user_mrs";
    ExitWithError();
}

if (g_user_mrs->init(65536) < 0) {
    PLOG(WARNING) << "Fail to initialize g_user_mrs";
    ExitWithError();
}

g_mrs = new (std::nothrow) std::vector<ibv_mr*>;
if (!g_mrs) {
    PLOG(ERROR) << "Fail to allocate a RDMA MR list";
    ExitWithError();
}
```

5. 初始化brpc内部使用的MR

```cpp
typedef uint32_t (*RegisterCallback)(void*, size_t);

// Initialize RDMA memory pool (block_pool)
if (!InitBlockPool(RdmaRegisterMemory)) {
    PLOG(ERROR) << "Fail to initialize RDMA memory pool";
    ExitWithError();
}
```