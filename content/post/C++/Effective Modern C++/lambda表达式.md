---
title: Effective Modern C++ 读书笔记（五）：lambda 表达式
date: 2022-07-12T16:54:13+08:00
categories:
    - C++
tags:
    - C++
    - Effective Modern C++
    - 读书笔记 
---

# lambda 表达式

## 条款 31：避免使用默认捕获模式

- 默认的按引用捕获可能会导致悬空指针问题。

  - 按引用捕获会导致闭包中包含指涉到局部变量的的引用，或者是指涉到定义 lambda 表达式的作用域内的形参的引用。一旦 lambda 表达式所创建的闭包越过了该局部变量/形参的生命周期，就会导致引用空悬。

    例如下面的代码，由于 lambda 中指涉了局部变量 divisor，因此当 addDivisorFilter 的生命周期结束时，该变量也会消亡，而我们添加进去的那个 labmda 表达式则操作了一个已经释放的空间，因此会产生未定义的行为。

    ```cpp
    void addDivisorFilter()
    {
        auto calc1 = computeSomeValue1();
        auto calc2 = computeSomeValue2();
    
        auto divisor = computeDivisor(calc1, calc2);
    
        filters.emplace_back(                               //危险！对divisor的引用
            [&](int value) { return value % divisor == 0; } //将会悬空！
        );
    }
    ```

    解决这个问题最简单的方法就是改为按值的默认捕获模式：

    ```cpp
    filters.emplace_back( 							    //现在divisor不会悬空了
        [=](int value) { return value % divisor == 0; }
    );
    ```

- 默认的按值捕获对于悬空指针较为敏感（尤其是 this 指针），并且它会误导人们认为 lambda 表达式是独立的。

  - 当我们使用按值捕获时，可能会出现一些隐含的问题，例如下面的代码：

    ```cpp
    using FilterContainer = 					//跟之前一样
        std::vector<std::function<bool(int)>>;
    
    FilterContainer filters;                    //跟之前一样
    
    void doSomeWork()
    {
        auto pw =                               //创建Widget；std::make_unique
            std::make_unique<Widget>();         //见条款21
    
        pw->addFilter();                        //添加使用Widget::divisor的过滤器
    
        …
    }              
    
    void Widget::addFilter() const
    {
        filters.emplace_back(
            [=](int value) { return value % divisor == 0; }
        );
    }
    ```

    在该代码中，实际上并没有将 divisor 的值拷贝进 lambda 中，而是隐式的拷贝了 this 指针，并隐式使用这个 this 指针的拷贝来访问 divisor。一旦 pw 的生命周期结束，make_unique 就会将它的资源释放掉，这时 lambda 中访问到的就是一个悬空的 this 指针。

    要解决这个问题，我们可以使用一个临时变量来拷贝类的数据成员，并以传值的形式传递进 lambda 中：

    ```cpp
    //c++11
    void Widget::addFilter() const
    {
        auto divisorCopy = divisor;                 //拷贝数据成员
    
        filters.emplace_back(
            [divisorCopy](int value)                //捕获副本
            { return value % divisorCopy == 0; }	//使用副本
        );
    }
    
    //c++14
    void Widget::addFilter() const
    {
        filters.emplace_back(                   //C++14：
            [divisor = divisor](int value)      //拷贝divisor到闭包
            { return value % divisor == 0; }	//使用这个副本
        );
    }
    ```

  - 默认的值捕获模式看起来似乎是独立的，与外界绝缘，然而实际并非如此，因为 lambda 表达式还依赖于静态存储期的对象：

    ```cpp
    void addDivisorFilter()
    {
        static auto calc1 = computeSomeValue1();    //现在是static
        static auto calc2 = computeSomeValue2();    //现在是static
        static auto divisor =                       //现在是static
        computeDivisor(calc1, calc2);
    
        filters.emplace_back(
            [=](int value)                          //什么也没捕获到！
            { return value % divisor == 0; }        //引用上面的static
        );
        ++divisor;                                  //调整divisor
    }
    ```




## 条款 32：使用初始化捕获将对象移入闭包

- 使用 C++ 14 的初始化捕获将对象移动到闭包中。

  在 C++ 11 中没有办法将一个对象移动到 lambda 的闭包中，而 C++ 14 提出的初始化捕获机制则可以做到这件事：

  ```cpp
  class Widget {                          //一些有用的类型
  public:
      …
      bool isValidated() const;
      bool isProcessed() const;
      bool isArchived() const;
  private:
      …
  };
  
  auto pw = std::make_unique<Widget>();   //创建Widget；使用std::make_unique
                                          //的有关信息参见条款21
  
  …                                       //设置*pw
  
  auto func = [pw = std::move(pw)]        //使用std::move(pw)初始化闭包数据成员
              { return pw->isValidated()
                       && pw->isArchived(); };
  ```

  位于 = 号左侧的是指定的闭包类成员变量名，它的作用域就是闭包类的作用域。位于 = 号右侧则是该变量的初始化表达式，它的作用域则和外面一层的相同。

- 在 C++ 11 中通过自定义类或 std::bind 的方式来模拟初始化捕获。

  - **自定义类**。lambda 表达式的原理其实也就是生成一个类并重载它的()操作符，那么我们完全可以手动模拟这个过程，例如下面的代码：

    ```cpp
    class IsValAndArch {                            
    public:
        using DataType = std::unique_ptr<Widget>;
        
        explicit IsValAndArch(DataType&& ptr)     
        : pw(std::move(ptr)) {}
        
        bool operator()() const
        { return pw->isValidated() && pw->isArchived(); }
        
    private:
        DataType pw;
    };
    
    auto func = IsValAndArch(std::make_unique<Widget>());
    ```

  - **std::bind**。std::bind 同样可以生成函数对象，因此我们也可以使用它来模拟初始化捕获：

    ```cpp
    std::vector<double> data;               //同上
    
    …                                       //同上
    
    auto func =
        std::bind(                              //C++11模拟初始化捕获
            [](const std::vector<double>& data) 
            { /*使用data*/ },
            std::move(data)                 
        );
    ```

    


## 条款 33：对于 std::forward 的 auto&& 形参使⽤ decltype

- 对于 std::forward 的 auto&& 形参使⽤ decltype。

  - C++ 14 中支持使用泛型 lambda 表达式，即我们可以在 lambda 中使用 auto 推导形参的类型。假设我们想实现一个泛型的 lambda 函数，并在其内部将 auto 推导出的形参转发给另一个函数，此时很容易就能够写出下面这样的代码：

    ```cpp
    auto f = [](auto x){ return func(normalize(x)); };
    ```

    这样的代码存在问题，因为它无法区分左值和右值，即使你传入的参数是右值，它转发后的结果却丢失了右值的属性，仍然为左值。

    为了能够支持在内部能够完美转发左值与右值，就需要将参数类型声明为 auto&&，并使用 std::forward 对参数进行转发。同时，使用 decltype 推导出 std::forward 所需要的参数的类型。

    ```cpp
    //C++14 支持多个参数并进行完美转发的泛型lambda模板
    auto f =
        [](auto&&... params)
        {
            return
                func(normalize(std::forward<decltype(params)>(params)...));
        };
    ```




## 条款 34：优先使用 lambda 表达式而非 std::bind

- 相比于 std::bind，lambda 表达式可读性更好、表达力更强、运行效率更高。

  - 这里以一段代码来举例子：

    ```cpp
    //一个时间点的类型定义（语法见条款9）
    using Time = std::chrono::steady_clock::time_point;
    
    //“enum class”见条款10
    enum class Sound { Beep, Siren, Whistle };
    
    //时间段的类型定义
    using Duration = std::chrono::steady_clock::duration;
    
    //在时间t，使用s声音响铃时长d
    void setAlarm(Time t, Sound s, Duration d);
    
    //lambda写法
    auto setSoundL =
        [](Sound s) 
        {
            //使std::chrono部件在不指定限定的情况下可用
            using namespace std::chrono;
    
            setAlarm(steady_clock::now() + hours(1),    
                     s,                                
                     seconds(30));
        };
    
    //bind写法
    using namespace std::chrono;             
    using namespace std::placeholders;
    auto setSoundB =
        std::bind(setAlarm,
                  std::bind(std::plus<steady_clock::time_point>(),
                            std::bind(steady_clock::now),
                            hours(1)),
                  _1,
                  seconds(30));
    ```

    对于以上代码：

    - **可读性**：由于参数中存在一个表达式，而 std::bind 会在调在 std::bind 时就计算出表达式的结果，并绑定在 setSoundB 对象中，而不是在调用 setAlarm 时再计算表达式结果，因此会导致结果有误差。为了解决这个问题，就需要在 std::bind 中再嵌套一个 std::bind 并将表达式改为模板 std::plus，以此将表达式求值推迟。这样带来的问题就是代码的可读性变差，理解成本增高。
    - **运行效率**：lambda 中调用 setAlarm 时，会使用 inline 对其进行优化，而 std::bind 是通过传入的函数指针调用函数，因此编译器不太可能将其内联，此时 lambda 效率更高。

- 仅在 C++ 11 中，std::bind 在实现移动捕获、使用模板化函数调用运算符来绑定对象这两个场景才能发挥作用。而在 C++  14 及以后的版本，lambda 已经完全可以替代 std::bind 了。



1. https://zh.wikipedia.org/zh-cn/GNU%E4%BE%A6%E9%94%99%E5%99%A8)
