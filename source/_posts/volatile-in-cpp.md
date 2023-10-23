---
title: volatile in c/cpp
date: 2023/10/23 15:59:32
---
## volatile的行为
对于c/cpp中的volatile关键字，标准中有如下规定：

[https://vinci-897.github.io/cppwp/n4868/dcl.type.cv#6](https://vinci-897.github.io/cppwp/n4868/dcl.type.cv#6)

<img width="796" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/feb09c42-919e-44dc-a288-b4d7faa4ef7b">

[https://en.cppreference.com/w/cpp/language/cv](https://en.cppreference.com/w/cpp/language/cv)

<img width="811" alt="截屏2023-10-23 16 06 35" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/ceb6a64c-e608-4b88-afd3-f91824a5d225">

从上述规定中可以看出，volatile关键字的存在主要是避免编译器对这一变量进行有侵略性的优化，volatile变量的访问1不能够被优化掉，2也不能够被和其他volatile变量的访问之间变换顺序。
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
我们知道，除了在将c/cpp语言编译成机器语言代码时，会被编译器改变其执行顺序之外，cpu在执行时还会对指令进行乱序执行，
然而，cpu在访问内存时还会使用cache，多核之间的cache可能会不同步，导致不同线程看到的变量修改顺序不同，虽然缓存一致性协议保证了多核缓存的同步，但是由于大多数cpu并没有实现完美的缓存一致性协议，而是保留了自己的内存模型，并且存在store buffer一类的延迟写入策略([store buffer问题的相关介绍](https://zhuanlan.zhihu.com/p/546562532))，因此缓存不同步依然会导致多个线程在访问内存上的同一地址时得到各自缓存中不同的值。

但volatile不能用于帮助多线程看到共享变量的修改，首先volatile不保证多写的原子性，那么我们不考虑多写，只考虑一读一写。

