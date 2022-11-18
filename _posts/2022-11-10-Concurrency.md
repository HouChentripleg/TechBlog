---
title: ConcurrencyBasics(Atomic, memory order)
author: Hou Chen
date: 2022-11-10 11:10:00 +0800
categories: [C++, Concurrency, Basics]
tags: [c++, concurrency, basics]
---

# 代码执行顺序
- 关于代码的执行顺序，先看一个🌰

```c++
// 全局变量
int x = 0;
int y = 0;

// 线程A
x = 1;
y = 2;

// 线程B
if(y == 2) {
    x = 3;
    y = 4;
}
```

- 上面这段代码的执行结果有这样几种可能：
    - x = 1, y = 2
    - x = 3, y = 4
    - x = 1, y = 4
- 结果1和2是很自然的，但是x和y在两个并行的线程中被读写，出现了data race，且编译器指令reorder和CPU乱序执行也可能会造成结果3
    - 编译器：在满足程序“可观测”外部行为的一致性时(大白话就是外部看到的每一次修改顺序是一样的，尽管不同轮的运行顺序可能不一样)，编译器根据上下文调整代码的执行顺序，使其最有利于CPU架构，此即**编译器优化**。对于单个线程而言，先执行x = 1还是y = 2完全无关紧要，因为它们没有外部“可观测”的区别，但对于多线程就有很大的区别了
    - 处理器：CPU也会对代码的执行顺序进行调整，此即**CPU乱序执行**。在多处理器架构中，各个CPU可能存在缓存不一致的问题(每个CPU对自己的缓存的写操作在各个CPU同步之前会造成主存中的同一个数据在各个CPU缓存中的不一致)，这取决于CPU类型、缓存策略和变量地址，x86使用的内存模型基本提供了sequential consistency顺序一致性，而其他架构如ARM就只提供了relaxed consistency松散一致性

---

# std::atomic

> 在引入原子对象前，对于多线程间的内存同步问题，我们或许会想到volatile，在java和C#中它也的确能用来处理指令重排，但这并不是volatile的设计初衷，volatile用于防止编译器优化掉对内存的读写，避免编译器一直只读取该变量在寄存器中的值，也就是说它主要用来读写映射在内存地址上的I/O操作，并不能保证在多处理器环境下多线程能看到相同顺序的数据变化
{: .prompt-info }

> 此外，我们或许也会想到使用std::mutex解决并发读写的问题，但是它是操作系统级的，编译后是一组cpu指令，对于单个变量的读写过于复杂
{: .prompt-info }

> 意识到这些后，C++11里终于引入了适合多线程的内存模型，此处我们重点关注std::atomic、原子操作、内存序等
{: .prompt-tip }

## Intro

- C++11在头文件`<atomic>`中引入的模板`std::atomic`对原子对象进行了封装，将原子类型的读写操作从一组指令最小化到单个CPU指令
- 让我们callback最初的那个🌰，如果希望只有结果1和2，需要将y声明为原子变量，并在x = 1和y = 2之间加入内存屏障，禁止这两句交换顺序，在此常用的是“获得acquire”和“释放release”语义
    - **acquire**: 对内存的**读**操作，当前线程的任何**后面**的读写操作都不允许重排到这个操作的**前面**
    - **release**: 对内存的**写**操作，当前线程的任何**前面**的读写操作都不允许重排到这个操作的**后面**

```c++
// 全局变量
int x = 0;
std::atomic<int> y = 0;

// 线程A
x = 1;
y.store(2, std::memory_order_release);

// 线程B
if(y.load(std::memory_order_acquire) == 2) {
    x = 3;
    y.store(4, std::memory_order_relaxed);
}
```
- 借用吴老师的图示意，每一边的代码都不允许重排越过黄色区域，且若y的release早于y的acquire，线程A中release之前对内存的修改在线程B的acquire后都是可见的(“可见”这个概念我们在后面解释Happens-before时再解释，大致理解起来就是执行acquire和release操作的两个线程能观察到相同的内存修改结果)

![acquire/release](/_posts/img/Concurrency/barrier.png){: width="972" height="589" }
_Full screen width and center alignment_

<br>

- 理论上，`std::atomic<T>`能对任意类型进行原子封装，并非所有的原子类型都能有原子操作，这取决于CPU的架构，以及它所实例化的类型结构是否满足CPU架构对内存对齐的要求，例如对于整型和指针等简单类型，通常封装后就是无锁的原子对象，但另外一些类型，例如64-bit机器上大小不是1, 2, 4, 8的类型，编译器还是会为这些原子对象的操作加锁，可以通过`std::atomic<T>::is_lock_free()`检查该原子类型是否无锁(是否可以用CPU的指令直接完成原子操作)

```c++
#include <iostream>
#include <atomic>

using namespace std;

struct Test {
    float x;
    int y;
    long long z;
};

int main() {
    atomic<Test> t;
    cout << boolalpha << t.is_lock_free() << endl;

    return 0;
}
```

- 当然，各个类型的原子封装也有别名，详细的可见cppreference
```c++
typedef std::atomic<int> atomic_int;
typedef std::atomic<long long> actomic_llong;
```

## 原子操作

- 原子操作主要有下面三类：
    - **store**: 原子地将一个值写入原子对象中
    - **load**: 原子地读取原子对象中的值
    - **Read-Modify-Write**: 原子地读取内存、修改数值、写回内存，整个操作过程中不会有其他写操作的插入

## Happens-before

> Happens-before是理解内存序和内存模型的一个非常基础的概念：如果操作1 Happens-before操作2，则操作1的结果对于操作2可见
{: .prompt-tip }

### 单线程: sequenced-before

- 单线程的情况非常好理解，代码就是顺序执行，有前后顺序，那么前面的语句总是sequenced-before后面的语句
- sequenced-before具有传递性：如果操作1 sequenced-before操作2,操作2 sequenced-before操作3，那么操作1 sequenced-before操作3

### 多线程: synchronizes-with和inter-thread happens-before

- synchronizes-with: 线程A中的操作1和线程B中的操作2是“同步”的，即synchronizes-with(之后我们再解释如何通过内存序实现synchronizes-with)
- 如果线程A中的操作1 synchronizes-with线程B中的操作2，则操作1 inter-thread happens-before 操作2
- inter-thread happens-before也有传递性：
    - 如果线程A中的操作1 synchronizes-with线程B中的操作2，且操作2 sequenced-before操作3，则操作1 inter-thread happens-before 操作3
    - 如果线程A中的操作1 sequenced-before 操作2，操作2 inter-thread happens-before线程B中的操作3，那么操作1 inter-thread happens-before操作3
    - 如果线程A中的操作1 inter-thread happens-before线程B中的操作2，且操作2 inter-thread happens-before线程C的操作3，则操作1 inter-thread happens-before操作3

![happens-before](/_posts/img/Concurrency/happensbefore.png){: width="972" height="589" }
_Full screen width and center alignment_

<br>

- 来举个例子，假设操作2 synchronizes-with 操作3，那么操作1 inter-thread happens-before操作4，也即操作1 Happens-before操作4，所以操作4总能读到操作1对a的修改

```c++
void threadA() {
    a += 1;     // 1
    unlock();   // 2
}

void threadB() {
    lock();     // 3
    cout << a << endl;  // 4
}
```
- 值的注意的是，Happens-before是C++语义层面的概念，并不表示指令在CPU中实际的执行顺序

## 内存序

- 每个原子操作都需要指定一个**内存序memory order**，不同的内存顺序代表不同的语义，也对应不同的**顺序模型order modal**
    - **memory_order_relaxed**: 松散内存序，只用于保证对原子对象的操作是原子的，单个线程内的原子操作都是顺序执行的，不允许指令重排，但不同线程间原子操作的顺序是任意的
    - **memory_order_consume**: 不推荐使用
    - **memory_order_acquire**: 获得操作，在读取该原子对象时，当前线程的任何后面的读写操作都不允许重排到此操作的前面，且其他线程对同一个原子对象释放前的所有内存写入都在当前线程可见
    - **memory_order_release**: 释放操作，在写入该原子对象时，当前线程的任何前面的读写操作都不允许重排到此操作的后面，且当前线程的所有内存写入都对同一个原子对象进行获取的其他线程可见
    - **memory_order_acq_rel**: 获得-释放操作，一个Read-Modify-Write操作同时具有acquire语义和release语义，即它前后的任何读写操作都不允许重排，且其他线程对同一个原子对象释放前的所有内存写入都在当前线程可见，当前线程的所有内存写入都在对同一个原子对象进行获取的其他线程可见
    - **memory_order_seq_cst**: 顺序一致性语义，所有线程看到的所有操作都有一个全局一致的顺序，对于读操作相当于acquire，对于写操作相当于release，对于Read-Modify-Write操作相当于acquire-release，是所有原子操作的默认内存序（此外，顺序一致性还保证了多原子量的修改在所有线程里观察到的修改顺序都相同）

### memory_order_seq_cst

- 下面的代码各轮的运行结果可能都不一样，有可能先执行操作1再执行2，也有可能先执行操作2再执行操作1，但是每一轮中所有线程看到的顺序都是一样的，x和y的store()操作都使用了memory_order_seq_cst，对象的修改是顺序一致的。假如先执行操作1再执行2，那么当tD线程执行操作5退出循环时，即y为true时，x也必然在它之前就置true了，再执行6，++z；假如先执行操作2再执行操作1，那么当tC线程执行操作3时，即x为true时，y必然在此之前也被置true了，所以操作4也会执行。所以无论是哪种执行顺序，操作7的断言永远不会退出

```c++
#include <iostream>
#include <atomic>
#include <thread>
#include <assert.h>

using namespace std;

atomic<bool> x(false), y(false);
atomic<int> z(0);

void threadA() {
    x.store(true, memory_order_seq_cst);    // 1
}

void threadB() {
    y.store(true, memory_order_seq_cst);    // 2
}

void x_then_y() {
    while(!x.load(memory_order_seq_cst));   // 3
    if(y.load(memory_order_seq_cst)) ++z;   // 4
}

void y_then_x() {
    while(!y.load(memory_order_seq_cst));   // 5
    if(x.load(memory_order_seq_cst)) ++z;   // 6
}

int main() {
    thread tA(threadA), tB(threadB), tC(x_then_y), tD(y_then_x);

    tA.join();
    tB.join();
    tC.join();
    tD.join();

    assert(z.load() != 0);  // 7

    return 0;
}
```

- 顺序一致性语义可以实现synchronizes-with的关系：如果一个memory_order_seq_cst的load()操作读到了在这个原子对象中由memory_order_seq_cst的store()操作写入的值，那么memory_order_seq_cst的store()操作synchronizes-with memory_order_seq_cst的load()操作，反映到上面的代码中就是：1 synchronizes-with 3, 2 synchronizes-with 5
- 但是要注意，实现顺序一致性的开销比较大，因为要保证多核CPU的缓存一致

### memory_order_relaxed

- memory_order_relaxed的开销非常小，但是它只保证了store, load, Read-Modify-Write操作的原子性，不能实现synchronizes-with
- 修改一下内存序，再来看一段代码：线程A对x, y这两个原子对象执行store()操作，使用memory_order_relaxed，在同一轮运行中，在某些线程看来可能是先执行1再执行2，另一些线程看来却是先执行2再执行1，所以4处的断言就可能失败，因为2和3没有synchronizes-with的关系，不能保证操作1 Happens-before 操作2

```c++
#include <iostream>
#include <atomic>
#include <thread>
#include <assert.h>

using namespace std;

atomic<bool> x(false), y(false);

void threadA() {
    x.store(true, memory_order_relaxed);    // 1
    y.store(true, memory_order_relaxed);    // 2
}

void threadB() {
    while(!y.load(memory_order_relaxed));   // 3
    assert(x.load());  // 4
}

int main() {
    thread tA(threadA), tB(threadB);

    tA.join();
    tB.join();

    return 0;
}
```

- memory_order_relaxed可以用在不需要线程同步的场景，比如shared_ptr增加引用计数时就用的memory_order_relaxed，因为不需要同步，但减少引用计数的时候就不能用它，因为需要与析构操作同步

### memory_order_acquire, memory_order_release, memory_order_acq_rel

- memory_order_acquire, memory_order_release, memory_order_acq_rel能实现synchronizes-with的关系：如果一个acquire操作在该原子对象上读到一个release操作写入的值，则release操作synchronizes-with acquire操作
    - load操作使用memory_order_acquire
    - store操作使用memory_order_release
    - Read-Modify-Write操作使用memory_order_acquire时表示load操作，使用memory_order_release时表示store操作，使用memory_order_acq_rel兼具两者
- 再来看个例子：显然操作2 synchronizes-with操作3，所以操作1 happens-before操作4，断言永远不会失败，即使x使用的是memory_order_relaxed

```c++
#include <iostream>
#include <atomic>
#include <thread>
#include <assert.h>

using namespace std;

atomic<bool> x(false), y(false);

void threadA() {
    x.store(true, memory_order_relaxed);    // 1
    y.store(true, memory_order_release);    // 2
}

void threadB() {
    while(!y.load(memory_order_acquire));   // 3
    assert(x.load(memory_order_relaxed));  // 4
}

int main() {
    thread tA(threadA), tB(threadB);

    tA.join();
    tB.join();

    return 0;
}
```

- 注意：memory_order_acquire和memory_order_release一定是成对使用的，他俩都不能和memory_order_relaxed组队实现synchronizes-with的关系
- 让我们再使用顺序一致性的那个例子巩固一下acquire/release：这次就不能保证7处的断言一定不失败，因为在同一轮运行中，tC可能看到先1后2，tD却可能看到先2后1，不保证全局一致性，这样就可能导致4和6的load()结果都是false，最终导致z还是0

```c++
#include <iostream>
#include <atomic>
#include <thread>
#include <assert.h>

using namespace std;

atomic<bool> x(false), y(false);
atomic<int> z(0);

void threadA() {
    x.store(true, memory_order_release);    // 1
}

void threadB() {
    y.store(true, memory_order_release);    // 2
}

void x_then_y() {
    while(!x.load(memory_order_acquire));   // 3
    if(y.load(memory_order_acquire)) ++z;   // 4
}

void y_then_x() {
    while(!y.load(memory_order_acquire));   // 5
    if(x.load(memory_order_acquire)) ++z;   // 6
}

int main() {
    thread tA(threadA), tB(threadB), tC(x_then_y), tD(y_then_x);

    tA.join();
    tB.join();
    tC.join();
    tD.join();

    assert(z.load() != 0);  // 7

    return 0;
}
```

## 一致性模型

> 关于一致性模型的引入，借用Modern CPP中的解释：并行执行的多线程在某种程度上可以看作分布式系统，在分布式系统中，任何通信乃至本地操作都需要消耗一定的时间，甚至出现不可靠的通信，假如我们强行将一个对象x在多线程间的操作设为原子操作，也就是任何一个线程在操作完x后，其他线程均能同步感知到x的变化，但是对于x来说，它并没有因为引入多线程带来效率提升，那么有什么办法能提升效率呢？于是就要削弱原子操作在多线程间的同步条件
{: .prompt-info }

- 有四种一致性模型：
    - 线性一致性：最强的一致性，要求任何一次读操作都能读到这个数据的最近一次写入的数据，且所有线程的操作顺序与全局时钟下的顺序是一致的，很难实现
    - 顺序一致性：不要求与全局时钟的顺序一致，但同样要求任何一次读操作都能读到这个数据的最近一次写入的数据
    - 因果一致性：只要求有因果关系的操作顺序得到保障，没有因果关系的操作顺序不做要求
    - 最终一致性：最弱的一致性，只要求操作是原子的，在未来的某个时间点上会被观测到更改后的结果

```
// 线性一致性
        x.store(1)      x.load()
T1 ---------+----------------+------>

T2 -------------------+------------->
                x.store(2)

// x.store(1)严格发生在x.store(2)前，x.store(2)严格发生在x.load()前
```

```
// 顺序一致性
        x.store(1)  x.store(3)   x.load()
T1 ---------+-----------+----------+----->

T2 ---------------+---------------------->
              x.store(2)
或者
        x.store(1)  x.store(3)   x.load()
T1 ---------+-----------+----------+----->

T2 ------+------------------------------->
      x.store(2)

// x.load()必须读到最近一次写入的数据，x.store(2)与x.store(1)并无强制先后，但x.store(2)必须在x.store(3)前发生
```

```
// 因果一致性
      a = 1      b = 2
T1 ----+-----------+---------------------------->

T2 ------+--------------------+--------+-------->
      x.store(3)         c = a + b    y.load()
或者
      a = 1      b = 2
T1 ----+-----------+---------------------------->

T2 ------+--------------------+--------+-------->
      x.store(3)          y.load()   c = a + b
亦或者
     b = 2       a = 1
T1 ----+-----------+---------------------------->

T2 ------+--------------------+--------+-------->
      y.load()            c = a + b  x.store(3)

// 整个过程只有c对a, b产生依赖，因此x,y在何时操作均可，只要保证c = a + b在确定a, b后发生即可
```
    
```
// 最终一致性
    x.store(3)  x.store(4)
T1 ----+-----------+-------------------------------------------->

T2 ---------+------------+--------------------+--------+-------->
         x.read()      x.read()           x.read()   x.read()

// 假设x的初始值为0,那么T2的四次读取就可能有如下结果
3 4 4 4 // x 的写操作被很快观察到
0 3 3 4 // x 的写操作被观察到的时间存在一定延迟
0 0 0 4 // 最后一次读操作读到了 x 的最终值，但此前的变化并未观察到
0 0 0 0 // 在当前时间段内 x 的写操作均未被观察到，
        // 但未来某个时间点上一定能观察到 x 为 4 的情况
```


## 常用成员函数

| API | 说明 |
| --- | --- |
|`store()`|原子地将非原子对象写入原子对象，第二参数为内存序|
|`load()`|原子地从原子对象读取内置的对象，第二参数为内存序|
|`is_lock_free()`|判断原子对象的操作是否无锁(是否可以用CPU指令直接完成原子操作)|
|`exchange()`|原子地替换原子对象的值并获取它先前的值, Read-Modify-Write操作，第二参数为内存序|
|`compare_exchange_weak()` <br> `compare_exchange_striong()`|CompareAndSwap，Read-Modify-Write操作，可以分别指定成功和失败时的内存序，也可以只指定一个，或者使用默认的内存序|
|`wait()`(C++20)|阻塞线程直至被通知原子值改变|
|`notify_one()` <br> `notify_all()`(C++20)|通知在原子对象上阻塞等待的线程|
|`fetch_add()` <br> `fetch_sub()`|只对整数和指针内置对象有效，对原子对象执行+/-操作，返回原始值，Read-Modify-Write操作，第二参数为内存序|
|`++` <br> `--`(前置and后置)|只对整数和指针内置对象有效，对原子对象执行+1/-1操作，Read-Modify-Write操作，使用顺序一执性语义|
|`+=` <br> `-=`(前置and后置)|只对整数和指针内置对象有效，对原子对象执行+/-操作，返回操作后的数值，Read-Modify-Write操作，使用顺序一执性语义|
|`fetch_and()` <br> `fetch_or()` <br> `fetch_xor()`|只对布尔内置对象有效，对原子对象执行与/或/异或操作，返回原始值，Read-Modify-Write操作，第二参数为内存序|
|`&=` <br> `|=` <br> `^=`|只对布尔内置对象有效，对原子对象执行与/或/异或操作，返回操作后的值，Read-Modify-Write操作，使用顺序一执性语义|

---

# 应用

## 自旋锁

- 自旋锁适用于临界区执行时间短的场景，相比于一般的互斥锁，能减少线程上下文切换带来的开销
- 自旋锁需要保证同一时间只有一个线程能获取到锁，且加锁后，被保护的数据总是最新的，也就是需要保证线程同步
- 看下面的代码：线程A往deq中添加数据，push_back()这一条操作的背后就隐含着val的复制、deq迭代器的移动，也许还会resize队列，所以得对这一系列操作加锁，但是整个操作又不是很耗时，没有必要切换线程，所以就采用自旋锁，而且我们需要保证线程A获取到mtx时，对其他线程是可见的，当然线程B的出队也同理，所以**自旋锁需要保证unlock操作 synchronizes-with lock操作**

```c++
deque<int> deq;
spinlock mtx;

void threadA() {
    int val;
    while(val = read_from_remote) {
        mtx.lock();         // 1
        deq.push_back(val); // 2
        mtx.unlock();       // 3
    }
}

void threadB() {
    while(true) {
        mtx.lock();         // 4
        cout << deq.front() << endl;
        deq.pop_front();    // 5
        mtx.unlock();       // 6
    }
}
```
- spinlock就可以使用acquire/release来实现synchronizes-with关系，**锁被占用时为flag为true**：
    - 加锁时：flag初始值是false，加锁后进入lock()函数执行操作7，flag被置为true同时返回false，退出循环完成加锁，此时若有其他线程来抢占锁，7处的exchange()会一直返回true，该线程就会阻塞在循环中
    - 解锁时：flag被置为false，阻塞在7的线程会把flag重新置true抢占锁，但返回false退出循环，再次加锁成功
    - 显然，8 synchronizes-with 7，所以unlock() synchronizes-with lock()，总能保证上一次的解锁与这一次的加锁synchronizes-with

```c++
class spinlock {
private:
    atomic<bool> flag(false);
public:
    void lock() {
        while(flag.exchange(true, memory_order_acquire));   // 7
    }
    void unlock() {
        flag.store(false, memory_order_release); // 8
    }
};
```

## 线程安全的Singleton

- Singleton是一个非常注重性能的设计模式，它适用于在系统中只存在一份实例的特殊类
- 常规的方法是使用一个静态成员指针存储该class的唯一实例，再用静态成员函数获取，但这并不是线程安全的

```c++
#include <atomic>
#include <mutex>

class Singleton {
private:
    // 把ctor, copy-ctor放在private
    Singleton();
    Singleton(const Singleton& other);
public:
    static Singleton* m_instance;   // 静态成员指针存储类的唯一实例
    static Singleton* getInstance();    // 静态成员函数获取静态成员指针
};

Singleton* Singleton::m_instance = nullptr;

// 非线程安全版本：假设threadA和threadB同时进入if()，没法保证第20行只执行一次
Singleton* Singleton::getInstance() {
    // 未创建过实例就创建，否则直接返回
    if(m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}
```

- **加锁**：但代价过高，因为只有在初始创建时才会有并发问题, 后续只需要返回指针即可, 所以无须给整个函数加锁
- **双检查锁**: 锁前锁后双检查, m_instance为空才加锁，否则直接返回, 加锁后再次判断是否为空，避免22判断后&&获取锁之前, 有其他线程创建了对象，但是这种做法也存在问题
    - 22行并没有在锁的保护下，它就有可能与25行并发，导致data race
    - 双检查锁由于内存读写reorder也会导致不安全: 当我们使用第25行创建对象时，我们默认CPU指令的执行逻辑是先开辟内存，再调用对象的ctor，最后返回内存地址给m_instance，但实际上，CPU很可能会先开辟内存，再返回内存地址给m_instance，最后调用对象ctor，此即reorder，对指令的重排序，reorder后返回内存地址给m_instance, 它不是nullptr, threadA还在执行第22行, threadB就会直接返回m_instance, 但是对象的ctor还没有调用，是不完整的对象创建，error

```c++
class Singleton {
private:
    Singleton();
    Singleton(const Singleton& other);
public:
    static Singleton* m_instance;
    static std::mutex m_mutex;
    static Singleton* getInstance();
};

// 线程安全版本: 加锁(全函数版)
Singleton* Singleton::getInstance() {
    std::lock_guard<std::mutex> lck(m_mutex);
    if(m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}

// 双检查锁版本
Singleton* Singleton::getInstance() {
    if(m_instance == nullptr) {
        std::lock_guard<std::mutex> lck(m_mutex);
        if(m_instance == nullptr) {
            m_instance = new Singleton();
        }
    }
    return m_instance;
}
```

- 内存栅栏atomic_thread_fence + memory_order_relaxed版本

```c++
class Singleton {
private:
    Singleton();
    Singleton(const Singleton& other);
public:
    static std::atomic<Singleton*> m_instance;  // 原子对象
    static std::mutex m_mutex;
    static Singleton* getInstance();
};

// 使用atomic_thread_fence, 宽松模型即可
Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);    // 获取内存fence
    if(tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if(tmp == nullptr) {
            tmp = new Singleton();
            std::atomic_thread_fence(std::memory_order_release);    // 释放内存fence
            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return m_instance;
}
```

- 但常用的其实是acquire/release版本：12行 synchronizes-with 18行，15使用memory_order_relaxed是因为它在锁的保护下，能保证线程同步

```c++
class Singleton {
private:
    Singleton();
    Singleton(const Singleton& other);
public:
    static std::atomic<Singleton*> m_instance;  // 原子对象
    static std::mutex m_mutex;
    static Singleton* getInstance();
};

Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_acquire);
    if(tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if(tmp == nullptr) {
            tmp = new Singleton();
            m_instance.store(tmp, std::memory_order_release);
        }
    }
    return m_instance;
}
```

## 并发队列的接口

> 这个例子来自吴老师，并发会给STL的接口带来冲击
{: .prompt-info }

- 回顾`std::queue<T>`的接口，如果我们正通过`front()`访问队列的第一个元素，但与此同时它被另一个线程`pop()`了，怎么办？并发安全的接口大概长下面这样
```c++
template <typename T>
class queue {
public:
    void wait_and_pop(T& dest);
    bool try_pop(T& dest);
};
```
- 利用原子量实现的无锁且高效版并发队列，是个非常复杂的话题，之后单开记录，让我们先mark一下吴老师的推荐阅读
    - [单生产者、单消费者的无锁并发定长环形队列](https://github.com/adah1972/nvwa)
    - [多生产者、多消费者的无锁通用并发队列](https://github.com/cameron314/concurrentqueue)
    - [无锁队列](https://coolshell.cn/articles/8239.html)

---

# reference
- [cppreference](https://en.cppreference.com/w/cpp/thread)
- [现代C++教程：高速上手C++11/14/17/20](https://changkun.de/modern-cpp/)
- 吴咏炜：极客时间《现代C++编程实战》
- C++ Concurrency in action