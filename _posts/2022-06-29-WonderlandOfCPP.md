---
title: WonderlandOfCPP
author: HouChen
date: 2022-06-29 18:32:00 +0800
categories: [C++, Basics]
tags: [c++, syntax]
---

# WonderlandOfCpp

> 持续学习，持续更新

## 内存管理与智能指针

### RAII与引用计数

- RAII(Resource Acquisition is Initialization)：构造时申请空间，析构时释放空间

- 引用计数：对于动态分配的对象进行引用计数，每增加一次对该对象的引用，引用计数+1，每删除一次引用，引用计数-1，当该对象的引用计数==0时，自动删除指向的堆内存

- 传统C++中，使用`new/delete`会导致对象的内存所有权不清晰，容易产生不销毁、多销毁的情况，引入智能指针，依赖RAII即可避免

### `shared_ptr`

- 基于引用计数的共享内存

- API

    `use_count()`：查看对象的引用计数

    `get()`：返回shared_ptr类中的内置指针，便于和C语言衔接

    `reset()`：减少一个引用计数

- `std::make_shared<>()`

    避免显示调用`new`

- 第二参数自定义Deleter：当制作内存池时，用完某个内存后不交给系统释放，通过自定义的Deleter可以放回内存池

- 支持数组

    C++17支持`shared_ptr<T[]>`

    C++20支持`make_shared<T[]>`

### `unique_ptr`

- 独占内存

- unique_ptr不支持copy，但支持move

    ```c++
        auto x = std::make_unique<int>(5);
        // auto y = x;  // 非法
        auro y = std::move(x);
    ```

- unique_ptr自定义Deleter时需要指定参数模板

### `weak_ptr`

- 防止shared_ptr循环引用而引入的弱引用智能指针，weak_ptr也指向共享内存，但不影响引用计数

- API

    没有*和->运算符，无法对资源进行操作

    `expired()`：check被引用的资源是否已删除，资源未被释放返回false，资源已被释放返回true

    `lock()`：资源未释放时把weak_ptr转为shared_ptr，资源已释放返回nullptr

## 正则表达式

- 正则表达式描述了一种字符串匹配的模式

- API
    `std::regex_match(std::string, std::regex)`用于匹配字符串和正则表达式，匹配成功返回true，不匹配返回false

    `std::regex_match(std::string, std::smatch, std::regex)` std::smatch便于获取匹配的结果

## 迭代器Iterator

### 插入迭代器

- `back_insert_iterator`对它所关联的容器执行`push_back()`，可缩写为`back_inserter`

- `front_insert_iterator`对它所关联的容器执行`push_front()`，可缩写为`front_inserter`

- `insert_iterator`可对它所关联的容器的指定位置插入元素，可缩写为`inserter`

### 流迭代器

- `istream_iterator`是single-pass的输入迭代器，使用`operator>>`相继读入`istringstream`对象的T类型的数据，默认自动跳过空格

    读操作发生在迭代器递增时，并不在解引用时

    默认缺省的`istream_iterator`是end-of-stream迭代器

- `ostream_iterator`是single-pass的输出迭代器，使用`operator<<`相继写basic_ostream对象的T类型数据，每次写操作后可写入分割字符串delimiter

### 反向迭代器

- `rbegin`, `rend`

### 移动迭代器`move_iterator`

### 并行算法ExecutionPolicy

- 编译时加-O3和-ltbb

- `std::execution::seq`顺序执行

- `std::execution::par`多线程并行执行

- `std::execution::par_unseq`多线程且SIMD(single instruction multiple data)

- `std::execution::unseq`单线程中使用SIMD执行

## bind expression

- `std::bind`用于绑定函数的部分参数，待调用时传齐参数即可

### `std::bind1st`, `std::bind2nd`

- c++98中的`std::bind1st`, `std::bind2nd`初具bind思想，但功能受限

### `std::bind`

- f为callable object，其后是待绑定的参数列表，未绑定的参数使用std::placeholders的占位符`_1`, `_2` ...代替

    ```c++
        template <class F, class... Args>
        bind(F&& f, Args&&... args)
    ```

- 使用`std::bind`时，传入的参数是被复制的，类似于值传递，丧失了引用传递的优势；另外如果传递的参数是个指针，该指针指向一个局部变量，当跳出局部变量的作用域时，就失去了指针所指的内容，只是单纯复制了内容所在的地址

- `std::bind`可以使用`std::ref`或`std::cref`传引用，避免复制

### `bind_front`

- C++20引入的`bind_front`, `bind_back`是对bind的简化

## lambda expression

### 捕获capture list

- 针对lambda函数体中使用的局部自动对象进行捕获，static不用捕获，可直接使用

- 值捕获、引用捕获、混合捕获

- this捕获

- 初始化捕获(c++14)：在构造lambda类时即完成某些初始化的逻辑，避免在函数体里多次计算，提高性能；其次，初始化允许使用任意表达式，也就是说也允许右值捕获

- `*this`捕获(c++17)：this捕获存在丢失this指针指向内容的风险，所以引入`*this`

### 说明符

- `mutable`: 值捕获的参数也可以在lambda函数体内进行修改，本质是删去了lambda类中operator()函数的const属性

- `constexpr`: 函数可在编译or运行时调用

- `consteval`: 函数只能在编译时调用

### 模板形参(C++20)

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

### 捕获时计算(C++14)

    ```c++
        int x = 2;
        int y = 7;
        auto lambda = [z = x + y]() {
            return z;
        }
    ```

### 使用auto避免复制(C++14)

### Lifting

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

### 递归调用

## ranges

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

## 模板

- 模板的魅力在于将一切能在编译期间处理的问题丢给编译期进行处理，仅在运行时处理那些最核心的动态服务，进而大幅优化运行期的性能

### 函数模板function template

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

### 外部模板

### alias template

    '''c++
    using xxx = xxx
    '''

### variadic templates

递归、变参模板展开、初始化列表展开

### 折叠表达式

### 非类型模板参数推导

## 面向对象Object Oriented

### 委托构造

### 继承构造

### 显示虚函数重载：override final

### 显示禁用默认函数 =default =delete

### 强类型枚举

## 并发Concurrency

## 细枝末节

### `constexpr`

- `constexpr`修饰的表达式、函数在编译期即可确定结果

- C++14起`constexpr`函数可以在内部使用局部变量、循环和分支等简单语句

### if/switch语句中声明临时变量(C++17)

### 初始化列表initializer_list(C++11)

### 结构化绑定(C++17)

### `auto`

### `decltype`

### `decltype(auto)`(C++14)

### `if constexpr`(C++17)

### range-based-for(C++11)

### `noexcept`

- C++11将异常的声明简化为函数可能抛出异常与函数不能抛出异常，并使用`noexcept`对这两种情况进行限制

    ```c++
    void may_throw()
    void no_throw() noexcept;   // 如果no_throw()抛出异常，编译器会使用std::terminate()终止程序
    ```

- `noexcept`还可用于判断一个表达式是否有异常，无异常返回true，有异常返回false

- `noexcept`修饰一个函数后能封锁异常扩散(sehr wichtig超级重要)

### 字面量

- 古早的C++传特殊字符会非常麻烦，需要添加大量转义符`\`，C++11提供了原始字符串字面量的写法

    ```c++
    std::string str = "C:\\File\\To\\Path"; // long long ago
    std::string str = R"(C:\File\To\Path)"; // C++11
    ```

- 自定义字面量(C++11): 重载双引号后缀运算符

### 内存对齐

- C++11引入`alignof`和`alignas`支持内存对齐：前者能获得与当前平台相关的`std::size_t`类型的值，用于查询该平台的对齐方式；后者可以重新修饰某个struct的对齐方式，实现自定义对齐方式

### `extern "C"`

- 使用`extern "C"`将*.cpp中的某段代码{}起来，可以让编译器在执行到这一段时以C语言的方式处理，将C和C++分开编译、再统一链接

### `std::function`(C++11)

- `std::function`可以实现对callable object的封装

### 右值引用

- 纯右值(pure rvalue): 纯字面量、非引用返回的临时变量、运算表达式产生的临时变量、lambda表达式

    > 字符串字面量在普通函数中时左值，在类中是右值

- 将亡值(expring value)：即将被销毁、却能被移动的值

  ```c++
  auto func() {
      std::vector<int> tmp = {1, 2, 3, 4, 5};
      return tmp;
  }
  std::vector<int> v = func();
  ```

`func()`返回的`tmp`是xvalue，C++11以后编译器自动将`tmp`隐式右值转换，等价于`static_cast<std::vector<int>&&>(tmp)`，避免拷贝

- C++11提供`std::move()`将左值转为右值

    > 非常量引用不允许绑定到非左值，常量引用允许绑定到非左值

- 右值引用的声明可以让临时对象的生命周期延长

### 移动语义move semantics

### 完美转发

- 引用坍缩

| 函数形参类型 | 实参类型 | 推导后函数形参类型 |
| :---: | :---: | :---:|
| T& | 左引用 | T&|
| T& | 右引用 | T&|
| T&& | 左引用 | T&|
| T&& | 右引用 | T&&|

- `std::forward<>()`保证在传参时的参数类型保持不变(左引用保持左引用，右引用保持右引用)
