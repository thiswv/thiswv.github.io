---
title: "Arena源码分析"
date: 2025-03-02
tags: leveldb
---

![Arena_class_](assets\images\Arena_class_.png)

讲个题外话，这用draw.io画出来的图，导出来后怎么这么糊涂啊，把缩放放大后确实清晰，但也很占地方啊！！



Arena的设计，在leveldb/util 下的arena.cc和arena.h中

上图中，我们可以看到，Arena类的设计。



## 1. 私有变量的含义

我们先讲解其中三个私有变量的含义，先暂时不讲memory_usage_。

```c++
char* alloc_ptr_;
char* alloc_bytes_remaining_;
vector<char*> blocks_;
```

![Arena_variable_](assets\images\Arena_variable.png)



其中blocks\_为vector类型，存储char*的指针，指向内存池中固定大小的内存块——block

在leveldb中block的大小被设置为4096B，4KB

alloc_ptr\_的含义是，某一个block中剩余空间的起始地址。alloc_bytes_remaining\_是指，当前的block还有多少剩余空间。



```c++
Arena::Arena()
	: alloc_ptr_(nullptr), alloc_bytes_remaining_(0), memory_usage_(0) {}

Arena::~Arena() {
    for (int i = 0; i < blocks_.size(); i ++) {
		delete[] blocks_[i];
    }
}
```



从析构函数，我们可以看到，内存只有在程序退出时才一次性释放。



然后我们一次了解Arena中其他几个函数。



## 2. 细看函数

### 2.1 Allocate(size_t bytes)

首先回答，为什么用size_t，这个主要是保证跨平台的使用

不同机器，32位，64位不同。对应的int位数不同，用size_t可以保证良好的跨平台性。

```c++
inline char* Arena::Allocate(size_t bytes) {
	assert(bytes > 0);
	if (bytes <= alloc_bytes_remaining_) {
		char* result = alloc_ptr_;
		alloc_bytes_remaining_ -= bytes;
		return result;
	}
	
	return AllocateFallback(bytes);
}
```



先不看AllocateFallback，我们只看Allocate

我们不考虑0字节分配，必须都大于等于0字节

如果需要的内存**空间小于还拥有的剩余空间，就正常分配**，否则调用AllocateFallback

### 2.2 AllocateFallback(size_t bytes)

```c++
char* Arena::AllocateFallback(size_t bytes) {
	if (bytes > KBlockSize / 4) {
		char* result = AllocateNewBlock(bytes);
		return result;
	}
	
	alloc_ptr_ = AllocateNewBlock(KBlockSize);
	alloc_bytes_remaining_ = KBlockSize;
	
	char* result = alloc_ptr_;
	alloc_ptr_ += bytes;
	alloc_bytes_remaining_ -= bytes;
	return result;
}
```

四分之一是个策略，目的是减少内存的浪费。

当前剩余空间不够且大于默认block（4KB）的四分之一，直接用**需要的大小新开**一片空间。

否则就开个默认4KB的大小，来进行分配。



### 2.3 AllocateNewBlock(size_t block_bytes)

```c++
char* Arena::AllocateNewBlock(size_t block_bytes) {
	char* result = new char[block_bytes];
    blocks_.push_back(result);
    memory_usage_.fetch_add(block_bytes + sizeof(char*), 
                            std::memory_order_relaxed);
    
    return result;
}
```

std::memory_order_relaxed保证fetch_add操作的原子性。

一个从堆申请内存的函数，sizeof(char*)是额外需要的指针内存空间。



### 2.4 AllocateAligend(size_t bytes) 

```c++
char* Arena::AllocateAligned(size_t bytes) {
	const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
    //保证align是2的幂次数，不是的话，会报错
    //static_assert 编译阶段的断言
    static_assert((align & (align - 1)) == 0,
                  "Pointer size should be a power of 2");
    
    // 计算当前 alloc_ptr_ 地址是否对齐
    size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
    size_t slop = (current_mod == 0 ? 0 : align - current_mod);   //没对齐的话，计算出需要的额外填充数
    //总需求 = 申请的大小 + 填充对齐大小
    size_t needed = bytes + slop;
    char* result;
    
    //当前剩余空间够用
    if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
    } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
    }
    
    //确保最终返回的是对齐的
    //reinterpret_cast 把指针转化成整数类型，方便进行位运算
    //uintptr 无符号整数类型 与 void*大小相同（4字节或8字节）
    assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
    return result;
	
}
```



这段内存对齐的分配是最难懂得。

sizeof(void*)是保证通用指针对齐。align代表对齐数，最小值为8，是为了通用性。

32b -> sizeof(void*)  == 4    4B

64b -> sizeof(void*)  == 8    8B



## 3. 疑问，解答

我在看源码的时候，就有疑问了，为什么leveldb不直接在开始时就new超级大一片空间呢！我直接new个1G空间，你要多少，我直接顺序给你呗，那多爽啊！

但深入了解，弊端很多。反而，这样需要一次默认4KB，超过四分之一，直接给你需要的空间确实有优势。

1. 一次分配超大的空间，容易造成浪费。我明明只需要10KB，你直接来1G，浪费太多。而且要是程序不退出，一直占着内存。
2. 多个Arena对象，每一个都先来个1G，那空间也不够用。
3. 可能一般的计算机内存，没有超大的连续内存给你用啊！
