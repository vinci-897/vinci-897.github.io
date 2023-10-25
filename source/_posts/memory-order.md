---
title: memory order
date: 2023/10/25 15:58:32
disqusId: memory-order
---
根据上一篇文章[https://vinci-897.github.io/volatile-in-cpp/](https://vinci-897.github.io/volatile-in-cpp/)中memory order部分提到的内容，我们知道，不同的cpu有不同的memory model和memory order，这导致代码在一个cpu核心上执行时，对于其他的cpu核心来说，会看到与原有顺序不同的访存顺序。
