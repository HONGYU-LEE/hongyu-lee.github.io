---
title: Effective Modern C++ 读书笔记（六）：并发 API
date: 2022-07-12T17:54:13+08:00
categories:
    - C++
tags:
    - C++
    - Effective Modern C++
    - 读书笔记 
---

# 并发 API

## 条款 35：优先基于任务编程而非基于线程

- std::thread 的 API 没有提供直接获取异步运行函数的返回值的途径，如果那些函数抛出异常，则程序会终止运行。
- 基于线程的程序设计要求手动管理线程耗尽、资源超额、负载均衡，以及跨平台适配。
- 基于任务的程序设计中 std::async 会默认解决以上两个问题。
  - std::async 返回的 future 提供了 get 函数，从而可以获取返回值。即使函数发生了异常，get 函数也能够访问到异常。
  - 当使用默认启动策略调用 std::async 时，系统不保证会创建一个新的线程，它允许调度器把指定函数运行在请求结果的线程中。合理的调度器在系统资源超额或者线程耗尽时就会利用这个自由度。




## 条款 36：如果必须使用异步，则指定 std::launch::async

- std::async 的默认启动策略既允许任务以异步方式执行，也允许使用同步方式。

  - std::launch::async 启动策略意味着函数必须以异步方式运行，即在另一个线程上执行。

  - std::launch::deferred 启动策略意味着函数只会在 std::async 返回的 future 上调用 get 或者 wait 时才会同步执行（即调用方阻塞到函数运行结束为止）。如果 get 或 wait 没有调用则函数不会运行。

  - std::async在不指定启动策略时会使用默认启动策略，即采用对上述两者进行或运算的结果：

    ```cpp
    auto fut1 = std::async(f);                      //使用默认启动策略运行f
    
    //等价于上一条
    auto fut2 = std::async(std::launch::async |     //使用async或者deferred运行f
                           std::launch::deferred,
                           f);
    ```

    默认启动策略运行函数以同步或者异步的方式运行，也因此导致函数具有不确定性。

- 这样的灵活性导致使用 thread_local 变量时的不确定性，隐含了任务可能永远不会被执行的意思，还会影响运用了基于超时的 wait 调用的程序逻辑。

  - 由于默认启动策略的不确定性，导致在使用 thread_local 变量时，如果函数读/写该线程级局部存储（thread local storage，TLS）时无法预知到读取的是哪个线程。

  - 如果使用基于 wait 的循环超时机制，很可能会导致任务永远不会被执行：

    ```cpp
    using namespace std::literals;      //为了使用C++14中的时间段后缀；参见条款34
    
    void f()                            //f休眠1秒，然后返回
    {
        std::this_thread::sleep_for(1s);
    }
    
    auto fut = std::async(f);           //异步运行f（理论上）
    
    while (fut.wait_for(100ms) !=       //循环，直到f完成运行时停止...
           std::future_status::ready)   //但是有可能永远不会发生！
    {
        …
    }
    ```

    对于上面这个代码，如果采用的是 std::launch::deferred 策略，则 fut.wait_for 将总是返回 std::future_status::deferred。这永远不等于 std::future_status::ready ，循环会永远执行下去。

    想要解决这个问题也很简单，只需要加个判断，查看返回值是否为 std::future_status::deferred 即可：

    ```cpp
    auto fut = std::async(f);               //同上
    
    if (fut.wait_for(0s) ==                 //如果task是deferred（被延迟）状态
        std::future_status::deferred)
    {
        …                                   //在fut上调用wait或get来异步调用f
    } else {                                //task没有deferred（被延迟）
        while (fut.wait_for(100ms) !=       //不可能无限循环（假设f完成）
               std::future_status::ready) {
            …                               //task没deferred（被延迟），也没ready（已准备）
                                            //做并行工作直到已准备
        }
        …                                   //fut是ready（已准备）状态
    }
    ```

- 如果必须使用异步，则指定 std::launch::async。

  - 如果下面条件满足不了，则你可能要想确保任务以异步执行，此时就需要指定 std::launch::async：
    1. 任务不需要与调用 get 或 wait 的线程并发执行。
    2. 读/写哪个线程的 thread_local 变量并无影响。
    3. 可以保证会在 std::async 返回的 future 之上调用 get 或 wait，或者可以接受任务永不执行。
    4. 使用 wait_for 或者 wait_until 的代码会考虑任务被延迟的可能。

  

## 条款 37：使 std::thread 类型对象在所有路径皆 unjoinable

- 在所有路径上保证 std::thread 类型对象最终是 unjoinable。
  - unjoinable 的 std::thread 对象包括：
    - **默认构造的 std::thread**。此类 std::thread 没有可以执行的函数，因此也没有对应的底层执行线程。
    - **已经被移动走的 std::thread**。一个 std::thread 所对应的底层执行线程被对应到另一个 std::thread 上。
    - **已经被 join 的 std::thread**。在 join 之后，std::thread 不再对应已结束运行的底层执行线程。
    - **已经被 detach 的 std::thread**。detach 断开了 std::thread 和它的底层执行线程的连接。

- 在析构时调用 join 可能导致难以调试的性能异常。
  - std::thread 的析构函数会等待底层异步执行线程完成。

- 在析构时调用 detach 可能导致难以调试的未定义行为。
  - std::thread 的析构函数会分离 std::thread 类型对象与底层执行线程之间的连接，而该底层执行线程会继续执行。

- 声明类的数据成员时，最后再声明 std::thread 类型对象。
  - 通常一个数据成员的初始化会依赖另一个成员，而 std::thread 对象会在初始化结束后立即执行函数，因此最好将其在最后再声明，这样就能保证一旦构造结束，在前面的所有数据成员都会初始化完毕，可以供 std::thread 数据成员绑定的异步运行的线程安全使用。




## 条款 38：关注不同线程句柄的析构行为

- future 的正常析构行为就是销毁 future 本身的数据。
  - future 的正常析构行为仅会析构 future 对象，并对共享状态里的引用计数（由引用它的 future 和被调用者的 std::promise 共同控制的）进行一次自减。它不会对任何东西实施 join 或 detach。

- 最后一个经由 std::async 创建的共享状态的 future 析构函数会在任务结束前阻塞。
  - 本质上，这种 future 的析构函数对执行异步任务的线程执行了隐式的 join。




## 条款 39：对于一次性事件的通信使用无返回值的 future

- 对于简单的时间通信，基于条件变量的设计需要一个多余的互斥锁，这不仅会给相互关联的检测和反应任务带来约束，还需要反应任务来验证事件是否已发生。

  - 基于条件变量的设计如下面的示例：

    ```cpp
    std::condition_variable cv;         //事件的条件变量
    std::mutex m;                       //配合cv使用的mutex
    
    //检测任务
    …                                   //检测事件
    cv.notify_one();                    //通知反应任务
    
    //反应任务
    …                                       //反应的准备工作
    {                                       //开启关键部分
    
        std::unique_lock<std::mutex> lk(m); //锁住互斥锁
    
        cv.wait(lk);                        //等待通知，但是这是错的！
        
        …                                   //对事件进行反应（m已经上锁）
    }                                       //关闭关键部分；通过lk的析构函数解锁m
    …                                       //继续反应动作（m现在未上锁）
    ```

    由于条件变量需要确保线程安全，因此需要使用互斥锁。而在检测和反应任务这一场景下，检测线程和反应线程之间并不存在数据竞争，这就导致锁不仅带来了性能损耗，还会阻止两个线程同时运行。

    除了互斥锁，条件变量本身也存在问题，即使检测线程没有发出信号，反应线程也有可能会被虚假唤醒。或者在反应线程条用 wait 之间，检测线程发出了信号，这也会导致信号的丢失。

- 使用标志位的设计可以避免上述问题，但这一设计基于轮询而非阻塞。

  - 除了条件变量，我们也可以使用原子的布尔类型作为标记，当标记为 true 时则代表事件发生：

    ```cpp
    std::atomic<bool> flag(false);          //共享的flag；std::atomic见条款40
    
    //检测任务
    …                                       //检测某个事件
    flag = true;                            //告诉反应线程
    
    //反应任务
    …                                       //准备作出反应
    while (!flag);                          //等待事件
    …                                       //对事件作出反应
    ```

    这种方法不需要互斥锁，也没有虚假唤醒的问题，但是其基于轮询而非阻塞，这就导致其大量占用了 cpu 资源。

- 条件变量和标志位可以一起使用，但这样的通信机制设计结果并不让人愉快。

  - 我们也可以将上述两种方案结合起来，用标志位标记事件，用条件变量阻塞线程：

    ```cpp
    std::condition_variable cv;             //跟之前一样
    std::mutex m;
    bool flag(false);                       //不是std::atomic
    
    //检测任务
    …                                       //检测某个事件
    {
        std::lock_guard<std::mutex> g(m);   //通过g的构造函数锁住m
        flag = true;                        //通知反应任务（第1部分）
    }                                       //通过g的析构函数解锁m
    cv.notify_one();                        //通知反应任务（第2部分）
    
    //反应任务
    …                                       //准备作出反应
    {                                       //跟之前一样
        std::unique_lock<std::mutex> lk(m); //跟之前一样
        cv.wait(lk, [] { return flag; });   //使用lambda来避免虚假唤醒
        …                                   //对事件作出反应（m被锁定）
    }
    …                                       //继续反应动作（m现在解锁）
    ```

    这种方法既避免了方法一的事件丢失和虚假唤醒，又弥补了方法二基于轮询的缺点。但是仍然有不好的地方，即就算反应任务被唤醒，他也要检测标志位的值后才决定是否做出反应，这就导致大量无意义的线程阻塞与唤醒，浪费了大量 CPU 时间的同时带来了上下文切换的损耗。

- 使用 std::promise 和 future 的方案就能够回避这些问题，但是这样就需要为了共享状态而使用堆内存，且需要考虑堆内存的分配和销毁开销，且同时有只能使用一次通信的限制。

  - 除了上面提到的方案，还有一个方法能够避免使用条件变量、互斥锁和标记位。即让反应任务在检测任务设置的 future 上 wait 等待通知，例如下面的代码：

    ```cpp
    std::promise<void> p;                   //通信信道的promise
    
    //检测任务
    …                                       //检测某个事件
    p.set_value();                          //通知反应任务
    
    //反应任务
    …                                       //准备作出反应
    p.get_future().wait();                  //等待对应于p的那个future
    …                                       //对事件作出反应                         //继续反应动作（m现在解锁）
    ```

    这个方案不需要互斥锁与条件变量，也不存在虚假唤醒，同时还保证了事件基于阻塞，且在等待时不会浪费系统资源。但是它仍然有许多缺点：

    1. std::promise 和 future 之间有个共享状态，且共享状态是动态分配的，因此就会导致产生在堆上进行内存分配与释放的成本。
    2. std::promise 类型对象只能设置一次，不能用于重复通信。




## 条款 40：当需要并发时使用 std::atomic，特殊内存才使用 volatile

- std::atomic 用于无锁、多线程读写变量的场景。它是用来编写并发程序的工具。

- volatile 用于读写操作不可以被优化的特殊内存，它可以避免编译器优化内存带来的一些问题。

  - volatile 的作用就是告诉编译器当前处理的内存是特殊内存，不要对这块内存做任何特殊优化。如果没有声明 volatile，则编译器会消除一些代码中的重复赋值或者中间结果，例如下面这个代码：

    ```cpp
    int x;
    auto y = x;                             //读x
    y = x;                                  //再次读x
    x = 10;                                 //写x
    x = 20;                                 //再次写x
    
    //编译器优化后
    auto y = x;                             //读x
    x = 20;                                 //写x
    ```

    