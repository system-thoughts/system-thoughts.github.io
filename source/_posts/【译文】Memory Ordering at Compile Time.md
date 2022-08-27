---
title: 【译文】Memory Ordering at Compile Time
date: 2021-03-22T19:21:52+08:00
tags: [lock-free, memory-reordering]
categories: [translation, lock-free]
---

我们编写的C/C++代码在处理器上实际的执行顺序和其在源码中的顺序可能并不相同。编译器会通过指令重排优化编译效果，CPU也会通过乱序执行优化执行效率。本文将介绍编译时的内存重排。

<!-- more -->

------

> 原文链接：https://preshing.com/20120625/memory-ordering-at-compile-time/

您写的C/C++源码与其实际在CPU上执行的内存操作顺序可能不同，内存操作可能会根据某些规则重新排序。为了让代码运行得更快，编译器(在编译时)和处理器(在运行时)都会对内存顺序进行更改。
编译器开发人员和CPU供应商普遍遵循的内存重排的基本规则可以表述为：
> Thou shalt not modify the behavior of a single-threaded program.

依据单线程程序行为不可修改的原则，程序员在写单线程的代码时基本不会注意到内存重排。在多线程编程中，它也经常被忽略，因为互斥(mutexes)、信号量(semaphores)和事件(events)都是为了防止在它们的调用位置附近进行内存重排而设计的。只有在使用无锁技术时（线程之间共享内存而没有任何相互排斥的情况），内存重排的效果才能够被明显地{% post_link 当场抓获内存重排 '观察到' %}。

注意，在为多核平台编写无锁代码时，有些情况可以避免内存重排的麻烦，正如我在introduction to lock-free programming中提到的那样，可以利用顺序一致(sequentially consistent)的类型，例如Java的`volatile`变量或C ++ 11原子变量，这可能会牺牲一点性能。本文我不会细究这些情况，我将重点讨论编译器对常规、非顺序一致性类型的内存排序的影响。

## Compiler Instruction Reordering
如您所知，编译器的工作是将人类可读的源代码转换为CPU可读的机器代码。在此转换过程中，编译器有很大的自由度。
![](compiler-reordering.png)
编译器在保证单线程行为不变的情况下有指令重排的自由。指令重排通常在启用编译器优化时发生，考虑以下函数：
```c
int A, B;

void foo()
{
    A = B + 1;
    B = 0;
}
```
使用GCC 4.6.1在没有编译优化的情况下，会生成如下机器代码：
```asm
$ gcc -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR _B  (redo this at home...)
        add     eax, 1
        mov     DWORD PTR _A, eax
        mov     DWORD PTR _B, 0
        ...
```
对全局变量B的内存存储发生在对A的内存存储之后，和源码中一样。
`-O2`编译优化之后的机器代码如下：
```asm
$ gcc -O2 -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR B
        mov     DWORD PTR B, 0
        add     eax, 1
        mov     DWORD PTR A, eax
        ...
```
这一次，编译器将全局变量B的存储放到全局变量A的存储之前。这并未打破内存排序的基本规则，单线程永远感知不到差别。

然而，在编写无锁代码时，此类编译器重新排序可能会导致问题。下面的代码是一个经常被引用的示例，全局变量`IsPublished`用于指示全局变量`Value`的修改已经完成：
```c
int Value;
int IsPublished = 0;
 
void sendValue(int x)
{
    Value = x;
    IsPublished = 1;
}
```
想象一下，如果编译器将`IsPublished`的内存写重排到`Value`内存写之前，会发生什么？即使是在单处理器系统上，也会遇到问题:一个线程的两次内存写操作可能会被OS抢占，让其他线程相信`Value`已经更新，而实际却没有。
当然，编译器可能未对这些操作进行重新排序，生成的机器代码作为无锁操作可以在任何强内存模型(strong memory model)的多核CPU（例如x86/64）上或在单处理器环境中正常运行。如果没有发生内存重排，我们会很幸运。然而，更好的做法是认识到共享变量存在内存重新排序的可能性，并确保正确的执行顺序。

## Explicit Compiler Barriers
防止编译器内存重排最简单的方法是使用编译器屏障(compiler barrier)指令。我们在{% post_link 当场抓获内存重排 '上一篇文章中' %}已经提及编译器屏障。下面是GCC的完全编译器屏障(full compiler barrier)。Microsoft C++的[_ReadWriteBarrier](https://docs.microsoft.com/en-us/cpp/intrinsics/readwritebarrier?redirectedfrom=MSDN&view=msvc-160)具有相同功能。
```c
int A, B;

void foo()
{
    A = B + 1;
    asm volatile("" ::: "memory");
    B = 0;
}
```
加上完全编译器屏障并保持编译器优化，观察内存存储指令并未重排：
```asm
$ gcc -O2 -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR _B
        add     eax, 1
        mov     DWORD PTR _A, eax
        mov     DWORD PTR _B, 0
        ...
```
同样，如果我们要保证`sendMessage`示例在单处理器系统数正常工作，那么我们至少必须在此处引入编译器屏障。不仅发送操作需要编译器屏障防止内存写操作重排，接收方也需要在内存读操作之间加入屏障。
```c
#define COMPILER_BARRIER() asm volatile("" ::: "memory")

int Value;
int IsPublished = 0;

void sendValue(int x)
{
    Value = x;
    COMPILER_BARRIER();          // prevent reordering of stores
    IsPublished = 1;
}

int tryRecvValue()
{
    if (IsPublished)
    {
        COMPILER_BARRIER();      // prevent reordering of loads
        return Value;
    }
    return -1;  // or some other value to mean not yet received
}
```
如前所述，编译器屏障足以防止单处理器系统上的内存重排。如今在多核计算已经成为常态。如果我们想要确保我们的指令交互在任何架构的多处理器环境中都以期望的顺序发生，那么编译器屏障是不够的。我们需要发送`CPU fence`指令，或执行任何在运行时充当内存屏障的操作。我将在下一篇文章中会更多地介绍这块内容，内存屏障就像源代码版本控制操作一样。

Linux内核以宏的形式提供了多个CPU fence指令，例如`smb_rmb`。这些宏在单处理器系统中会被编译成简单的编译器屏障。

## Implied Compiler Barriers
还有其他方法防止编译器指令重排。刚刚提及的`CPU fence`指令也可以充当编译器屏障。这是PowerPC的`CPU fence`指令，在GCC中定义为宏：
```asm
#define RELEASE_FENCE() asm volatile("lwsync" ::: "memory")
```
在代码中加入`RELEASE_FENCE`不仅可以防止某些类型的处理器重排，而且能够确保阻止编译器重排。在`sendValue`函数中使用`RELEASE_FENCE`可以确保其在多处理器环境中安全。
```c
void sendValue(int x)
{
    Value = x;
    RELEASE_FENCE();
    IsPublished = 1;
}
```
在C++11原子标准库中，每个non-releaxed的原子操作都可以作为编译器屏障：
```c++
int Value;
std::atomic<int> IsPublished(0);

void sendValue(int x)
{
    Value = x;
    // <-- reordering is prevented here!
    IsPublished.store(1, std::memory_order_release);
}
```
如您所料，每个包含编译器屏障的函数(即使是内联函数)也可充当编译器屏障。（但是，[Microsoft文档](https://docs.microsoft.com/en-us/cpp/intrinsics/readwritebarrier?redirectedfrom=MSDN&view=msvc-160)表明，早期版本的Visual C++编译器中可能不是这种情况。Tsk，tsk！）
```c
void doSomeStuff(Foo* foo)
{
    foo->bar = 5;
    sendValue(123);       // prevents reordering of neighboring assignments
    foo->bar2 = foo->bar;
}
```
事实上，不论函数中是否包含编译器屏障指令，大多数函数调用可以作为编译器屏障。也有函数调用不会作为编译器屏障：内联函数、使用[pure属性](https://lwn.net/Articles/285332/)声明的函数以及链接时生成的代码。除此之外，对外部函数的调用甚至比编译器屏障更为强大，因为编译器不知道外部函数的副作用。
仔细想想，还是很有道理。假设上面代码片段中`sendValue`的实现在外部库中。编译器如何知道`sendValue`不依赖于`foo->bar`的值?它如何知道`sendValue`将不会修改`foo->bar`的内存?编译器无法做出此类假设，因此，为了遵守内存排序的基本规则，它不能对`sendValue`周围的任何内存操作重排。同样，即便开启了编译优化，`sendValue`函数调用完成之后，不能假设`foo->bar`的值仍然为5，需要从内存中读取`foo->bar`的值。
```asm
$ gcc -O2 -S -masm=intel dosomestuff.c
$ cat dosomestuff.s
        ...
        mov    ebx, DWORD PTR [esp+32]
        mov    DWORD PTR [ebx], 5            // Store 5 to foo->bar
        mov    DWORD PTR [esp], 123
        call    sendValue                     // Call sendValue
        mov    eax, DWORD PTR [ebx]          // Load fresh value from foo->bar
        mov    DWORD PTR [ebx+4], eax
        ...
```
如您所见，在许多情况下，编译器指令重排是被禁止的，甚至编译器需要从内存重新加载某些值。我想，正是因为这些隐藏规则导致人们长期以来一直认为`volatile`数据类型在正确编写的多线程代码中不是必需的。

## Out-Of-Thin-Air Stores
指令重排会让无锁编程变得棘手吗?在c++ 11标准化之前，技术上，没有任何规则可以阻止编译器使用更糟糕的技巧。特别是，在以前没有对共享内存写的情况下，编译器可以自由地引入共享内存写操作。下面这个例子[在Hans Boehm在多篇文章中](https://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf)均有提及：
```c
int A, B;

void foo()
{
    if (A)
        B++;
}
```
尽管实际上不太可能写这样的代码，但是没有什么可以阻止编译器在检查A之前，将B提升到寄存器，从而会有以下的等效内容：
```c
void foo()
{
    register int r = B;    // Promote B to a register before checking A.
    if (A)
        r++;
    B = r;          // Surprise! A new memory store where there previously was none.
}
```
上述代码仍然遵守了最基本的内存排序规则。单线程应用程序对“将B提升到寄存器”毫无感知。但是在多线程环境中，`foo`函数会清除在其他线程中所做的任何修改--即便是A为0的情况下。`foo`函数的本意并非如此。尽管我们数十年来一直在使用C/++写多线程和无锁代码，因为这种晦涩、技术上的不可能性让人们一直说C++不支持线程。

我不知道是否有人在实践中成为这种“out-of-thin-air”的受害者。或许是我们通常编写的无锁代码类型没有这样的优化模式。如果我遇到了这种编译器转换，我想我得好好调教下我的编译器。
无论如何，在会引入数据竞争的情况下，新的C++11标准明确禁止编译器的此类行为。[C++11工作草案](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)的§1.10.22中可以找到：
> Compiler transformations that introduce assignments to a potentially shared memory location that would not be modified by the abstract machine are generally precluded by this standard.

## Why Compiler Reordering?
正如我在文章开头提到的，编译器修改内存交互顺序的原因与处理器进行性能优化的原因相同。这种优化是现代CPU复杂性的直接结果。
我有个大胆的怀疑：在80年代早期，当cpu最多只有[几十万个晶体管](https://en.wikipedia.org/wiki/Microprocessor_chronology)的时候，编译器做了大量的指令重排。但从那以后，摩尔定律为CPU设计者提供了大约10000倍数量的晶体管，而这些晶体管被花费在诸如流水线、内存预取、ILP以及最近的多核等技巧上。由于这些特性，我们已经看到某些架构中程序指令的顺序会对性能产生显著的影响。
1993年Intel发布的第一款奔腾处理器，带有所谓的U和v管道，是我记得的第一个[流水线和指令顺序的重要性的处理器](https://www.agner.org/optimize/microarchitecture.pdf)。然而，最近，我在Visual Studio中执行x86反汇编时，我惊讶地发现指令的重排是如此之少。另一方面，我在Playstation 3上进行SPU拆解的时候，发现编译器真的很厉害。这些只是轶事。它可能无法反映其他人的经验，也不应该影响我们在无锁代码中强制执行内存排序的方式。