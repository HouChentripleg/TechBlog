---
title: MemoryManagment(Heap, Stack, RAII, SmartPointer)
author: Hou Chen
date: 2022-07-01 18:32:00 +0800
categories: [C++, Basics, MemoryManagment]
tags: [c++, syntax, basics]
---

# 堆Heap

- 堆是动态分配的内存区域，通过new分配内存并构造对象，再通过delete手动释放，否则会引发内存泄漏
- 来看下面这个例子，看起来没啥问题，new的内存最后也delete了，但却存在2个问题：如果中间省略的代码抛出异常，那最后肯定没有delete；另外，这本身就在一个函数func()里，根本没有必要使用堆，用栈就ok了
```c++
void func() {
    Bar* ptr = new Bar();
    // ...
    delete ptr;
}
```
更常见也更合理的代码应该是这样的，使用堆是在内存分配和释放并不在一个函数的情况下

```c++
Bar* make_bar(/*...*/) {
    Bar* ptr = nullptr;
    try {
        ptr = new Bar();
    } catch(/*...*/) {
        delete ptr;
        throw;
    }
    return ptr;
}

void func() {
    // ...
    Bar* ptr = make_bar(/*...*/);
    // ...
    delete ptr;
}
```

---

# 栈Stack

- 栈是函数调用过程中产生的局部变量和调用数据的内存区域，满足LIFO(last in first out)，永远不会出现内存碎片，只移动栈指针就能完成内存的分配与释放
- 栈展开stack unwinding: 发生异常时调用析构函数。由下可知，无论是否发生异常，都会调用obj的析构函数

```c++
#include <iostream>

class Obj{
public:
    Obj() {
        std::puts("Obj()");
    }
    ~Obj() {
        std::puts("~Obj()");
    }
};

void func(int n) {
    Obj obj;
    if(n == 2) 
        throw "throw exception";
}

int main() {
    try {
        func(1);
        func(2);
    }
    catch(const char* s) {
        std::puts(s);
    }

    return 0;
}

// Output
/*
Obj()
~Obj()
Obj()
~Obj()
throw exception 
*/
```

---

# RAII

- RAII(resource acquisition is initialization)是C++所特有的资源管理方式
- 对象通常时存储在栈上的，但是有些时候，对象不能or不应该存储在栈上，例如：
    - 对象很大
    - 对象的大小在编译时无法确定
    - 对象是函数的返回值，但由于特殊原因，不应该使用对象的值返回
- 下面来举个Factory Method的例子，create_shape()返回值一定是Shape*类型的指针，这样才能起到**多态**的作用

```c++
enum class shape_type{
    circle,
    triangle,
    rectangle
};

class Shape {};
class Circle : public Shape {};
class Triangle : public Shape {};
class Rectangle : public Shape {};

Shape* create_shape(shape_type type) {
    switch(type) {
        case shape_type::circle:
            return new Circle();
        case shape_type::triangle:
            return new Triangle();
        case shape_type::rectangle:
            return new Rectangle();
    }
}
```

那又如何保证使用create_shape()的返回值时不发生内存泄漏呢？通常的做法是把这个返回值放到一个局部变量中，并保证其析构函数在析构时会清理这个对象

```c++
class shape_wrapper {
private:
    Shape* ptr_;
public:
    explicit shape_wrapper(Shape* ptr = nullptr) : ptr_(ptr) {}
    ~shape_wrapper() {
        delete ptr_;
    }

    Shape* get() const {
        return ptr_;
    }
};

void func() {
    shape_wrapper ptr_wrapper(create_shape(type));  // 返回值放进局部变量，并保证这个对象的dtor在析构时会清理
}
```

- RAII正是依托于栈和析构函数(关闭文件、释放同步、释放其他系统资源等)，能实现所有资源的管理
- 上面这个shape_wrapper类其实就是智能指针的雏形

---

# 改造智能指针类

> 下面我们尝试将shape_wrapper类改写成智能指针类
{: .prompt-tip }

## 模板化

- 使用类模板扩展使用范围

```c++
template <typename T>
class smart_ptr {
private:
    T* ptr_;
public:
    explicit smart_ptr(T* ptr = nullptr) : ptr_(ptr) {}
    ~smart_ptr() {
        delete ptr_;
    }

    T* get() const {
        return ptr_;
    }
};
```

## 指针化

- 附加解引用*、重载->运算符、可用于布尔表达式等功能

```c++
template <typename T>
class smart_ptr {
private:
    T* ptr_;
public:
    explicit smart_ptr(T* ptr = nullptr) : ptr_(ptr) {}
    ~smart_ptr() {
        delete ptr_;
    }

    T* get() const {
        return ptr_;
    }
    T& operator*() const {
        return *ptr_;
    }
    T* operator->() const {
        return ptr_;
    }
    operator bool() const {
        return ptr_;
    }
};
```

## 拷贝构造和拷贝赋值

- 再来考虑拷贝构造和拷贝赋值的问题，最简单的就是禁用它们，至少可以避免`smart_ptr<Shape> ptr2(ptr1)`这种语句通过编译，因为它就算通过了编译也会在运行时产生未定义的行为，例如会造成对同一块内存的**double free**

```c++
smart_ptr(const smart_ptr&) = delete;
smart_ptr& operator=(const smart_ptr&) = delete;
```

- 然后我们再考虑能不能在拷贝智能指针时把对象也拷贝一份？显然这不符合我们减少对象拷贝的初衷，那尝试在拷贝时转移指针的所有权？it seems to work
    - 拷贝构造时通过调用other的release()释放other对指针的所有权，将所有权交给this对象的ptr_
    - 拷贝赋值时通过拷贝构造产生一个临时对象，并调用swap()交换指针的所有权

```c++
smart_ptr(const smart_ptr& other) {
    ptr_ = other.release();
} 
smart_ptr& operator=(const smart_ptr& rhs) {
    smart_ptr(rhs).swap(*this);
    return *this;
}

T* release() {
    T* ptr = ptr_;
    ptr_ = nullptr;
    return ptr;
}
void swap(const smart_ptr& rhs) {
    using std::swap;
    swap(ptr_, rhs.ptr_);
}
```

这里提一下拷贝赋值函数检测自我赋值self assignment的情况：它的异常安全性并不比上面的写法好，如果在赋值过程中抛出异常，而this对象的内容可能已经被部分删除，那么this对象就不再是一个完整的状态了
而上面的代码保证了**强异常安全性**：拷贝赋值函数分成了拷贝构造和交换两个部分，异常只可能发生在拷贝构造部分，那也不会影响this对象，也就是说，无论拷贝构造成功与否，只存在赋值成功与赋值没有效果这两种状态，不会发生因为赋值破坏了this对象的情况

```c++
smart_ptr& operator=(const smart_ptr& rhs) {
    if(this == &rhs)
        return *this;
    // ...
}
```

将拷贝赋值分成拷贝构造和交换的这种写法本质就是auto_ptr的定义，当然我们已经知道在C++17时auto_ptr已经从标准中删除了，因为它太容易造成所有权转移的问题了，给我们写代码带来了诸多问题

## 移动指针

- 接着看移动语义产生的移动构造和移动赋值

```c++
smart_ptr(smart_ptr&& other) {
    ptr_ = other.release();
}
smart_ptr& operator=(smart_ptr rhs) {
    rhs.swap(*this);
    return *this;
}
```

注意：移动赋值的参数改成了值拷贝smart_ptr，这样就不用在函数体中构造临时对象，而是直接生成了新的智能指针，这样**赋值函数的行为是copy还是move，完全取决于构造参数rhs时调用的是copy ctor还是move ctor**

- 此外，注意C++的一条规则：如果创造的类写了move ctor而没有手动写copy ctor，那么后者自动被禁用

## 子类指针->基类指针

- 子类指针转换为基类指针，例如`smart_ptr<circle>`转换为`smart_ptr<shape>`，这里只需要增加一个ctor即可，将子类智能指针类移动给基类智能指针类

```c++
template <typename U>
smart_ptr(smart_ptr<U>&& other) {
    ptr_ = other.release();
}
```

注意：上面这个ctor不被编译器看作move ctor，因而也不能自动触发删除copy ctor的行为

## 引用计数

- shared_ptr不仅共享同一个对象，也贡献同一个引用计数，下面我们先创造一个引用计数类，此处不考虑多线程安全，先实现一个简易版本

```c++
class shared_count {
private:
    long count_;
public:
    shared_count() noexcept : count_(1) {}
    void add_count() noexcept {
        ++count_;
    }
    long reduce_count() noexcept {
        return --count_;
    }
    long get_count() const noexcept {
        return count;
    }
};
```

- 再把引用计数加入智能指针类

```c++
#include <utility>  // std::swap

template <typename T>
class smart_ptr {
private:
    T* ptr_;
    shared_count* shared_count_;
public:
    explicit smart_ptr(T* ptr = nullptr) : ptr_(ptr) {
        if(ptr) {
            shared_count_ = new shared_count();
        }
    }
    ~smart_ptr() {
        if(ptr_ && !shared_count_ -> reduce_count()) {  // 引用计数为0时才彻底删除对象和引用计数
            delete ptr_;
            delete shared_count_;
        }
    }

    // copy ctor
    smart_ptr(const smart_ptr& other) {
        ptr_ = other.ptr_;
        if(ptr_) {
            other.shared_count_ -> add_count();
            shared_count_ = other.shared_count_;
        }
    }
    template <typename U>
    smart_ptr(const smart_ptr<U>& other) noexcept {
        ptr_ = other.ptr_;
        if(ptr_) {
            other.shared_count_ -> add_count();
            shared_count_ = other.shared_count_;
        }
    }
    // move ctor
    template <typename U>
    smart_ptr(smart_ptr<U>&& other) noexcept {
        ptr_ = other.ptr_;
        if(ptr_) {
            shared_count_ = other.shared_count_;
            other.ptr_ = nullptr;
        }
    }
    // copy/move assignment
    smart_ptr& operator=(smart_ptr rhs) noexcept {
        rhs.swap(*this);
        return *this;
    }

    T* get() const noexcept {
        return ptr_;
    }
    // 删除手动释放所有权的release()，不适配
    // 返回引用计数值的成员函数
    long use_count() const noexcept {
        if(ptr_) {
            return shared_count_ -> get_count();
        } else return 0;
    }
    void swap(smart_ptr& rhs) {
        using std::swap;
        swap(ptr_, rhs.ptr_);
        swap(shared_count_, rhs.shared_count_);
    }
    
    T& operator*() const noexcept {
        return *ptr_;
    }
    T* operator->() const noexcept {
        return ptr_;
    }
    operator bool() const noexcept {
        return ptr_;
    }

    // 显式声明模板的各个实例间存在友元关系，否则会编译失败
    template <typename U>
    friend class smart_ptr;
};

// 全局swap()
template <typename T>
void swap(smart_ptr<T>& lhs, smart_ptr<T>& rhs) noexcept {
    lhs.swap(rhs);
}
```

## 指针类型转换

- C++中的显示类型转换如下：
    - static_cast<T>(): 编译时就能确定的T都可以使用static_cast进行转换，运行时不会类型检查，用于非多态类型的转换，例如基本类型转换、子类->父类的指针/引用转换、把void*转为T*
    - dynamic_cast<T>(): 运行时会检查类型是否合法，仅适用于指针/引用的转换，常用于父类与子类的指针/引用转换、把T*转为void*、菱形继承中的up-cast
    - const_cast<T>(): 移除类型的const, volatile属性，常用于const ptr/reference转为ptr/reference
    - reinterpret_cast<T>(): 任意转换，极不安全
- 当在智能指针类中添加以上转换的实现时，需要再添加一个构造函数，允许在对智能指针内部的指针对象赋值时，使用一个现有的智能指针的引用计数

```c++
template <typename U>
smart_ptr(const smart_ptr<U>& other, T* ptr) {
    ptr_ = ptr;
    if(ptr_) {
        other.shared_count_ -> add_count();
        shared_count_ = other.shared_count_;
    }
}
```

接下来就可以实现这四个转换

```c++
template <typename T, typename U>
smart_ptr<T> static_pointer_cast(const smart_ptr<U>& other) {
    T* ptr = static_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> dynamic_pointer_cast(const smart_ptr<U>& other) {
    T* ptr = dynamic_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> const_pointer_cast(const smart_ptr<U>& other) {
    T* ptr = const_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> reinterpret_pointer_cast(const smart_ptr<U>& other) {
    T* ptr = reinterpret_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}
```

## 代码总结

```c++
#include <utility>

class shared_count {
private:
    long count_;
public:
    shared_count() noexcept : count_(1) {}
    void add_count() noexcept {
        ++count_;
    }
    long reduce_count() noexcept {
        return --count_;
    }
    long get_count() const noexcept {
        return count;
    }
};

template <typename T>
class smart_ptr {
private:
    T* ptr_;
    shared_count* shared_count_;
public:
    explicit smart_ptr(T* ptr = nullptr) : ptr_(ptr) {
        if(ptr) {
            shared_count_ = new shared_count();
        }
    }
    ~smart_ptr() {
        if(ptr_ && !shared_count_ -> reduce_count()) {
            delete ptr_;
            delete shared_count_;
        }
    }

    // copy ctor
    smart_ptr(const smart_ptr& other) {
        ptr_ = other.ptr_;
        if(ptr_) {
            other.shared_count_ -> add_count();
            shared_count_ = other.shared_count_;
        }
    }
    template <typename U>
    smart_ptr(const smart_ptr<U>& other) noexcept {
        ptr_ = other.ptr_;
        if(ptr_) {
            other.shared_count_ -> add_count();
            shared_count_ = other.shared_count_;
        }
    }
    // move ctor
    template <typename U>
    smart_ptr(smart_ptr<U>&& other) noexcept {
        ptr_ = other.ptr_;
        if(ptr_) {
            shared_count_ = other.shared_count_;
            other.ptr_ = nullptr;
        }
    }
    // for transform
    template <typename U>
    smart_ptr(const smart_ptr<U>& other, T* ptr) noexcept {
        ptr_ = ptr;
        if(ptr_) {
            other.shared_count_ -> add_count();
            shared_count_ = other.shared_count_;
        }
    }

    // copy/move assignment
    smart_ptr& operator=(smart_ptr rhs) noexcept {
        rhs.swap(*this);
        return *this;
    }

    T* get() const noexcept {
        return ptr_;
    }
    long use_count() const noexcept {
        if(ptr_) {
            return shared_count_ -> get_count();
        } else return 0;
    }
    void swap(smart_ptr& rhs) {
        using std::swap;
        swap(ptr_, rhs.ptr_);
        swap(shared_count_, rhs.shared_count_);
    }
    
    T& operator*() const noexcept {
        return *ptr_;
    }
    T* operator->() const noexcept {
        return ptr_;
    }
    operator bool() const noexcept {
        return ptr_;
    }

    template <typename U>
    friend class smart_ptr;
};

// 全局swap()
template <typename T>
void swap(smart_ptr<T>& lhs, smart_ptr<T>& rhs) noexcept {
    lhs.swap(rhs);
}

template <typename T, typename U>
smart_ptr<T> static_pointer_cast(const smart_ptr<U>& other) noexcept {
    T* ptr = static_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> dynamic_pointer_cast(const smart_ptr<U>& other) noexcept {
    T* ptr = dynamic_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> const_pointer_cast(const smart_ptr<U>& other) noexcept {
    T* ptr = const_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}

template <typename T, typename U>
smart_ptr<T> reinterpret_pointer_cast(const smart_ptr<U>& other) noexcept {
    T* ptr = reinterpret_cast<T*>(other.get());
    return smart_ptr<T>(other, ptr);
}
```

---

# reference
- [cppreference](https://en.cppreference.com/w/cpp/thread)
- [现代C++教程：高速上手C++11/14/17/20](https://changkun.de/modern-cpp/)
- 吴咏炜：极客时间《现代C++编程实战》