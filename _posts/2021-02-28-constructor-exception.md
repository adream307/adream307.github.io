---
title:  "c++ 构造函数抛异常"
author: adream307
date:   2021-02-28 10:56:00 +0800
categories: [c++11]
tags: [c++11]
---

如果构造函数抛异常，则析构函数不会被调用，已经初始化了成员变量则被系统依次释放，此处可能引起内存泄露

```cpp
#include <iostream>

class T2{
public:
    T2(){std::cout << "T2()" << std::endl;}
    ~T2(){std::cout << "~T2()" << std::endl;}
};

class T1{
public:
    T1(){
        a = new int[16];
        std::cout << "T1()" << std::endl;
        throw 1;
    }
    ~T1(){
        delete [] a;
        std::cout << "~T1()" << std::endl;
    }
private:
    int *a;
    T2 t2;
};

int main(){
    try{
        T1 t1;
    }catch(...){
        std::cout << "catch exception" << std::endl;
    }
    return 0;
}
```

以上测试程序输出
```txt
T2()
T1()
~T2()
catch exception
```

使用 `valgrind` 检测内存泄露
```bash
valgrind --leak-check=yes ./t 
==26198== Memcheck, a memory error detector
==26198== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==26198== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==26198== Command: ./t
==26198== 
T2()
T1()
~T2()
catch exception
==26198== 
==26198== HEAP SUMMARY:
==26198==     in use at exit: 64 bytes in 1 blocks
==26198==   total heap usage: 4 allocs, 3 frees, 73,924 bytes allocated
==26198== 
==26198== 64 bytes in 1 blocks are definitely lost in loss record 1 of 1
==26198==    at 0x4C3089F: operator new[](unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==26198==    by 0x108DBA: T1::T1() (in /home/abs/tmp/mem-leak/t)
==26198==    by 0x108C4D: main (in /home/abs/tmp/mem-leak/t)
==26198== 
==26198== LEAK SUMMARY:
==26198==    definitely lost: 64 bytes in 1 blocks
==26198==    indirectly lost: 0 bytes in 0 blocks
==26198==      possibly lost: 0 bytes in 0 blocks
==26198==    still reachable: 0 bytes in 0 blocks
==26198==         suppressed: 0 bytes in 0 blocks
==26198== 
==26198== For counts of detected and suppressed errors, rerun with: -v
==26198== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```


