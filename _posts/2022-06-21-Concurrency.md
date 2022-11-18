---
title: ConcurrencyBasics(thread, mutex, cv, future)
author: Hou Chen
date: 2022-6-21 18:45:00 +0800
categories: [C++, Concurrency, Basics]
tags: [c++, concurrency, basics]
---

> 编译完一个cpp程序并运行起来，在操作系统OS看来就是一个进程，一个进程可以包含多个线程，多线程并发即为本篇关注的重点
{: .prompt-tip }

# std::thread

## 线程创建

```c++
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

using namespace std;

mutex output_lock;

void func(const char* name) {
    this_thread::sleep_for(100ms);
    lock_guard<mutex> guard(output_lock);
    cout << "I am thread " << name << endl;
}

int main() {
    thread t1(func, "A");
    thread t2(func, "B");
    t1.join();
    t2.join();

    return 0;
}
```

- head file `<thread>`
- 调用thread类的构造函数需要传入新线程的入口函数和参数（参数一般采用值拷贝，若考虑到值拷贝耗时的问题，可以使用指针or引用，但需警惕参数的生命周期，若参数已被销毁，该线程仍尝试访问，就会出现问题）
- 编译细节：Linux编译命令行需要添加`-pthread`参数

## 线程分离

|API|说明|
|---|---|
|join|阻塞主线程直至子线程运行结束|
|detach|子线程独立执行|

- 必须在thread对象销毁前（例如上面那段代码，在结束main()跳出scope{}时发生thread对象的销毁）决定使用`join`还是`detach`，否则thread对象调用析构函数dtor时会在调用`std::terminate`时异常退出
- 调用`join`：主线程（此处指`main()`）一直阻塞在`join`，直至子线程（此处指`func()`）完成任务
- 调用`detach`：子线程独立运行，成为守护线程，即便thread对象销毁也不影响子线程内部任务的执行，并且主线程再也无法与子线程进行通信
- 调用`joinable()`，判断线程是否活跃执行，true则可以设置`join/detach`
- c++20以后可以通过`jthread`类实现析构时自动join，在此之前我们可以自己封装一个这样的thread类
    - 使用variadic template和perfect forwarding构造thread
    - thread禁止拷贝，但可以移动
    - 当`joinable()`为true时（已经join过，已经detach过，或者空线程对象都不满足此条件），才可以对thread对象进行join操作

```c++
#include <thread>
#include <forward>
#include <utility>

using namespace std;

class scoped_thread {
private:
    thread thread_; 
public:
    template <typename... Args>
    // ctor
    scoped_thread(Args&&... args) : thread_(forward<Args>(args)...) {}
    // move-ctor
    scoped_thread(scoped_thread&& other) : thread_(move(other.thread_)) {}
    // copy-ctor
    scoped_thread(const scoped_thread&) = delete;
    // dtor
    ~scoped_thread() {
        if(thread_.joinable()) {
            thread_.join();
        }
    }
};

int main() {
    scoped_thread t1(func, "A");
    scoped_thread t2(func, "B");

    return 0;
}
```

## 一次调用
- `call_once`在多线程环境下也能保证函数只被调用一次，例如资源的初始化
- 下面3个线程都会执行work()，但只有一个线程会执行init()

```c++
#include <iostream>
#include <thread>
#include <mutex>

using namespace std;

void init() {
    cout << "Initializing...\n";
    // ...
}

void work(once_flag* flag) {
    cout << "working...\n";
    call_once(*flag, init);
}

int main() {
    once_flag flag;

    thread t1(work, &flag);
    thread t2(work, &flag);
    thread t3(work, &flag);

    t1.join();
    t2.join();
    t3.join();

    return 0;
}
```

---

# Mutual Exclusion

> 多线程很容易出现共享资源的竞争问题，互斥锁能保证每个线程对共享数据的互斥访问
{: .prompt-info }

## std::mutex

|class|c++标准|说明|
|---|---|---|
|mutex|c++11|最基础的互斥锁|
|recursive_mutex|c++11|递归锁<br> 同一线程对同一互斥锁能无阻塞地多次加锁(也必须有对应数量的解锁操作)|
|timed_mutex|c++11|带超时功能的互斥锁|
|recursive_timed_mutex|c++11|带超时功能的递归互斥锁|
|shared_mutex|c++17|允许共享和独占两种获取方式的互斥锁|
|shared_timed_mutex|c++14|带超时功能的,允许共享和独占两种获取方式的互斥锁|

|API|说明|
|---|---|
|lock|加锁，若锁已被其他线程获取则阻塞|
|try_lock|尝试加锁，获得锁返回true，若锁已被其他线程获取则直接返回false|
|unlock|解锁(只允许在获取锁后调用)|

- `shared_mutex`和`shared_timed_mutex`在头文件`<shared_mutex>`中
- `timed_mutex`和`recursive_timed_mutex`另外有`try_lock_for()`和`try_lock_until`这两个api，分别指定超时的时长和时间点，如果在超时范围内没有获取到锁，则直接返回
- 递归锁能让同一线程对同一互斥锁无阻塞地多次加锁，避免了死锁
- `shared_mutex`实际提供了2把锁，一把基础的互斥锁，一把共享锁（可同时被多个线程获取）
一旦互斥锁被获取，其他线程无法获取互斥锁和共享锁；但若共享锁被获取，其他线程虽然无法获取互斥锁，但还能获取共享锁
这种性质可以用于实现**读写锁**
- `shared_mutex`和`shared_timed_mutex`的API
    - `lock_shared()`加锁，若无法获取则阻塞
    - `try_lock_shared`尝试加锁，若无法获取则直接返回
    - `unlock_shared`解锁
- 请谨记加锁解锁的时间消耗比较高
- 一般使用**锁的粒度**描述锁的范围，考虑到性能问题，需要保证锁的粒度尽可能细，且不要在锁的范围内执行耗时的操作（例如IO操作读磁盘等）

## 锁的RAII包装类

|class|c++标准|说明|
|---|---|---|
|lock_guard|c++11|基于作用域的mutex wrapper|
|unique_lock|c++11|可移动的mutex wrapper|
|shared_lock|c++14|可移动的shared_mutex wrapper|
|scoped_lock|c++17|多个锁的免死锁wrapper|

- 创建`lock_guard`对象时接收给定互斥体的所有权，离开创建它的作用域时，自动调用dtor销毁对象，解锁互斥锁，注意此类不可复制，不能移动，只在一个scope中使用，不能显示调用`lock()`和`unlock()`
- `unique_lock`不可复制，但可以移动，使得它可以被函数回传，也能放入STL容器中，还能显示调用`lock()`和`unlock()`，比`lock_guard`更灵活，但占用空间更大且更慢
使用条件变量时必须使用`unique_lock`
- `scoped_lock`将多个锁包装成一种锁类，用于线程一次性申请多个锁，避免死锁

```c++
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <functional>
#include <string>
#include <chrono>

using namespace std;

struct Employee {
    string id_;
    vector<string> dance_partners;
    mutex mtx;

    Employee(string id) : id_(id) {}
    string output() const {
        string ret = "Employee " + id_ + " has dance partners: ";
        for(const string& partner : dance_partners) {
            ret += partner + " ";
        }
        return ret;
    }
};

void send_email(Employee&, Employee&) {
    // 模拟耗时的发送邮件操作
    this_thread::sleep_for(1s);
}

void assign_dance_partner(Employee& e1, Employee& e2) {
    static mutex io_mutex;
    {
        lock_guard<mutex> lck(io_mutex);
        cout << e1.id_ << " and " << e2.id_ << " are waiting for locks\n";
    }

    {
        scoped_lock lock(e1.mtx, e2.mtx);   // 使用scoped_lock锁定了一组互斥锁, 避免死锁
        {
            lock_guard<mutex> lck(io_mutex);
            cout << e1.id_ << " and " << e2.id_ << " get locks\n";
        }
        e1.dance_partners.push_back(e2.id_);
        e2.dance_partners.push_back(e1.id_);
    }

    send_email(e1, e2);
    send_email(e2, e1);
}

int main() {
    Employee alice("alice"), bob("bob"), chris("chris"), dave("dave");

    vector<thread> threads;
    threads.emplace_back(assign_dance_partner, ref(alice), ref(bob));
    threads.emplace_back(assign_dance_partner, ref(chris), ref(bob));
    threads.emplace_back(assign_dance_partner, ref(chris), ref(alice));
    threads.emplace_back(assign_dance_partner, ref(dave), ref(bob));

    for(auto& thread : threads) {
        thread.join();
    }
    cout << alice.output() << '\n' << bob.output() << '\n' << chris.output() << '\n' << dave.output() << '\n';

    return 0;
}

/*
alice and bob are waiting for locks
alice and bob get locks
chris and bob are waiting for locks
chris and bob get locks
chris and alice are waiting for locks
chris and alice get locks
dave and bob are waiting for locks
dave and bob get locks
Employee alice has dance partners: bob chris 
Employee bob has dance partners: alice chris dave 
Employee chris has dance partners: bob alice 
Employee dave has dance partners: bob
*/
```

|锁定策略|c++标准|说明|
|---|---|---|
|defer_lock|c++11|defer_lock_t空结构体的实例，不获取互斥锁的所有权|
|try_to_lock|c++11|try_to_lock_t空结构体的实例，尝试获取互斥锁的所有权而不阻塞|
|adopt_lock|c++11|adopt_lock_t空结构体的实例，假设调用方已经获取互斥锁的所有权|


```c++
#include <thread>
#include <mutex>
#include <iostream>

using namespace std;

struct bank_account {
    int balance;
    mutex mtx;
    explicit bank_account(int balance) : balance(balance) {}
};

void transfer(bank_account& from, bank_account& to, int amount) {
    // 法一
    unique_lock<mutex> lock1(from.mtx, defer_lock);
    unique_lock<mutex> lock2(to.mtx, defer_lock);
    lock(lock1, lock2);

    /* 法二
    lock(from.mtx, to.mtx);
    lock_guard<mutex> lock1(from.mtx, adopt_lock);
    lock_guard<mutex> lock2(to.mtx, adopt_lock);
    */

   /* 法三
   scoped_lock lock(from.mtx, to.mtx);
   */

  from.balance -= amount;
  to.balance += amount;
}

int main() {
    bank_account zhangsan(1000);
    bank_account lisi(2000);

    thread t1(transfer, ref(zhangsan), ref(lisi), 100);
    thread t2(transfer, ref(lisi), ref(zhangsan), 50);

    t1.join();
    t2.join();

    cout << zhangsan.balance << endl;
    cout << lisi.balance << endl;

    return 0;
}
```

---

# std::condition_variable

> 条件变量的创建是为了唤醒等待中的线程从而避免死锁，条件变量始终关联着一个互斥锁
{: .prompt-info }

|class/function/enum|c++标准|说明|
|---|---|---|
|condition_variable|c++11|class, 提供关联`std::unique_lock`的条件变量|
|condition_variabke_any|c++11|class, 提供关联任何锁类型的条件变量|
|notify_all_at_thread_exit|c++11|function, 线程结束时解锁并调用`notify_all()`|
|cv_status|c++11|enum, 列出条件变量上定时等待的可能结果|

- head file `<condition_variables>`
- cv_status为condition_variable和condition_variabke_any的`wait_for()`和`wait_until()`所用
```c++
enum class cv_status {
    no_timeout,
    timeout
};
```

|API|说明|
|---|---|
|notify_one()|唤醒一个等待中的线程|
|notify_all()|唤醒所有等待中的线程|
|wait()|阻塞当前线程，直至条件变量被唤醒|
|wait_for()|指定时长地阻塞当前线程，直至条件变量被唤醒|
|wait_until()|指定时间点地阻塞当前线程，直至条件变量被唤醒|

## Producer-Consumer

```c++
#include <iostream>
#include <queue>
#include <chrono>
#include <thread>
#include <mutex>
#include <atomic>
#include <condition_variable>

using namespace std;

int main() {
    queue<int> depository;  // 仓库
    mutex mtx;
    condition_variable cv;
    atomic<bool> notified = false;  // 通知信号
    int num = 0;

    // producer
    auto producer = [&]() {
        for(int i = 0; ; i++) {
            this_thread::sleep_for(500ms);  // 模拟生产操作, 耗时操作放在加锁范围外
            // 对共享资源仓库的操作
            unique_lock<mutex> lck(mtx);
            cout << this_thread::get_id() << " is producing " << i << endl;
            depository.push(i);
            notified = true;    // 通知cs来消费
            cv.notify_one();    // cv.notify_all()也ok
        }
    };

    // consumer
    auto consumer = [&]() {
        while(true) {
            unique_lock<mutex> lck(mtx);
            while(!notified) {  // 无商品则阻塞
                cv.wait(lck);
            }
            lck.unlock();   // 短暂解锁, 让producer操作仓库
            this_thread::sleep_for(1s); // 模拟消费操作, 慢于producer
            lck.lock();
            while(!depository.empty()) {
                cout << this_thread::get_id() << " is consuming " << depository.front() << endl;
                depository.pop();
            }
            notified = false;   // 通知p来生产
        }
    };

    thread p(producer); // 1个producer
    thread cs[2];   // 2个consumer
    for(int i = 0; i < 2; i++) {
        cs[i] = thread(consumer);
    }
    p.join();
    for(int i = 0; i < 2; i++) {
        cs[i].join();
    }

    return 0;
}

/*
140131060971072 is producing 0
140131060971072 is producing 1
140131052578368 is consuming 0
140131052578368 is consuming 1
140131060971072 is producing 2
140131044185664 is consuming 2
140131060971072 is producing 3
140131052578368 is consuming 3
140131060971072 is producing 4
140131044185664 is consuming 4
140131060971072 is producing 5
140131052578368 is consuming 5
140131060971072 is producing 6
140131044185664 is consuming 6
140131060971072 is producing 7
140131052578368 is consuming 7
140131060971072 is producing 8
140131044185664 is consuming 8
140131060971072 is producing 9
140131052578368 is consuming 9
*/
```

---

# Futures

## Preview
> 没有future类之前，如何获取后台任务的执行结果？传统做法是使用信号量or条件变量，c++20才支持信号量，这里我们先用条件变量来实现
{: .prompt-info }

```c++
#include <iostream>
#include <queue>
#include <chrono>
#include <thread>
#include <mutex>
#include <condition_variable>

using namespace std;

class scoped_thread {
private:
    thread thread_; 
public:
    template <typename... Args>
    // ctor
    scoped_thread(Args&&... args) : thread_(forward<Args>(args)...) {}
    // move-ctor
    scoped_thread(scoped_thread&& other) : thread_(move(other.thread_)) {}
    // copy-ctor
    scoped_thread(const scoped_thread&) = delete;
    // dtor
    ~scoped_thread() {
        if(thread_.joinable()) {
            thread_.join();
        }
    }
};

void work(condition_variable& cv, int& result) {
    this_thread::sleep_for(1s); // 模拟耗时的操作
    result = 729;
    cv.notify_one();
}

int main() {
    condition_variable cv;
    mutex mtx;
    int result;
    scoped_thread t(work, ref(cv), ref(result));

    cout << "waiting for result...\n";
    unique_lock<mutex> lck(mtx);
    cv.wait(lck);
    cout << "result is " << result << endl;

    return 0;
}
```
- 为了一个小小的result，我们竟然定义了5个变量(cv, mtx, thread, lck, result)，还用了`std::ref()`传递cv和result的引用给线程的ctor(因为线程ctor默认通过值拷贝的方式传递参数)，写起来非常繁琐，所以就诞生了future
- 当然C++11之前也可以使用全局变量去存储result，主线程需要时再去取

|class/function/enum|c++标准|说明|
|---|---|---|
|async|c++11|function template, 异步运行一个函数，并返回存储结果的std::future类型的对象|
|future|c++11|class template, 提供了访问异步操作结果的途径，只能move不能copy |
|packaged_task|c++11|class template, 函数包装器，存储其返回值以进行异步获取，只能move不能copy |
|promise|c++11|class template, 存储一个值以进行异步获取，只能move不能copy |
|shared_future|c++11| class template, 允许多个线程等待同一个shared state, 可copy, 多个shared_future对象能指向同一个shared state |

## std::async

|异步执行策略|说明|
|---|---|
|std::launch::async|开启新线程，立刻异步执行任务|
|std::launch::deferred|lazy evaluation, 调用方第一次请求结果时才执行任务|

## std::future
- 异步操作是指通过`std::async, std::packaged_task, std::promise`创建的操作
- 以上的异步操作会向创建者提供一个std::future类型的对象object
- 异步操作的创建者可以从这个object中查询，等待并获取异步操作的结果，若结果not ready，就阻塞等待
- 异步操作结果ready后，通过修改std::future的shared state就能把结果发送给创建者

| API | 说明 |
| --- | --- |
| get() | 返回结果 |
| share() | 从*this转移shared state给shared_future并返回shared_future |

```c++
#include <iostream>
#include <future>
#include <thread>
#include <chrono>

using namespace std;

int work() {
    this_thread::sleep_for(1s); // 模拟耗时的操作
    return 729;
}

int main() {
    auto future = async(launch::async, work);   // 函数模板自行推导出返回类型, future<int>

    cout << "waiting for result...\n";
    cout << "result is " << future.get() << endl;

    return 0;
}
```
- 与使用条件变量的例子相比，使用std::future就只产生了一个future对象
- 使用成员函数`get()`可以获得异步操作的结果or异常，注意一个future对象只能调用一次`get()`，第二次调用是未定义的行为，会导致程序crash down，所以也不建议future对象在多线程中使用(吴老师提到这种情况可以move-ctor一个shared_future，或者调用future的share()生成一个shared_future)

## std::promise
- std::promise存储value或者exception，后续可以异步地被std::promise创建的std::future对象获取（相当于在promise-future这个channel通信中，promise作为push端，future作为pop端）
- 每个promise对象都与future对象中的shared state关联，有3种状态:
    - ***ready***: promise对象将value或者exception存储到shared state中，标记state为ready状态，解除线程的阻塞等待
    - ***release***: promise对象放弃对shared state的引用，如果此promise对象是shared state的最后一个引用，则销毁shared state（类似引用计数）
    - ***abandon***: promise对象存储std::future_error类型的异常，error code为std::future_errc::broken_promise，使shared state变为ready状态，然后释放它

| API | 说明 |
| --- | --- |
| get_future() | 返回一个与promise关联的future对象 |
| set_value() | 设置value | 
| set_value_at_thread_exist() | 在线程退出时设置value |
| set_exception() | 设置exception |
| set_exception_at_thread_exist() |  在线程退出时设置exception |

```c++
#include <iostream>
#include <future>
#include <thread>
#include <chrono>
#include <utility>

using namespace std;

class scoped_thread {
private:
    thread thread_; 
public:
    template <typename... Args>
    // ctor
    scoped_thread(Args&&... args) : thread_(forward<Args>(args)...) {}
    // move-ctor
    scoped_thread(scoped_thread&& other) : thread_(move(other.thread_)) {}
    // copy-ctor
    scoped_thread(const scoped_thread&) = delete;
    // dtor
    ~scoped_thread() {
        if(thread_.joinable()) {
            thread_.join();
        }
    }
};

void work(promise<int> prom) {
    this_thread::sleep_for(1s); // 模拟耗时的操作
    prom.set_value(729);
}

int main() {
    promise<int> prom;
    auto future = prom.get_future();
    scoped_thread t(work, move(prom));  // 把prom移动给新线程, 主线程就不需要再管理它的生命周期

    cout << "waiting for result...\n";
    cout << "result is " << future.get() << endl;

    return 0;
}
```
- 相比于function template: async来说，promise不需要在函数结尾的时候才去设置future的值（也就是说不用return
- 还有一个有意思的点，使用void特化的promise-future对能对线程间的状态发信号，当一个线程在`future<void>`上等待时(使用get()或者wait())，另一个线程可以通过调用`promise<void>`的set_value()让它结束等待，继续往下执行

```c++
#include <iostream>
#include <future>
#include <thread>
#include <chrono>
#include <utility>

using namespace std;

void work(promise<void> prom) {
    this_thread::sleep_for(5s);
    prom.set_value();
    cout << "void" << endl;
}

int main() {
    promise<void> prom;
    auto future = prom.get_future();
    thread t(work, move(prom));

    future.wait();
    t.join();

    return 0;
}
```

## std::packaged_task
- packaged_task可以包装任何callable target，包括function, lambda expression, bind expression还有functor
- packaged_task不会自己启动，要其对象被调用时才会异步地执行任务，并将返回值or异常存储在shared state中，让future对象获取

```c++
#include <iostream>
#include <future>
#include <thread>
#include <chrono>
#include <utility>

using namespace std;

class scoped_thread {
private:
    thread thread_; 
public:
    template <typename... Args>
    // ctor
    scoped_thread(Args&&... args) : thread_(forward<Args>(args)...) {}
    // move-ctor
    scoped_thread(scoped_thread&& other) : thread_(move(other.thread_)) {}
    // copy-ctor
    scoped_thread(const scoped_thread&) = delete;
    // dtor
    ~scoped_thread() {
        if(thread_.joinable()) {
            thread_.join();
        }
    }
};

int work() {
    this_thread::sleep_for(1s); // 模拟耗时的操作
    return 729;
}

int main() {
    packaged_task<int()> task(work);
    auto future = task.get_future();
    scoped_thread t(move(task));

    cout << "waiting for result...\n";
    cout << "result is " << future.get() << endl;

    return 0;
}
```
- 再来看看其他callable target的使用方法

```c++
#include <iostream>
#include <thread>
#include <future>
#include <cmath>
#include <functional>

int func(int x, int y) {
    return std::pow(x, y);
}

void task_lambda() {
    std::packaged_task<int(int, int)> task([](int a, int b){
        return std::pow(a, b);
    });
    std::future<int> future = task.get_future();
    task(2, 1);
    std::cout << "task_lambda: " << future.get() << std::endl;
}

void task_bind() {
    std::packaged_task<int()> task(std::bind(func, 2, 2));
    std::future<int> future = task.get_future();
    task();
    std::cout << "task_bind: " << future.get() << std::endl;
}

void task_thread() {
    std::packaged_task<int(int, int)> task(func);
    std::future<int> future = task.get_future();
    std::thread th(std::move(task), 2, 3);
    th.join();
    std::cout << "task_thread: " << future.get() << std::endl;
}

int main() {
    task_lambda();
    task_bind();
    task_thread();

    return 0;
}
```

## std::shared_future
- 虽然多个shared_future对象能指向同一个shared state，但每个线程如果通过它自身的shared_future对象(copy版)访问这同一个shared state，是线程安全的
- 来看cppreference的一个例子，shared_future同时向多个线程发送信号，类似于`std::condition_variable::notify_all()`

```c++
#include <iostream>
#include <chrono>
#include <future>

using namespace std;

int main() {
    promise<void> ready_promise, t1_ready_promise, t2_ready_promise;
    shared_future<void> ready_future = ready_promise.get_future();

    chrono::time_point<chrono::high_resolution_clock> start;

    auto func1 = [&, ready_future]() -> chrono::duration<double, milli> {
        t1_ready_promise.set_value();
        cout << "func1 lambda\n"; 
        ready_future.wait();    // 等待来自main()的信号
        cout << "func1 ready_future\n";
        return chrono::high_resolution_clock::now() - start;
    };

    auto func2 = [&, ready_future]() -> chrono::duration<double, milli> {
        t2_ready_promise.set_value();
        cout << "func2 lambda\n"; 
        ready_future.wait();    // 等待来自main()的信号
        cout << "func2 ready_future\n";
        return chrono::high_resolution_clock::now() - start;
    };

    cout << "1...\n";
    auto fut1 = async(launch::async, func1);
    auto fut2 = async(launch::async, func2);

    // 等待线程get ready
    cout << "2...\n";
    t1_ready_promise.get_future().wait();
    t2_ready_promise.get_future().wait();

    start = chrono::high_resolution_clock::now();

    cout << "3...\n";
    ready_promise.set_value();  // 向自线程发送信号(to 16, 24行)

    cout << "4...\n";
    cout << "thread1 received signal " << fut1.get().count() << "ms after start\n";
    cout << "thread2 received signal " << fut2.get().count() << "ms after start\n";

    return 0;
}
```
- 30, 31行异步地开启新线程，2个线程执行cout后分别阻塞在16, 24行，直到主线程通过41行发送信号，2个自线程才继续执行（我在各个操作后加了输出，便于分析执行步骤）

---

# reference
- [cppreference](https://en.cppreference.com/w/cpp/thread)
- [现代C++教程：高速上手C++11/14/17/20](https://changkun.de/modern-cpp/)
- 吴咏炜：极客时间《现代C++编程实战》