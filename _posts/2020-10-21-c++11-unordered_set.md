---
title:  "C++11 使用自定义 hash 函数及比较函数的 unordered_set"
author: adream307
date:   2020-10-21 13:23:00 +0800
categories: [c++11]
tags: [c++11, unordered_set]
---

```c++
#include <unordered_set>
#include <functional>
#include <iostream>

struct MyKey
{
	int key;
};

struct MyKeyHashHasher
{
	size_t operator()(const MyKey &k) const noexcept
	{
		return std::hash<int>{}(k.key);
	}
};

struct MyKeyHashComparator
{
	bool operator()(const MyKey &k1, const MyKey &k2) const noexcept
	{
		return k1.key == k2.key;
	}
};

int main()
{
	std::unordered_set<MyKey,MyKeyHashHasher,MyKeyHashComparator> ss;
	return 0;
}
```