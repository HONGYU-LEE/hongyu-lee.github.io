---
title: Effective Modern C++ 读书笔记（一）：类型推导与 auto
date: 2022-07-12T12:54:13+08:00
categories:
    - C++
tags:
    - C++
    - Effective Modern C++
    - 读书笔记
---

# 类型推导

## 条款 1：模板类型推导

- 在模板类型推导时，具有引用类型的实参会被视为非引用类型，即引用会被忽视。

  - ```cpp
    template<typename T> 
    void f(T & param);
    
    int x = 27; //x是int 
    const int cx = x; //cx是const int 
    const int & rx = cx; //rx是指向const int的引⽤
    
    f(x); //T是int，param的类型是int& 
    f(cx); //T是const int，param的类型是const int & 
    f(rx); //T是const int，param的类型是const int &
    ```

- 对万能引用形参进行推导时，左值参数会特殊处理。

  - 左值：T 和 paramType 都为左值引用。
  - 右值：与第一条原则相同，即 paramType 为右值引用，T 为非引用类型。

- 对按值传递的形参进行推导时，若实参类型中带有 const 或 volatile 饰词，会将其忽视，当作不带这些饰词的类型。

- 在模板类型推导时，数组或者函数类型的实参会退化成对应的指针，除非它们用来初始化引用。

  - ```cpp
    void someFunc(int, double);         //someFunc是一个函数，
                                        //类型是void(int, double)
    template<typename T>
    void f1(T param);                   //传值给f1
    
    template<typename T>
    void f2(T & param);                 //传引用给f2
    
    f1(someFunc);                       //param被推导为指向函数的指针，
                                        //类型是void(*)(int, double)
    f2(someFunc);                       //param被推导为指向函数的引用，
                                        //类型是void(&)(int, double)
    ```



## 条款 2：auto 类型推导

- 在正常情况下，auto 类型推导和模板类型推导的结果是一样的。但是也有例外，当我们使用大括号括起的初始化表达式时，auto 会将其推导为 std::initializer_list，而模板推导则不会。

  - ```cpp
    auto x = { 11, 23, 9 };         //x的类型是std::initializer_list<int>
    
    template<typename T>            //带有与x的声明等价的
    void f(T param);                //形参声明的模板
    
    f({ 11, 23, 9 });               //错误！不能推导出T
    ```

    若是想要推导出 T 的类型，则需要将模板中的 param 指定为 std::initializer_list\<T\> ：

    ```cpp
    template<typename T>
    void f(std::initializer_list<T> initList);
    
    f({ 11, 23, 9 });               //T被推导为int，initList的类型为
                                    //std::initializer_list<int>
    ```

- 在函数返回值或 lambda 表达式中使用 auto 时，使用模板类型推导而非 auto 类型推导。

  - 自从 C++ 14 开始允许使用 auto 来推导函数返回值，但是其虽然是 auto，但是实际上确实模板类型推导。我们怎样确认呢？可以尝试返回一个大括号括起的初始化表达式。

    ```cpp
    auto createInitList()
    {
        return { 1, 2, 3 };         //错误！不能推导{ 1, 2, 3 }的类型
    }
    
    std::vector<int> v;
    …
    auto resetV = 
        [&v](const auto& newValue){ v = newValue; };        //C++14
    …
    resetV({ 1, 2, 3 });            //错误！不能推导{ 1, 2, 3 }的类型
    ```

    

##  条款 3：decltype

- 通常 decltype 会得出变量的类型并且不做任何修改。

  - ```cpp
    const int i = 0;                //decltype(i)是const int
    
    bool f(const Widget& w);        //decltype(w)是const Widget&
                                    //decltype(f)是bool(const Widget&)
    
    struct Point{
        int x,y;                    //decltype(Point::x)是int
    };                              //decltype(Point::y)是int
    
    Widget w;                       //decltype(w)是Widget
    
    if (f(w))…                      //decltype(f(w))是bool
    
    template<typename T>            //std::vector的简化版本
    class vector{
    public:
        …
        T& operator[](std::size_t index);
        …
    };
    
    vector<int> v;                  //decltype(v)是vector<int>
    …
    if (v[0] == 0)…                 //decltype(v[0])是int&
    ```

- 对于类型为 T 的左值表达式，decltype 总是推导出类型 T&。

  - ```cpp
    decltype(auto) f1()
    {
        int x = 0;
        …
        return x;                            //decltype(x）是int，所以f1返回int
    }
    
    decltype(auto) f2()
    {
        int x = 0;
        return (x);                          //decltype((x))是int&，所以f2返回int&
    }
    ```

- C++ 14 支持 decltype(auto)，与 auto 一样，它会依据初始化表达式来推导类型，但它使用的是 decltype 的规则。

  - ```cpp
    Widget w;
    
    const Widget& cw = w;
    
    auto myWidget1 = cw;                    //auto类型推导
                                            //myWidget1的类型为Widget
    decltype(auto) myWidget2 = cw;          //decltype类型推导
                                            //myWidget2的类型是const Widget&
    ```




## 条款 4：查看类型推导结果的方法

- 利用 IDE 编辑器、编译器错误消息 和 Boost.TypeIndex 库通常能够查看到推导的类型。

- 工具判断的结果可能不准确，因此理解 C++ 类型推导规则是必要的。


​	

# auto

## 条款 5：优先选用 auto，而非显式类型声明

- auto 变量必须初始化，它通常可以避免⼀些移植性和效率性的问题，也使得重构更⽅便，比显式指定类型更加简洁。

  - ```cpp
    //大大的简化了代码
    //使用auto
    auto derefUPLess = 
        [](const std::unique_ptr<Widget> &p1,      
           const std::unique_ptr<Widget> &p2)      
        { return *p1 < *p2; };     
    
    //显式执行类型
    std::function<bool(const std::unique_ptr<Widget> &,
                       const std::unique_ptr<Widget> &)>
    derefUPLess = [](const std::unique_ptr<Widget> &p1,
                     const std::unique_ptr<Widget> &p2)
                    { return *p1 < *p2; };
    
    //可能会避开一些潜在的风险
    unordered_map<std::string, int> m;
    
    //看起来没有问题，但是实际unordered_map的key是cosnt std::string
    //类型不匹配，虽然没有报错，但是编译器会将m的值拷贝到一个临时变量中，再将临时变量的引用绑定到p上，且在循环结束后再将这个临时变量销毁，增加了大量拷贝、销毁资源的成本
    for(const std::pair<std::string, int>& p : m)
    {
        …                                   //用p做一些事
    }
    
    for(const auto& p : m)
    {
        …                                 
    }
    ```

- auto 类型的变量有着条款 2 和 条款 6 中描述的毛病。



## 条款 6：当 auto 推导的类型不符合要求时，使用显式类型初始化

- 隐形的代理类可能会导致 auto 根据初始化表达式推导出错误的类型。

  - ```cpp
    std::vector<bool> features(const Widget& w);
    Widget w;
    …
        
    bool var_a = features(w)[5];     //bool，存在隐式类型转换
    auto vat_b = features(w)[5];     //代理类型std::vector<bool>::reference，如果不明确类型则可能导致未定义的行为。
    
    
    ```

    对于 std::vector\<bool\> 来说，为了节省空间，底层存储时使用了 1 bit 而非 1 byte 的 bool 类型来存储数据。由于 C++ 禁止对 bits 的引用，因此当我们调用 `operator[]` 时返回的其实是一个代理类型 std::vector\<bool\>::reference，其能够模拟 bool& 的行为。

- 显式类型的初始化惯用法会强制 auto 推导出想要的类型。

  - ```cpp
    double calcEpsilon();                           //返回公差值
    
    float ep = calcEpsilon();                       //double到float隐式转换
    auto ep = static_cast<float>(calcEpsilon());	//显式类型初始器惯用法
    ```

