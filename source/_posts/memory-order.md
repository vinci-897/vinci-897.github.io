---
title: memory order
date: 2023/10/25 15:58:32
disqusId: memory-order
---
根据上一篇文章[https://vinci-897.github.io/volatile-in-cpp/](https://vinci-897.github.io/volatile-in-cpp/)中memory order部分提到的内容，我们知道，不同的cpu有不同的memory model和memory order，这导致cpu会对代码进行重排序，虽然单线程上编译器和cpu对代码的重排序都不会导致最终结果的不同，但当代码在一个cpu核心上执行时，对于其他的cpu核心来说，会看到与原有顺序不同的访存顺序，所以在多线程中，乱序执行会引起一些问题。

编译期重排序已经在上一篇文章中描述过，因此本文中，我们所说的都是cpu内存模型导致的指令重排序。
## 内存屏障指令
在x86架构中，Load、Load， Load、Store， Store、Store这三种是不允许发生重排的，只有Store、Load这种被允许重排成Load、Store（即Stores after loads）。x86只允许store load指令的顺序交换，但这仅限于交换不同变量的store和load，因为对于同一变量的store load序列，这种交换显然是错误的，cpu很明白不能交换同一个变量的store load指令序列。

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

