---
title: memory order
date: 2023/10/25 15:58:32
disqusId: memory-order
---
根据上一篇文章[https://vinci-897.github.io/volatile-in-cpp/](https://vinci-897.github.io/volatile-in-cpp/)中memory order部分提到的内容，我们知道，不同的cpu有不同的memory model和memory order，这导致cpu会对代码进行重排序，虽然单线程上编译器和cpu对代码的重排序都不会导致最终结果的不同，但当代码在一个cpu核心上执行时，由于多核缓存不同步等问题，其他的cpu核心会看到与原有代码顺序不同的访存顺序，这就是我们所说的乱序执行，这会引起一些问题。

编译期重排序已经在上一篇文章中描述过，因此本文中，我们所说的都是cpu内存模型导致的指令重排序。**并且，编译器重排是从单线程角度考虑的，重排时会保证单线程逻辑不变，而cpu乱序执行，对于单核cpu内部看来，访存顺序也是不会改变的，仅仅是对其他cpu来说顺序改变**。

<!--more-->

有些时候我们会思考，存在分支情况下是否会重排，比如`if (a == 1) b = 2;`里，store b是否会被排序到load a前面？由于分支预测功能和流水线的存在，答案是会的，但如果后续发现a!=1，if分支最终没有进入，那么这次store b是无效的，cpu会采用另一分支或直接导致流水线暂停。
## 内存屏障指令
在x86架构中，Load、Load， Load、Store， Store、Store这三种是不允许发生重排的（我们说的重排不一定是真正的重排，而是在其他核心看来访存行为的顺序变化），只有Store、Load这种被允许重排成Load、Store（即Stores after loads）。x86只允许store load指令的顺序交换，但这仅限于交换不同变量的store和load，因为对于同一变量的store load序列，这种交换显然是错误的，cpu很明白不能交换同一个变量的store load指令序列。

[https://stackoverflow.com/questions/20316124/does-it-make-any-sense-to-use-the-lfence-instruction-on-x86-x86-64-processors](https://stackoverflow.com/questions/20316124/does-it-make-any-sense-to-use-the-lfence-instruction-on-x86-x86-64-processors)

<img width="682" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/908b97ee-a695-4496-a8b0-ed97fd3508dc">

```
void procedure0() {
    flag[0] = true;
    turn = 1; //1
    while (flag[1] && (turn == 1));//2
    gCount++;
    flag[0] = false;
}

void procedure1() {
    flag[1] = true;
    turn = 0;//3
    while (flag[0] && (turn == 0));//4
    gCount++;
    flag[1] = false;
}
```

上述代码是著名的peterson算法，用于控制两个线程互斥访问临界区，这个算法在不允许任何类型的重排序时是完美的，然而，这个算法也是x86 cpu可能出现异常情况的标志性案例，因为在同一个函数中，x86 cpu会颠倒store flag[0]和之后while中load flag[1]的顺序，当两个线程都不在临界区时，两个flag都为false，此时两个while的flag都被提前load进寄存器，在之后while判断时，第一个条件都必然被满足，这样会导致flag失去作用，而turn一个变量是不能保证互斥的，按照代码注释中1234的顺序执行，两个线程会同时进入临界区。

<img width="573" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/f366a564-6b9d-4d44-b8ab-15ab7ebf5f6e">

上面这张图来自intel的开发手册，可以看出，mfence前面和后面的load store都不能越过mfence，sfence前面和后面的store不能越过sfence，lfence比较特殊，他的效果在前后不是对称的，lfence前面的load不能到后面，lfence后面的load store不能到前面。

mfence的功能是，在后面的load store全局可见之前，前面的load store都要全局可见；sfence的功能是后面的store全局可见之前，前面的store都要全局可见；lfence的功能是后面的load store全局可见之间，前面的load都要全局可见。

比如说，如果有一条mfence指令，那么不管cpu0的硬件如何安排指令顺序，其他cpu都可以安全地认为，cpu0是先进行了前面的load store再进行了后面的load store。

<img width="591" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/3e17754f-64a5-48d1-bc7e-be76bdc60334">

有时我们会觉得，load load乱序没有影响，请看上图，按照正常的顺序r，2应该等于NEW。然而，如果C2的L2被放到了L1前面，发生load load乱序，且C1C2按照L2、S1、S2，L1、B1的顺序执行，那么r2会被赋值为0，这是不对的。

四种乱序都会对这个程序产生影响，具体内容引自[https://aijishu.com/a/1060000000222715](https://aijishu.com/a/1060000000222715)

注意，即使如此，mfence也并不等于sfence + lfence，仔细思考一下，会发现sfence + lfence时，无法阻止store load乱序，如果，lfence在sfence前，那么前面的store可以被变换到lfence和sfence之间，后面的load也可以到lfence和sfence之间，所以store仍然有可能跑到load后面；如果sfence在lfence之前，那么是可以阻止store load乱序的，但mfence只有一条指令并且可以阻止任何形式的乱序，因此我们可以发现，sfence + lfence != mfence。

**我们所说的乱序/顺序改变都是指多核场景下在cpu1看来，cpu0的访存指令执行顺序与代码顺序不同**，既然这样，我们之前说到x86根本不会出现除store load的乱序，那么也就是说，在多线程缓存/内存一致性这个问题上，lfence和sfence根本没有意义，他们两个所能阻止的乱序在x86上压根就不会发生。（x86在目前被认为是使用了tso内存模型，这种模型比较严格）

是的，lfence和sfence在多线程上没有意义，但这不代表他没有任何作用，实际上这两种指令在硬件上会做一些操作，且确确实实的能阻止某些load store指令的全局可见性刷新，对性能优化会起到一定作用。[https://www.zhihu.com/question/29465982](https://www.zhihu.com/question/29465982)

arm架构的内存模型十分复杂，且没有这么多限制，会出现更多的乱序情况。



