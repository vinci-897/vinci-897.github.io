---
title: 为什么不可以emplace_back({1, 2})?
---

# 为什么不可以emplace_back({1, 2})?
在cpp代码中，我们可以写出如下的代码：
```
vector<pair<int, int>> vec;
vec.push_back({1, 2});
```
然而，`vec.emplace_back({1, 2})`会报错，我们只能`vec.push_back(1, 2)`。

## why push_back works?

对于`void push_back( T&& value )`，由于下面这两条规则，我们可以使用一个braced-init-list作为函数的参数。

[https://timsong-cpp.github.io/cppwp/n4868/dcl.init.list#1.5](https://timsong-cpp.github.io/cppwp/n4868/dcl.init.list#1.5)

[https://timsong-cpp.github.io/cppwp/n4868/dcl.init.list#3.7](https://timsong-cpp.github.io/cppwp/n4868/dcl.init.list#3.7)

<img width="927" alt="截屏2023-10-22 22 11 37" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/9205b1f8-1428-4323-b699-ec3ae14a2de7">

由于下面这一条规则，我们最终找到了这一条构造函数

[https://timsong-cpp.github.io/cppwp/n4868/over.match#list-1.2](https://timsong-cpp.github.io/cppwp/n4868/over.match#list-1.2)
<img width="831" alt="截屏2023-10-22 22 37 40" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/5e8f98a8-5133-4f64-b50f-cc8817e53349">


<img width="265" alt="截屏2023-10-22 22 14 10" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/35fa0e73-32d3-4c31-aac0-043c645bac97">

## why emplace_back not works with a braced-list?

对于`vec.emplace_back(1, 2)`，只是一个std::forward参数转发，vector会调用allocator_trait的construct函数执行一个placement new来构造参数。

[https://en.cppreference.com/w/cpp/container/vector/emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back)

<img width="783" alt="截屏2023-10-22 22 17 33" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/53f42e32-f6fd-4893-84f4-4ad75f190b00">

而对于`vec.emplace_back({1, 2})`，由于emplace_back的函数签名是

<img width="532" alt="截屏2023-10-22 22 34 23" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/e470d969-ba62-4238-9909-30b0cbea4126">

在cpp中，由于以下的规则，不支持在函数参数中由一个大括号的参数把一个完全空白的模版参数T推断成完整的initializer_list<int>，如果把emplace_back的参数改为initializer_list<T>则可行。

<img width="817" alt="截屏2023-10-22 22 33 39" src="https://github.com/vinci-897/vinci-897.github.io/assets/55838224/506c7247-d953-4246-83aa-358637a5052c">

谜题解开了。
