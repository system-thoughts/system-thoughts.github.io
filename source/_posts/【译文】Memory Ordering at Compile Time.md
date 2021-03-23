---
title: 【译文】Memory Ordering at Compile Time
date: 2021-03-22T19:21:52+08:00
tags: [lock-free, memory-reordering]
categories:
    - translation
    - lock-free
---

您写的C/C++源码与其实际在CPU上执行的内存操作顺序可能不同，内存操作可能会根据某些规则重新排序。为了让代码运行得更快，编译器(在编译时)和处理器(在运行时)都会对内存顺序进行更改。
编译器开发人员和CPU供应商普遍遵循的内存重排的基本规则可以表述为：
> Thou shalt not modify the behavior of a single-threaded program.

依据单线程程序行为不可修改的原则，程序员在写单线程的代码时基本不会注意到内存重排。在多线程编程中，它也经常被忽略，因为互斥(mutexes)、信号量(semaphores)和事件(events)都是为了防止在它们的调用位置附近进行内存重排而设计的。只有在使用无锁技术时（线程之间共享内存而没有任何相互排斥的情况），内存重排的效果才能够被明显地{% post_path 当场抓获内存重排 '观察到' %}。

注意，在为多核平台编写无锁代码时，有些情况可以避免内存重排的麻烦，正如我在introduction to lock-free programming中提到的那样，可以利用顺序一致(sequentially consistent)的类型，例如Java的`volatile`变量或C ++ 11原子变量，这可能会牺牲一点性能。本文我不会细究这些情况，我将重点讨论编译器对常规、非顺序一致性类型的内存排序的影响。

## Compiler Instruction Reordering
如您所知，编译器的工作是将人类可读的源代码转换为CPU可读的机器代码。在此转换过程中，编译器有很大的自由度。
{% asset_img compiler-reordering.png %}
编译器在保证单线程行为不变的情况下有指令重排的自由。指令重排通常在启用编译器优化时发生，考虑以下函数：
```c
int A, B;

void foo()
{
    A = B + 1;
    B = 0;
}
```




