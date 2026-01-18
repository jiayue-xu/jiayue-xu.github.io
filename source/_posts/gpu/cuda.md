---
title: CUDA By Example
date: 2023-10-29 21:01:55
---

## Vector addtion

向量加法C = A + B是最简单的数据并行计算，相当于入门级的“Hello world”。由于硬件的限制，单次分配的block和每个block中的thread数量是有限的，因此如果vector的长度很长，需要将block和thread配合起来使用。

```c
__global__ void vecAddKernel(float* A, float* B, float* C, int n) { 
    int i= blockDim.x*blockIdx.x + threadIdx.x; 
    if(i<n) C[i] = A[i] + B[i]; 
}

void vecAdd(float *A, float *B, float *C, int n) {
    int size = n * sizeof(float);
    flost *d_A, *d_B, *d_C;

    cudaMalloc((void**)&d_A, size);
    cudaMemcpy(d_A, A, size, cudaMemcpyHostToDevice);
    cudaMalloc((void**)&d_B, size);
    cudaMemcpy(d_B, B, size, cudaMemcpyHostToDevice);

    cudaMalloc((void**) &d_C, size);
    vecAddKernel<<<ceil(n/256.0, 256)>>>(d_A, d_B, d_C, n);

    cudaMemcpy(C, d_C, size, cudaMemcpyDeviceToHost);

    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
}
```

CUDA版本的vertor addition分为以下几个步骤：

1. 在程序的一开始需要包含`cuda.h`头文件，它包含了CUDA API函数和内建变量
2. 申请GPU内存，将主机内存中的向量A、B拷贝到设备内存中
3. 在GPU上并行执行向量加法核函数
4. 将结果拷贝回主机内存，并释放设备内存

GPU有自己的DRAM，称为全局内存（global memory），上述的第2步和第4步在主机内存和设备全局内存中来回传输数据。`cudaMalloc`可以申请设备全局内存，使用完后用`cudaFree`释放设备内存。`cudaMemcpy`可以在主机内存和设备内存之间传输数据。

在第3步中，所有GPU上的线程执行相同的程序，称为单程序多数据（SPMD）并行编程。

- 当主机代码启动核函数，CUDA运行时系统创建一个包含多个线程的网格（grid），grid由线程块（block）组成，每个block最多包含1024个线程。由于硬件特性，线程块中每个维度的线程个数应该是32的倍数。？
- 内建变量threadID、blockID存放在设备寄存器中，每个线程都有它们自己的寄存器存放这些值。
- 核函数中的i是局部变量，这是每个线程私有的变量。
- 线程块之间的执行顺序是随机的

<!-- more -->

## Image Greyscale

计算一张图像中每个像素的灰度值

```c
__global__ void colorToGreyscaleConversion(unsigned char *pout, unsigned char *pin, int width, int height) {
    int col = blockDim.x * blockIdx.x + threadIdx.x;
    int row = blockDim.y * blockIdx.y + threadIdx.y;
    if (col < width && row < height>) {
        int greyOffset = row * width + col;
        int rgbOffset = greyOffset * 3;
        unsigned char r = pin[rgbOffset];
        unsigned char g = pin[rgbOffset + 1];
        unsigned char b = pin[rgbOffset + 2];
        pout[greyOffset] = 0.21f*r + 0.71f*g + 0.07f*b;
    }
} 

int main() {
    ...
    // m(col) * n(row) image
    dim3 dimGrid(ceil(m/16.0), ceil(n/16.0), 1);
    dim3 dimBlock(16, 16, 1);
    colorToGreyscaleConversion<<<dimGrid, dimBlock>>>(d_Pout, d_Pin, m, n);
    ...
}

```

## Device resources

在硬件中，计算资源被划分为一个个的SM，一个SM可以同时分配的block个数是有限的，同时每个block中的thread个数也是受限的。因此，应用程序需要查询硬件的各种能力，才能避免软件超出了硬件的限制。

```c
int device_count;
cudaGetDeviceCount(&device_count);

cudaDeviceProp dev_prop;
for (int i = 0; i < device_count; i++) {
    cudaGetDeviceProperties(&dev_prop, i);
    // dev_prop.maxThreadsPerBlock
    // dev_prop.multiProcessorCount
    // dev_prop.clockRate
    // dev_prop.maxThreadsDim[0~2]
    // dev_prop.maxGridSize[0~2]
    // dev_prop.warpSize
}
```

其中，warp是SM中线程调度的基本单元，warp由一组线程组成，warp中的线程个数是由硬件设计决定的，现有的硬件基本上是32个连续的线程组成一个warp。以下图为例：分配了3个block给一个SM，每个block被进一步划分为多个warp。当一个warp被调度到的时候，SM会取出一条指令，warp中的线程同时执行同一条指令，这种方式被称为SIMD，每一个线程都在单独的SP上执行。实际上，SM中SP的个数往往少于线程的个数，SM在同一时刻只能满足一个或几个warp的需求，当warp中的线程需要执行内存访问、浮点数计算等指令时，在等待的时间里SM会调用其他warp。GPU的这种延时容忍能力使得它可以使用更多的芯片面积用于优化浮点数计算而不是缓存设计和分支预测上面。

{% asset_img warps_for_thread_scheduling.jpg %}

## Matrix multiplication

```c
__global__ void MatrixMulKernel(float * M, float *N, float *P, int width) {
    int row = blockIdx.y*blockDim.y + threadIdx.y;
    int col = blockIdx.x*blockDim.x + threadIdx.x;
    if (row < width && col < width) {
        float pvalue = 0;
        for (int k = 0; k < width; k++) {
            pvalue = M[row*width+k] * N[k*width+col];
        }
        P[row*width+col] = pvalue;
    }
}
```

这种实现方式的compute-to-global-memory-access ratio为1.0，大概只能发挥GPU峰值执行速度的2%，需要优化算法从而提高计算吞吐量。

