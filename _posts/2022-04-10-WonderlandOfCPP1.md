---
title: WonderlandOfCPP(part1)
author: Hou Chen
date: 2022-04-10 18:32:00 +0800
categories: [C++, Basics]
tags: [c++, syntax, basics]
---

> 持续学习，持续更新

# 内存管理与智能指针

## RAII与引用计数

- ***RAII(Resource Acquisition is Initialization)***：构造时申请空间，析构时释放空间

- ***引用计数***：对于动态分配的对象进行引用计数，每增加一次对该对象的引用，引用计数+1，每删除一次引用，引用计数-1，当该对象的引用计数==0时，自动删除指向的堆内存

- 传统C++中，使用`new/delete`会导致对象的内存所有权不清晰，容易产生不销毁、多销毁的情况，引入智能指针，依赖RAII即可避免

## std::shared_ptr

- 基于引用计数的共享内存
- API
    - `use_count()`: 查看对象的引用计数
    - `get()`: 返回shared_ptr类中的内置指针，便于和C语言衔接
    - `reset()`: 减少一个引用计数
    - `std::make_shared<>()`: 避免显示调用`new`，避免代码重复且异常安全

- 第二参数自定义Deleter：当制作内存池时，用完某个内存后不交给系统释放，通过自定义的Deleter可以放回内存池
- 一个`shared_ptr`对象拥有一个**control block**，其内存放着该`shared_ptr`指向的对象和一个引用计数reference counter，对control block的访问是线程安全的，但对其内对象的访问并不是线程安全的
- 支持数组
    - C++17支持`shared_ptr<T[]>`
    - C++20支持`make_shared<T[]>`

## std::unique_ptr

- 独占内存
- unique_ptr不支持copy，但支持move

```c++
auto x = std::make_unique<int>(5);
// auto y = x;  // 非法
auro y = std::move(x);
```
- unique_ptr自定义Deleter时需要指定参数模板

## std::weak_ptr

- 防止shared_ptr循环引用而引入的弱引用智能指针，weak_ptr也指向共享内存，但不影响引用计数
- API
    - 没有*和->运算符，无法对资源进行操作
    - `expired()`：check被引用的资源是否已删除，资源未被释放返回false，资源已被释放返回true
    - `lock()`：资源未释放时把weak_ptr转为shared_ptr，资源已释放返回nullptr

# 正则表达式

- 正则表达式描述了一种字符串匹配的模式
- API
    - `std::regex_match(std::string, std::regex)`用于匹配字符串和正则表达式，匹配成功返回true，不匹配返回false
    - `std::regex_match(std::string, std::smatch, std::regex)` std::smatch便于获取匹配的结果

# 迭代器Iterator

## 插入迭代器

- `back_insert_iterator`对它所关联的容器执行`push_back()`，可缩写为`back_inserter`

- `front_insert_iterator`对它所关联的容器执行`push_front()`，可缩写为`front_inserter`

- `insert_iterator`可对它所关联的容器的指定位置插入元素，可缩写为`inserter`

## 流迭代器

- `istream_iterator`是single-pass的输入迭代器，使用`operator>>`相继读入`istringstream`对象的T类型的数据，默认自动跳过空格

    - 读操作发生在迭代器递增时，并不在解引用时

    - 默认缺省的`istream_iterator`是end-of-stream迭代器

- `ostream_iterator`是single-pass的输出迭代器，使用`operator<<`相继写basic_ostream对象的T类型数据，每次写操作后可写入分割字符串delimiter

## 反向迭代器

- `rbegin`, `rend`

## 移动迭代器

- `move_iterator`

## std::begin/end

- `std::begin(container)`和`std::end(container)`函数分别返回container的begin/end iterator，对于原始的数组也能work

```c++
#include <iostream>
#include <vector>
#include <algorithm>

template <typename T>
int CountTwos(const T& container) {
    return std::count_if(std::begin(container), std::end(container), [](int elem) {
        return elem == 2;
    });
}

int main() {
    std::vector<int> vec{1, 2, 2, 3, 4, 5};
    int arr[5] = {1, 1, 2, 9, 6};
    std::cout << CountTwos(vec) << std::endl;   // 2
    std::cout << CountTwos(arr) << std::endl;   // 1

    return 0;    
}
```

## 并行算法ExecutionPolicy

- 编译时加-O3和-ltbb

- `std::execution::seq`顺序执行

- `std::execution::par`多线程并行执行

- `std::execution::par_unseq`多线程且SIMD(single instruction multiple data)

- `std::execution::unseq`单线程中使用SIMD执行

# bind expression

- `std::bind`用于绑定函数的部分参数，待调用时传齐参数即可

## std::bind1st, std::bind2nd

- c++98中的`std::bind1st`, `std::bind2nd`初具bind思想，但功能受限

## std::bind

- f为callable object，其后是待绑定的参数列表，未绑定的参数使用std::placeholders的占位符`_1`, `_2` ...代替

```c++
    template <class F, class... Args>
    bind(F&& f, Args&&... args)
```

- 使用`std::bind`时，传入的参数是被复制的，类似于值传递，丧失了引用传递的优势；另外如果传递的参数是个指针，该指针指向一个局部变量，当跳出局部变量的作用域时，就失去了指针所指的内容，只是单纯复制了内容所在的地址

- `std::bind`可以使用`std::ref`或`std::cref`传引用，避免复制

## bind_front

- C++20引入的`bind_front`, `bind_back`是对bind的简化

# lambda expression

## 捕获capture list

- 针对lambda函数体中使用的局部自动对象进行捕获，static不用捕获，可直接使用

- 值捕获、引用捕获、混合捕获
    - 值捕获的前提是变量可以拷贝，且在lambda表达式创建时就完成了拷贝
    - 引用捕获的值是会变化的

- this捕获(C++14)

- 初始化捕获(C++14)：在构造lambda类时即完成某些初始化的逻辑，避免在函数体里多次计算，提高性能；其次，初始化允许使用任意表达式，也就是说也允许右值捕获

```c++
int func(int i) {
    return i*10;
}
auto lambda = [x = func(2)] {   // 捕获右值
    return x;
}

auto generator = [x = 0]() mutable {
    return x++;
}
auto a = generator();   // 1
auto b = generator();   // 2
auto c = generator();   // 3

auto p = std::make_unique<int>(1);
auto task1 = [=]{ *p = 5; };    // error, unique_ptr can't be captured by '='(copying)
auto task2 = [p = std::move(p)] {   // ok, p is move-constructed
    *p = 5;
}

auto x = 1;
auto f = [&r = x, x *= 10] {
    ++r;
    return r+x;
}
f();    // 12 = 2+10
```

- `*this`捕获(C++17)：this捕获存在丢失this指针指向内容的风险，所以引入`*this`

## parameter list

- C++14起允许()内使用`auto`

## 说明符

- `mutable`: 值捕获的参数也可以在lambda函数体内进行修改，本质是删去了lambda类中operator()函数的const属性

- `constexpr`: 函数可在编译or运行时调用

- `consteval`: 函数只能在编译时调用

## 模板形参(C++20)

```c++
auto lambda = []<typename T>(T val) {
    return val + 1;
}
```

## Immediately-involved function expression

```c++
auto lambda = [](int val) {
    return val + 1;
} ();
```

## 捕获时计算(C++14)

```c++
int x = 2;
int y = 7;
auto lambda = [z = x + y]() {
    return z;
}
```

## 使用auto避免复制(C++14)

## Lifting

- `std::bind`不能绑定存在重载的函数，它区分不了到底应该绑定哪个函数，但lambda能通过auto推导相应的调用版本

```c++
auto func(int x) {
    return x + 5;
}
auto func(double y) {
    return y + 5;
}

int main() {
    auto lambda = [](auto z) {
        return func(z);
    };

    lambda(1);
    lambda(2.2);
}
```

## 递归调用

# ranges

- ranges是C++20引入的，对泛型算法的又一大改进
- ranges可以直接使用容器，而非迭代器指定区间范围

```c++
#include <iostream>
#include <algorithm>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> x{1, 2, 3, 4, 5};
    // auto it = std::find(x.begin(), x.end(), 4);
    auto it = std::ranges::find(x, 4);
    std::cout << *it;
}
```

- 可以通过`std::ranges::dangling`避免返回无效的迭代器

```c++
#include <iostream>
#include <algorithm>
#include <vector>
#include <ranges>

auto func() {
    return std::vector<int> {1, 2, 3, 4, 5};
}

int main() {
    std::vector<int> x{1, 2, 3, 4, 5};
    auto it = std::ranges::find(func(), 4);
    std::cout << *it;
}
```

上面这段代码会报错，因为func()的返回值是局部自动变量，是右值，it就成为悬挂指针，后面输出时已丢失资源，`std::ranges`会提醒我们，它返回的it是`std::ranges::dangling`类型

- 映射Proj

当我们想找出umap中value==3的元素时，可以使用一个指针去指向某个数据完成映射

```c++
std::unoreder_map<int, int> umap{{2, 3}};
auto it = std::ranges::find(umap, &std::pair<const int, int>::second);
std::cout << it->first << ' ' << it->second;
```

- view

- 使用哨兵sentinel

# 模板

- 模板的魅力在于将一切能在编译期间处理的问题丢给编译期进行处理，仅在运行时处理那些最核心的动态服务，进而大幅优化运行期的性能

## 函数模板function template

- 函数模板包含两对参数：函数形参/实参、模板形参/实参

- 编译器的两阶段处理：模板语法检查、模板实例化

- 函数模板放宽了对一处定义的限制(见/template/multidefi)

- 函数模板重载

```c++
template <typename T>
void func(T input) {
    // ...
}

template <typename T>
void func(T* input) {
    // ...
}

template <typename T1, typename T2>
void func(T1 input1, T2 input2) {
    // ...
}
```