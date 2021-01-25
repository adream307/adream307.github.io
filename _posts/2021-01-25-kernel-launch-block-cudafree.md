---
title:  "在launch kernel的过程中不能调用cudaFree"
author: adream307
date:   2021-01-25 15:19:00 +0800
categories: [linux,cuda]
tags: [cuda]
---

测试发现 `cuda` 的 `launch kernel` 的过程中不能释放该 `gpu` 上的显存，`cuda` 任务被释放的显存可能被 `kernel` 调用，所以禁止在 `laucn kernel` 过程中释放该 `gpu` 上的显存。
测试代码如下：

```c++
#include <stdio.h>
#include <thread>
#include <unistd.h>
#include <iostream>
#include <cuda_runtime.h>
#include <vector>

#define GRID_X  128
#define BLOCK_X 128

__global__ void proc(int *mem){
    int idx = blockDim.x * blockIdx.x + threadIdx.x;
    while(true){
        mem[idx]++;
    }
}


void run_thread()
{
    cudaSetDevice(0);
    int *pdata;
    cudaMalloc(&pdata,GRID_X*BLOCK_X*sizeof(int));
    sleep(1);
    std::cout << "start kernel in run_thread" << std::endl;
    proc<<<GRID_X,BLOCK_X>>>(pdata);
    cudaDeviceSynchronize();
    std::cout << "end kernel in run_thread" << std::endl;
}

void free_thread()
{
    cudaSetDevice(0);
    //cudaSetDevice(1);
    int *pdata;
    cudaMalloc(&pdata,GRID_X*BLOCK_X*sizeof(int));
    sleep(5);
    std::cout << "start cudaFree in free_thread" << std::endl;
    cudaFree(pdata);
    std::cout << "end cudaFree in free_thread" << std::endl;
}


int main(){
    std::thread th0(&run_thread);
    std::thread th1(&free_thread);
    th0.join();
    th1.join();
    return 0;
}
```

运行过程中会发现 `free_thread` 的会阻塞在 `cudaFree` 函数上。

但是如果 `cudaFree` 释放另一块 `gpu` 上的显存，则完成不存在这个问题。

包上述代码 `free_thred` 线程中的 `cudaSetDevice(0)` 改成 `cudaSetDevice(1)` 则 `cudaFree` 可以被正常调用而不会被阻塞。
