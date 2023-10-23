---
title: volatile in c/cpp
date: 2023/10/23 15:59:32
---
## volatile的行为
对于c/cpp中的volatile关键字，标准中有如下规定：

[https://vinci-897.github.io/cppwp/n4868/dcl.type.cv#6](https://vinci-897.github.io/cppwp/n4868/dcl.type.cv#6)

<img width="796" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/feb09c42-919e-44dc-a288-b4d7faa4ef7b">

<!--more-->

[https://en.cppreference.com/w/cpp/language/cv](https://en.cppreference.com/w/cpp/language/cv)

<img width="811" alt="截屏2023-10-23 16 06 35" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/ceb6a64c-e608-4b88-afd3-f91824a5d225">

从上述规定中可以看出，volatile关键字的存在主要是避免编译器对这一变量进行有侵略性的优化，**volatile变量的访问1不能够被优化掉，2也不能够被和其他volatile变量的访问之间变换顺序**。
个人认为，前者主要是指以下情况：
```
int x = 1;
cout << x;
```
在这种情况中，x可能被外部状态改变，如MMIO中硬件状态改变、信号处理函数改变、可能被汇编实现的setjmp/longjmp改变等，但编译器在优化时只考虑作为单线程运行的流程，因此可能会让cout直接输出1。（也可能是被其他线程改变，但是本文讨论的volatile不能阻止多线程的问题）
```
int x = 0;
while(x) {
  cout << 123;
}
```
同样的原因，编译器可能会直接删除掉while循环，或者在while中只使用寄存器中存储的x值。
```
int x = 1;
void sig_handler(int signum) {
  x = 0;
}
int main() {
  while(x) {
    cout << 123;
  }
}
```
代码可能会被编译成一个while(true)的死循环，或者在while中只使用寄存器中存储的x值，编译器不会考虑x被内存中的signal_handler修改的情况，所以即使信号被触发，也不会跳出循环。
因此，volatile可以被用于以下几种情况：与硬件进行MMIO交互/信号处理函数修改/内联汇编，可以发现，正如标准中所说的那样，volatile可以阻止编译器的这种行为，告知编译器这一变量可能被本部分代码流程之外的行为修改。
## volatile不应该被用于多线程共享变量
在上述的内容中，我们提到，其他线程的修改也会导致编译器的优化变得不再正确，那么我们能否通过使用volatile变量解决多线程之间变量修改的同步呢？。
我们知道，除了在将c/cpp语言编译成机器语言代码时，会被编译器改变其执行顺序之外，cpu在执行时还会对指令进行乱序执行，这种优化在单线程时会保证访存结果与乱序前相同，然而对于多核来说，可能不再正确。
同时，刚刚说到，cpu在访问内存时还会使用cache，多核之间的cache可能会不同步，导致不同线程看到的变量修改顺序不同，虽然缓存一致性协议能够保证多核缓存的同步，但是由于大多数cpu并没有实现完美的缓存一致性协议，或者他们实现的一致性并不保证在任何时刻的多核缓存一致，而是保留了自己的内存模型（因此他们也提供了内存屏障指令），并且存在store buffer一类的延迟写入策略([store buffer问题的相关介绍](https://zhuanlan.zhihu.com/p/546562532))，因此缓存不同步依然会导致多个线程在访问内存上的同一地址时得到各自缓存中不同的值。
这几个问题导致我们即使阻止了编译器优化对操作顺序的影响，也不能阻止cpu乱序执行和cache的影响（这两个问题需要内存屏障来解决），然而volatile只能解决编译器的问题，volatile不会被编译为内存屏障指令，因此volatile无法解决多线程之间变量的同步问题。
## 内存屏障
我们之前提到了乱序执行和不同架构内存模型导致的cache不同步问题，x86/x64/arm架构都提供了各自的内存屏障指令。

![image](https://github.com/vinci-897/vinci-897.github.io/assets/55838224/34055c24-918b-4394-8fdc-af6a84c71a34)

在MESI一致性协议中，可以由多个cpu同时持有一个数据的缓存，也可以由一个cpu持有一个数据的缓存，但是如果一个cpu0想要修改一个地址，那么必须先要求所有持有这一地址的cpu的缓存失效。如果一个cpu0想要读取一个地址，同时这个地址刚刚被cpu1修改过，那么应该由cpu1的缓存返回这一地址的值，而不是由内存返回。引自[https://zhuanlan.zhihu.com/p/546562532](https://zhuanlan.zhihu.com/p/546562532)

在cpu中存在上图中的结构，store buffer和invalid queue，store buffer的作用是将缓存同步的任务分离出cpu，以防让cpu在缓存同步时等待，当cpu0要修改一个内存地址时，先把数据放入store buffer，然后通知cpu1让缓存失效，当cpu1反馈完成消息后，cpu0的store buffer会真正的修改cache，在这期间cpu0会直接使用自己store buffer中的新值，（但这里可以很容易的想到，其他cpu还没有来得及让缓存失效时如果使用了这一地址，那么我们的缓存就出现了不一致）。invalid queue的作用是当cpu1收到缓存失效消息时，不必真正的等待缓存行被设置成失效，而是先存入队列等待慢慢消费，直接给cpu0返回成功（很容易想到在invalid queue来不及消费时，如果访问了这个地址，也会导致缓存不同步）。



