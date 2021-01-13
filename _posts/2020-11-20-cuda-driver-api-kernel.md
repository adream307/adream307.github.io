---
title:  "[CUDA]关于Drive API中Kernel函数参数的一个坑"
author: adream307
date:   2020-11-20 16:16:00 +0800
categories: [gpu,cuda]
tags: [cuda]
---

在 `CUDA` 的 `Drive API` 中 `launch kernel` 函数原型如下:
```cpp
CUresult CUDAAPI cuLaunchKernel(CUfunction f,
                                unsigned int gridDimX,
                                unsigned int gridDimY,
                                unsigned int gridDimZ,
                                unsigned int blockDimX,
                                unsigned int blockDimY,
                                unsigned int blockDimZ,
                                unsigned int sharedMemBytes,
                                CUstream hStream,
                                void **kernelParams,
                                void **extra);
```

在该函数中 , 参数 `kernelParams` 是个指针的指针。

举例说明：

如果我们的 `kernel` 函数原型如下:
```cpp
void ktest(void *p1, void*p2, void* p3, void*p4, void*p5, void*p6);
```

假设我们现在有 6 个 `void*` 的指针变量，

```cpp
void *p1,*p2,*p3,*p4,*p5,*p6;
```

那么  `kernelParams` 的值不能是这个样子：

```cpp
void* kernelParams[]= {p1,p2,p3,p4,p5,p6}
```

而必须是这个样子
```cpp
void*kernelParams[]= {(void*)&p1,(void*)&p2,(void*)&p3,
                      (void*)&p4,(void*)&p5,(void*)&p6}
```

现在我们假设  `p1~p6` 是通过函数调用获得的，简单说我们可以获得 `p1~p6` 的值，但是我们无法获得 `p1~p6` 的地址。

那么在这种情况下，我们常常是这么处理的：

首先定义一个 `std::vector<void*> para_value`，用于存储 `p1~p6` 的值，间接的获得一个p1~p6的地址，具体实现如下:

```cpp
std::vector<void*>para_value;
std::vector<void*>kernel_para;
for(int i=0;i<6;++i){
    para_value.push_back((void*)(0x10000+i));
    kernel_para.push_back(&(para_value.back()));
}
```

在上面的代码中，我们假设 `p1~p6` 的值为 `0x10000~0x10005`

但是上面这段代码有一个非常隐蔽的坑。

在上面代码中，我们是通过 `para_value` 来存储 `p1~p6` 的值，然后通过 `para_value` 成员的地址简介获得 `p1~p6` 的地址，这里做了一个隐式的假设，就是 `para_value` 成员的地址是不会变的。

但是，在上面的这段代码中，我们是动态的向 `para_value` 添加 `p1~p6` 的值，并且每次都是动态的把`para_value` 最后一个成员的地址添加到 `kernel_para` 中的。如果在某次向 `para_value` 添加值时， `para_value` 满了，这样就会触发 `para_value` 重新分配更大的容量，在重新分配的过程中就无法保证 `para_value` 成员的地址保持不变了。

填这个坑的方法有两种：

方法一：

首先让 `para_value` 获得所有 `p1~p6` 的值，然后向 `kernel_para` 中填写 `p1~p6` 的地址

```cpp
std::vector<void*>para_value;
for(int i=0;i<6;++i){
    para_value.push_back((void*)(0x10000+i));
}
std::vector<void*>kernel_para;
for(int i=0;i<6;++i){
    kernel_para.push_back(&(para_value.at(i)));
}
```

方法二:

对 `para_value` 的存空间进行预分配

```cpp
std::vector<void*>para_value;
Para_value.reserve(6);
std::vector<void*>kernel_para;
for(int i=0;i<6;++i){
    para_value.push_back((void*)(0x10000+i));
    kernel_para.push_back(&(para_value.back()));
}
```

完整的测试程序如下：

测试环境为 `ubuntu 16.04 + cuda 9.0`

直接运行 `k.sh` 生成两个可执行程序 `k1` 和 `k2` ， `k1` 为错误的程序， `k2` 为正确的程序。

```bash
#!/bin/bash
# k.sh
nvcc --gpu-code=sm_61 -arch compute_61 -ptx k.cu -o k.ptx
clang++ k1.cpp -o k1 -std=c++11 -I/usr/local/cuda/include -L/usr/local/lib64 -lcuda
clang++ k2.cpp -o k2 -std=c++11 -I/usr/local/cuda/include -L/usr/local/lib64 -lcuda
```

`k.cu`
```cpp
#include <stdio.h>
// k.cu
extern "C" __global__ void ktest(void *p1, void*p2, void* p3, void*p4, void*p5, void*p6){
	printf("address of p1 = %x\n",p1);
	printf("address of p2 = %x\n",p2);
	printf("address of p3 = %x\n",p3);
	printf("address of p4 = %x\n",p4);
	printf("address of p5 = %x\n",p5);
	printf("address of p6 = %x\n",p6);
	return ;
}
//int main()
//{
//	k1<<<1,1>>>(nullptr,nullptr,nullptr,nullptr,nullptr,nullptr);
//	cudaDeviceSynchronize();
//	return 0;
//}
```

`k1.cpp`
```cpp
#include <cuda.h>
#include <cuda_runtime.h>
#include <iostream>
#include <vector>
// k1.cpp
int main()
{
	auto err = cuInit(0);
	if(err) return -1;
	CUdevice device;
	CUcontext context;
	err = cuDeviceGet(&device,0);
	if(err) return -2;
	err = cuCtxCreate(&context,CU_CTX_SCHED_BLOCKING_SYNC,device);
	if(err) return -3;
	std::cout << "cuda initial success" << std::endl;
	CUmodule module;
	err = cuModuleLoad(&module,"k.ptx");
	if(err) return -4;
	std::cout << "cuda load module success" << std::endl;
	CUfunction function;
	err = cuModuleGetFunction(&function,module,"ktest");
	if(err)return -5;
	std::cout << "cuda get function success" << std::endl;
	std::vector<void*>para_value;
	std::vector<void*>kernel_para;
	for(int i=0;i<6;++i){
		para_value.push_back((void*)(0x10000+i));
		kernel_para.push_back(&(para_value.back()));
	}
	err=cuLaunchKernel(function,
			   1,1,1,
			   1,1,1,
			   0,nullptr,
			   &(kernel_para.at(0)),
			   nullptr);
	if(err) return -6;
	std::cout << "cuda launch kernel success" << std::endl;
	err = cuCtxSynchronize();
	if(err) return -7;
	std::cout << "cuda context synchronize sucess" << std::endl;	
	return 0;	
}
```

`k2.cpp`
```cpp
#include <cuda.h>
#include <cuda_runtime.h>
#include <iostream>
#include <vector>
//k2.cpp
int main()
{
	auto err = cuInit(0);
	if(err) return -1;
	CUdevice device;
	CUcontext context;
	err = cuDeviceGet(&device,0);
	if(err) return -2;
	err = cuCtxCreate(&context,CU_CTX_SCHED_BLOCKING_SYNC,device);
	if(err) return -3;
	std::cout << "cuda initial success" << std::endl;
	CUmodule module;
	err = cuModuleLoad(&module,"k.ptx");
	if(err) return -4;
	std::cout << "cuda load module success" << std::endl;
	CUfunction function;
	err = cuModuleGetFunction(&function,module,"ktest");
	if(err)return -5;
	std::cout << "cuda get function success" << std::endl;
	std::vector<void*>para_value;
	para_value.reserve(6);
	std::vector<void*>kernel_para;
	for(int i=0;i<6;++i){
		para_value.push_back((void*)(0x10000+i));
		kernel_para.push_back(&(para_value.back()));
	}
	err=cuLaunchKernel(function,
			   1,1,1,
			   1,1,1,
			   0,nullptr,
			   &(kernel_para.at(0)),
			   nullptr);
	if(err) return -6;
	std::cout << "cuda launch kernel success" << std::endl;
	err = cuCtxSynchronize();
	if(err) return -7;
	std::cout << "cuda context synchronize sucess" << std::endl;	
	return 0;	
}
```
