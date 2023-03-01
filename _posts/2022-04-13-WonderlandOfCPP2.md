---
title: WonderlandOfCPP(part2)
author: Hou Chen
date: 2022-04-20 18:32:00 +0800
categories: [C++, Basics]
tags: [c++, syntax, basics]
---

> 本篇记录了一些C++11/14/17语法及STL库的细枝末节，有些细节以后用到还会再展开
{: .prompt-info }

# constexpr

- `constexpr`修饰的表达式、函数在编译期即可确定为常量表达式
- C++14起被`constexpr`修饰的函数可以在内部使用局部变量、循环和分支等简单语句

```c++
// C++11时递归函数只能这样写
constexpr int fibonacci(const int n) {
    return n == 1 || n == 2 ? 1 : fibonacci(n-1)+fibonacci(n-2);
}

// C++14后就能使用分支啦
constexpr int fibonacci(const int n) {
    if(n == 1) return 1;
    if(n == 2) return 1;
    return fibonacci(n-1)+fibonacci(n-2);
}
```

# if constexpr(C++17)

- C++17支持`if constexpr`关键字，可以在编译期间就完成分支判断

```c++
#include <iostream>

template <typename T>
auto func(const T& t) {
    if constexpr(std::is_integral<T>::value) {
        return t + 1;
    } else {
        return t + 0.001;
    }
}

int main() {
    std::cout << func(5) << std::endl;
    std::cout << func(5.5) << std::endl;
}
```

# if/switch语句中声明临时变量(C++17)

- 以前的C++的变量声明无法放在if或switch语句中，C++17后是允许的

```c++
// 假设我要判断一个数组vec中是否存在值为1的元素，之前的写法会是
const auto it = std::find(vec.begin(), vec.end(), 1);
if(it == vec.end()) return false;
else return true;

// C++17后就能在if语句中声明变量
if(const auto it = std::find(vec.begin(), vec.end(), 1); it != vec.end()) {
    return true;
} else return false;
```

# initializer_list(C++11)

```c++
#include <iostream>
#include <initializer_list>
#include <vector>

class Foo {
public:
    std::vector<int> vec;

    // 初始化列表构造函数
    Foo(std::initializer_list<int> list) {  
        for(auto iter = list.begin(); iter != list.end(); ++iter) {
            vec.push_back(*iter);
        }
    }

    // 初始化列表普通函数
    void func(std::initializer_list<int> list) {
        for(auto iter = list.begin(); iter != list.end(); ++iter) {
            vec.push_back(*iter);
        }
    }
};

int main() {
    Foo foo{1, 2, 3, 4};
    for(auto it = foo.vec.begin(); it != foo.vec.end(); ++it) {
        std::cout << *it << std::endl;
    }

    foo.func({5, 6, 7});
    for(auto it = foo.vec.begin(); it != foo.vec.end(); ++it) {
        std::cout << *it << std::endl;
    }
}
```

# structured bindings(C++17)

- C++11引入的tuple容器可以包含多个非同种类型的值，但那个时候并没有提供简单的方法直接从tuple中获取元素，虽然有`std::get`和`std::tie`，但是使用`std::tie`对tuple/pair拆包还要知道tuple中元素的个数和类型，还是比较麻烦，此外使用`std::tie`还能使用占位符placeholder `std::ignore`忽视不关心的值

- C++17引入了结构化绑定就获取`std::tuple`, `std::pair`和`std::array`元素的简单方式

```c++
// PlayerProfile has type std::tuple<int, const char*, const char*>
auto PlayerProfile = std::make_tuple(22, "HouChen", "China");
std::get<0>(PlayerProfile);     // 22

// tuple
std::string playerName;
std::tie(std::ignore, playerName, std::ignore) = std::make_tuple(23, "tripleg", "Germany");

// pairs
std::string yes, no;
std::tie(yes, no) = std::make_pair("ja", "nein");

// C++17
using Coordinate = std::pair<int, int>;
Coordinate func() {
    return Coordinate{0,0};
}
const auto [x, y] = func(); // structured bindings
```

# auto

- C++20后auto可以函数传参

```c++
auto add(auto x, auto y) {
    return x+y;
}
```

- 此外，C++17后也能通过auto进行模板参数的推导

```c++
template <auto value>
void func() {
    cout << value << endl;
}
func<100>();    // 100作为模板参数传递给func()，value被推导为int类型
```

# decltype

```c++
// C++11时只能这样写尾返回类型trailing return type
template <typename T, typename U>
auto add(T x, U y) -> decltype(x+y) {
    return x+y;
}

// C++14后就可以利用auto进行返回值推导
template <typename T, typename U>
auto add(T x, U y) {
    return x+y;
}
```

# decltype(auto)(C++14)

- C++14支持使用`decltype(auto)`实现转发函数or封装的返回类型推导，且它会保存reference和cv-qualifiers, `auto`就不会保存

```c++
#include <iostream>

std::string func1();
std::string& func2();

// c++11的函数封装
std::string BEFORE1() {
    return func1();
}
std::string& BEFORE2() {
    return func2();
}

// c++14的函数封装
decltype(auto) AFTER1() {
    return func1();
}
decltype(auto) AFTER2() {
    return func2();
}
```

```c++
const int x = 0;
auto x1 = x; // int
decltype(auto) x2 = x; // const int
int y = 0;
int& y1 = y;
auto y2 = y1; // int
decltype(auto) y3 = y1; // int&
int&& z = 0;
auto z1 = std::move(z); // int
decltype(auto) z2 = std::move(z); // int&&
```

# variadic template

- variadic template允许任意个数、任意类型地模板参数
- 有3种常用的方法展开参数包：递归模板函数、变参模板展开、初始化列表展开

```c++
// 递归模板函数
#include <iostream>
using namespace std;

void print1() {} // 递归的底层

template <typename T, typename... Types>
void print1(const T& firstArg, const Types&... args) {
    cout << sizeof...(args) << endl;
    cout << firstArg << endl;
    print1(args...);
}

int main() {
    print1(729, 729.09, "hello", 'a');
}

// 变参模板展开
template <typename T, typename... Types>
void print2(const T& firstArg, const Types&... args) {
    cout << firstArg << endl;
    if constexpr(sizeof...(args) > 0) print2(args...);
}

// 初始化列表展开：初始化列表将会把`(lambda, firstArg)...`完全展开
template <typename T, typename... Types>
void print3(const T& firstArg, const Types&... args) {
    cout << firstArg << endl;
    std::initializer_list<T>{([&args] {
        cout << args << endl;
    }(), firstArg)...};
}
```

- 此外，C++17将variadic template扩展给了折叠表达式

```c++
#include <iostream>
using namespace std;

template <typename... T>
auto sum(T... t) {
    return (t + ...);
}

int main() {
    cout << sum(1+2.5+5+7) << endl; // 15.5

    return 0;
}
```

# nullptr

- `nullptr`是`nullptr_t`类型的空指针，能隐式地转成其他指针类型，而`NULL`表示0，只能转成`bool`

# 强类型枚举strongly-typed enums

- 传统C++中的枚举并非类型安全的：枚举类型会被视为整数，从而让两种完全不同的枚举类型可以直接比较，这不是我们希望看到的，此外，在同一个namespace下不同枚举类型的枚举值名字也不能相同，颇为麻烦
- C++11引入`enum class`对上面的问题做出了优化，能实现类型安全的枚举类，注意:后若未指定类型，默认使用int

```c++
enum class Color : unsigned int {
    Red = 0xff0000,
    Green = 0xff00,
    Blue = 0xff
};
enum class Alert : bool {
    Red,
    Green
};

Color c = Color::Red;
```

- 此外，当我们希望输出类型时，需要重载操作符<<

```c++
#include <iostream>
using namespace std;

enum class new_enum : unsigned int {
    no,
    yes,
    yep
};

template <typename T>
std::ostream& operator<<(typename std::enable_if<std::is_enum<T>::value, std::ostream>::type& stream, const T& e) {
    return stream << static_cast<typename std::underlying_type<T>::type>(e);
}

int main() {
    cout << new_enum::yep << endl;  // 2

    return 0;
}
```

# noexcept

- C++11将异常的声明简化为函数可能抛出异常与函数不能抛出异常，并使用`noexcept`对这两种情况进行限制

```c++
void may_throw()
void no_throw() noexcept;   // 如果no_throw()抛出异常，编译器会使用std::terminate()终止程序
void func1() noexcept(true);    // 不抛异常
void func2() throw();   // 不抛异常
void func3() noexcept(false);   // 可能抛异常
```

- 注意：不抛异常的函数是允许调用可能抛异常的函数，假设后者抛出异常，那么处理程序在遇到不抛异常的函数的最外层时，就使用`std::terminate()`终止程序，这样就能**封锁异常扩散**(sehr wichtig)

- `noexcept`还可用于判断一个表达式是否有异常，无异常返回true，有异常返回false

```c++
#include <iostream>

void may_throw() {
    throw true;
}

auto non_block_throw = []() {
    may_throw();
};

void no_throw() noexcept {
    return;
}

auto block_throw = []() noexcept {
    no_throw();
};

int main() {
    std::cout << std::boolalpha 
        << "may_throw() noexcept? " << noexcept(may_throw()) << std::endl   // false
        << "no_throw() noexcept? " << noexcept(no_throw()) << std::endl     // true
        << "lambda_may_throw() noexcept? " << noexcept(non_block_throw()) << std::endl  // false
        << "lambda_no_throw() noexcept? " << noexcept(block_throw()) << std::endl;      // true

    try {
        may_throw();
    } catch (...) {
        std::cout << "Exception captured from may_throw()!" << std::endl;
    }

    try {
        non_block_throw();
    } catch (...) {
        std::cout << "Exception captured from non_block_throw!" << std::endl;
    }

    try {
        no_throw();
    } catch (...) {
        std::cout << "Exception captured from no_throw()!" << std::endl;
    }

    try {
        block_throw();
    } catch (...) {
        std::cout << "Exception captured from block_throw!" << std::endl;
    }

    return 0;
}

/*
may_throw() noexcept? false
no_throw() noexcept? true
lambda_may_throw() noexcept? false
lambda_no_throw() noexcept? true
Exception captured from may_throw()!
Exception captured from non_block_throw!
*/
```

# 字面量

- 古早的C++传特殊字符会非常麻烦，需要添加大量转义符`\`，C++11提供了原始字符串字面量的写法

```c++
std::string str = "C:\\File\\To\\Path"; // long long ago
std::string str = R"(C:\File\To\Path)"; // C++11
```

# user-defined literals

- C++11使用`T operator "" X(...) {...}`自定义语义，重载双引号后缀运算符，返回类型T，name为X，注意name需要以下划线'_'开头

```c++
#include <iostream>
#include <string>

std::string operator"" _DIY1(const char* str, size_t len) {
    return std::string(str) + "do it by yourself!";
}

std::string operator"" _DIY2(unsigned long long i) {
    return std::to_string(i) + "do it by yourself!";
}

int main() {
    auto str1 = "HouChen"_DIY1;
    auto str2 = 729_DIY2;
    std::cout << str1 << std::endl; // HouChendo it by yourself!
    std::cout << str2 << std::endl; // 729do it by yourself!

    return 0;
}
```

- C++14为`std::chrono`和`std::basic_string`库引入了新的user-defined literals

```c++
using namespace std::chrono_literals;
auto day = 24h;
day.count(); // 24
std::chrono::duration_cast<std::chrono::minutes>(day).count(); // 1440
```


# 内存对齐

- C++11引入`alignof`和`alignas`支持内存对齐
    - 前者能获得与当前平台相关的`std::size_t`类型的值，用于查询该平台的对齐方式
    - 后者可以重新修饰某个struct的对齐方式，实现自定义对齐方式

```c++
#include <iostream>
#include <cstddef>

struct Storage {
    char a;         // 1
    int b;          // 4
    double c;       // 8
    long long d;    // 8
};                  // 8

struct alignas(std::max_align_t) AlignasStorage {
    char a;
    int b;
    double c;
    long long d;
};

int main() {
    std::cout << alignof(Storage) << std::endl; // 8
    std::cout << alignof(AlignasStorage) << std::endl; // 16

    return 0;
}

// std::max_align_t要求每个scalar标量类型的对齐方式严格一致，即long double
```

# extern "C"

- 使用`extern "C"`将*.cpp中的某段代码{}起来，可以让编译器在执行到这一段时以C语言的方式处理，将C和C++分开编译、再统一链接

# std::function(C++11)

- `std::function`可以实现对callable object的封装
- `std::function`对函数进行封装，可看作函数的容器，比使用函数指针进行调用要更安全

```c++
#include <iostream>
#include <functional>

int foo(int arg) {
    return arg;
}

int main() {
    // 封装普通函数
    std::function<int(int)> func1 = foo;

    // 封装lambda expression
    int x = 1;
    std::function<int(int)> func2 = [&](int val) ->int {
        return 1 + x + val;
    };

    std::cout << func1(1) << std::endl;
    std::cout << func2(1) << std::endl;
}
```

# type traits

- type trait是C++11引入的一项template metaprogramming technique，它实际上是一个模板结构体，能在编译期间就对给定类型的属性进行查询or转换
- 举个例子，先来看看`std::is_floating_point`，对于给定的T，该结构体能通过成员变量`value`告诉我们T时不是floating point

```c++
// template <typename T>
// struct is_floating_point;

#include <iostream>
#include <type_traits>

class A {};

int main() {
    std::cout << std::is_floating_point<A>::value << std::endl; // 0
    std::cout << std::is_floating_point<float>::value << std::endl; // 1
    std::cout << std::is_floating_point<int>::value << std::endl; // 0
}
```

- 它能正确判断的原因是因为编译器在编译期间就生成了下面三个结构体

```c++
struct is_floating_point_A {
    static const bool value = false;
};

struct is_floating_point_float {
    static const bool value = true;
};

struct is_floating_point_int {
    static const bool value = false;
};
// 注意value是static的，外界要接触它需要使用::
```

- 下面来看个实用场景，在T为int时调用`algorithm_signed()`，在T为unsigned时调用`algorithm_unsigned()`

```c++
#include <iostream>
#include <type_traits>

void algorithm_signed(int i) {
    std::cout << "algorithm_signed" << std::endl;
}
void algorithm_unsigned(unsigned u) {
    std::cout << "algorithm_unsigned" << std::endl;
}

template <typename T>
void algorithm(T t)
{
    if constexpr(std::is_signed<T>::value)
        algorithm_signed(t);
    else
    if constexpr (std::is_unsigned<T>::value)
        algorithm_unsigned(t);
    else
        static_assert(std::is_signed<T>::value || std::is_unsigned<T>::value, "Must be signed or unsigned!");
}

int main() {

    algorithm(3);   // algorithm_signed

    unsigned x = 3;
    algorithm(x);   // algorithm_unsigned

    // algorithm("hello"); // error

    return 0;
}
```

- 另一个type trait的常用场景是改变变量的类型，例如`std::move(T)`就能输出T&&，它的底层实现长下面这样，利用了`std::remove_reference`去除原来类型的所有&，再转为右值引用&&，最后输出`struct std::remove_reference`的成员变量type

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg)
{
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

# std::chrono(C++11)

- chrono库涵盖了大量用于计时的函数和类型，典型的应用就是评估时间性能

```c++
std::chrono::time_point<std::chrono::steady_clock> start, end;
start = std::chrono::steady_clock::now();
// do something
end = std::chrono::steady_clock::now();

std::chrono::duration<double> elapsed_seconds = end - start;
double t = elapsed_seconds.count();
```

# std::ref

- std::ref(val)用于创造类型为`std::reference_wrapper`的对象，该对象是val的引用，当使用&不能编译or&被编译器的类型推导推没了的情况，`std::cref(val)`则提供对val的const reference

```c++
#include <iostream>
#include <vector>
#include <functional>

int main() {

    // create a container to store reference of objects.
    auto val = 1;
    auto _cref = std::cref(val);
    auto _ref = std::ref(val);
    _ref++;
    //_cref++; // error
    std::vector<std::reference_wrapper<int>> vec;
    // std::vector<int&> test;  // error, can't compile
    vec.push_back(_ref);
    // test.push_back(&val); // error: &val is rvalue
    std::cout << val << std::endl; // 2
    std::cout << vec[0] << std::endl; // 2
    std::cout << _cref; // 2

    return 0;
}
```

# type alias

```c++
template <typename T>
using VEC = std::vector<T>;
VEC<int> v;
```

# 委托构造delegating constructor

- ctor可以再同一个class种调用别的ctor

```c++
class Base {
public:
    int val1;
    int val2;
    Base() {
        val1 = 1;
    }
    Base(int value) : Base() {  // 委托Base()-ctor
        val2 = value;
    }
};

Base b(2);
cout << b.val1; // 1
cout << b.val2; // 2
```

# 继承构造

```c++
class Base {
public:
    int val1;
    int val2;
    Base() {
        val1 = 1;
    }
    Base(int value) : Base() {  // 委托Base()-ctor
        val2 = value;
    }
};

class Subclass : public Base {
public:
    using Base::Base;   // 继承构造
};

Subclass s(3);
cout << s.val1; // 1
cout << s.val2; // 3
```

# 显示虚函数重载 override, final

- `override`显式告诉编译器进行虚函数的覆盖
- `final`防止类被继续继承or终止虚函数的继续覆盖

```c++
struct Base {
    virtual void func() final;
};

struct Subclass1 final : Base {};

struct Subclass2 final : Subclass1 {};  // error

struct Subclass3 : Base {
    void func();    // error
};

```

# 显示禁用默认函数 =default =delete

```c++
class Base {
public:
    Base() = default;
    Base& operator=(const Base&) = delete;  // 禁用copy assignment
};
```

# range-based for loops

- 注意`int`与`int&`的区别

```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int& x : a) x *= 2; // a == { 2, 4, 6, 8, 10 }
for (int x : a) x *= 2; // a == { 1, 2, 3, 4, 5 }
```

# move ctor, assignment

```c++
struct A {
private:
  std::string s;
public:
  A() : s{"test"} {}    // ctor
  A(const A& o) : s{o.s} {} // copy ctor
  A(A&& o) : s{std::move(o.s)} {}   // move ctor
  A& operator=(A&& o) {     // move assignment
   s = std::move(o.s);
   return *this;
  }
};

A f(A a) {
  return a;
}

A a1 = f(A{}); // move-constructed from rvalue temporary
A a2 = std::move(a1); // move-constructed using std::move
A a3 = A{};
a2 = std::move(a3); // move-assignment using std::move
a1 = f(A{}); // move-assignment from rvalue temporary
```

# explicit conversion function

- 转换函数conversion function能进行对象类型间的转换，以`operator`开头，函数名为待转换的类型，没有参数，无需写返回类型，因为编译器会自动推导
- 当ctor加上`explicit`后，ctor只能在构造时使用，不会在类型转换时调用
- 转换函数加上`explicit`只能显示调用

```c++
struct A {
  operator bool() const { return true; }
};

struct B {
  explicit operator bool() const { return true; }
};

A a;
if(a); // OK calls A::operator bool()将类型A转换为bool
bool ba = a; // OK copy-initialization selects A::operator bool()

B b;
if(b); // OK calls B::operator bool()
bool bb = b; // error copy-initialization does not consider B::operator bool()
```

# inline namespace

- All members of an inline namespace are treated as if they were part of its parent namespace
- inline namespace具有传递性: if A contains B, which in turn contains C and both B and C are inline namespaces, C's members can be used as if they were on A

```c++
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
    bool isFirstVersion() { return true; }
  }
  inline namespace Version2 {
    int getVersion() { return 2; }
  }
}

int version {Program::getVersion()};              // Uses getVersion() from Version2
int oldVersion {Program::Version1::getVersion()}; // Uses getVersion() from Version1
bool firstVersion {Program::isFirstVersion()};    // Does not compile when Version2 is added
```

# non-static data member初始化

```c++
// prior to C++11
class Human {
    Human() : age{0} {}
  private:
    unsigned age;
};
// Default initialization on C++11
class Human {
  private:
    unsigned age{0};
};
```

# ref-qualified member functions

- 根据`*this`是左值引用or右值引用，可以调用不同类型的成员函数

```c++
struct Bar {
  // ...
};

struct Foo {
  Bar getBar() & { return bar; }
  Bar getBar() const& { return bar; }
  Bar getBar() && { return std::move(bar); }
private:
  Bar bar;
};

Foo foo{};
Bar bar = foo.getBar(); // calls `Bar getBar() &`

const Foo foo2{};
Bar bar2 = foo2.getBar(); // calls `Bar Foo::getBar() const&`

Foo{}.getBar(); // calls `Bar Foo::getBar() &&`
std::move(foo).getBar(); // calls `Bar Foo::getBar() &&`

std::move(foo2).getBar(); // calls `Bar Foo::getBar() const&&`
```

# binary literals

```c++
0b110   // 6
0b1111'1111 // 255 允许使用分割副'
```

# std::integer_sequence(C++14)

- `std::utility`库的类模板`std::integer_sequence`代表编译期的整数序列
    - 使用`std::make_integer_sequence<T, N>`创建类型为T的0, 1, ..., N-1序列
    - `std::index_sequence_for<T...>`能将一组模板参数转为整数序列

```c++
// 将array转tuple
#include <iostream>
#include <utility>
#include <array>
#include <tuple>
using namespace std;

template<typename Array, std::size_t... I>
decltype(auto) a2t_impl(const Array& a, std::integer_sequence<std::size_t, I...>) {
  return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N, typename Indices = std::make_index_sequence<N>>
decltype(auto) a2t(const std::array<T, N>& a) {
  return a2t_impl(a, Indices());
}

int main() {
    array<int, 5> arr{0, 1, 2, 3, 4};
    auto tuple = a2t(arr);
    cout << std::get<0>(tuple) << std::get<4>(tuple) << endl;
}
```

# reference

- [cppreference](https://en.cppreference.com/w/cpp/thread)
- [现代C++教程：高速上手C++11/14/17/20](https://changkun.de/modern-cpp/)
- [modern-cpp-features](https://github.com/AnthonyCalandra/modern-cpp-features)
