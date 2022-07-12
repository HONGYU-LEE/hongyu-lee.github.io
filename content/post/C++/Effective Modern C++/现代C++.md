---
title: Effective Modern C++ 读书笔记（二）：现代 C++
date: 2022-07-12T13:54:13+08:00
categories:
    - C++
tags:
    - C++
    - Effective Modern C++
    - 读书笔记
---



# 现代 C++

## 条款 7：在创建对象时区分 () 与 {}

- 大括号初始化可以引用的语境最广泛，还可以阻止隐式的窄化类型转换，以及对 C++ 中令人头疼的解析语法具有免疫。

  - 大括号初始化又被称为统一初始化，因为其适用于 C++ 中所有的初始化场景：

    ```cpp
    //大括号初始化让你可以表达以前表达不出的东西，如指定一个容器中的元素
    std::vector<int> v{ 1, 3, 5 };  //v初始内容为1,3,5
    
    //大括号初始化也能被用于为非静态数据成员指定默认初始值，C++11允许"="初始化也拥有这种能力：
    class Widget{
        …
    private:
        int x{ 0 };                 //没问题，x初始值为0
        int y = 0;                  //也可以
        int z(0);                   //错误！
    }
    
    //不可拷贝的对象可以使用大括号初始化或者小括号初始化，但是不能使用=初始化：
    std::atomic<int> ai1{ 0 };      //没问题
    std::atomic<int> ai2(0);        //没问题
    std::atomic<int> ai3 = 0;       //错误！
    ```

  - 大括号初始化禁止内置类型间隐式的窄化类型转换，而小括号为了兼容老旧代码则放宽了这一限制：

    ```cpp
    double x, y, z;
    
    int sum1{ x + y + z };          //错误！double的和可能不能表示为int
    int sum2(x + y +z);             //可以（表达式的值被截为int）
    ```

  - C++ 规定任何能被解析为声明的都要被解析为声明，因此有时候想要使用默认构造函数或者全缺省构造函数创建对象时，就会被视为一个函数声明，而大括号初始化则会对此免疫：

    ```cpp
    class Widget{
        Widget(int x = 0){
            x_ = x;
        }
    
    private:
        int x_;               
    }
    
    Widget w1(10);                  //使用实参10调用Widget的一个构造函数
    Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
    Widget w3{};                    //调用没有参数的构造函数构造对象
    ```

- 在构造函数重载决议期间，大括号初始化会尽可能的与带有 std::initializer_list 类型的形参相匹配，即使其他重载版本有着更加匹配的形参。

  - ```cpp
    class Widget { 
    public:  
        Widget(int i, bool b);      
        Widget(int i, double d);   
        Widget(std::initializer_list<long double> il);      
        …
    }; 
    
    Widget w1(10, true);    //使用小括号初始化，同之前一样调用第一个构造函数
    
    Widget w2{10, true};    //使用花括号初始化，但是现在
                            //调用std::initializer_list版本构造函数
                            //(10 和 true 转化为long double)
    
    Widget w3(10, 5.0);     //使用小括号初始化，同之前一样调用第二个构造函数 
    
    Widget w4{10, 5.0};     //使用花括号初始化，但是现在
                            //调用std::initializer_list版本构造函数
                            //(10 和 5.0 转化为long double)
    ```

- () 和 {} 在有些情况下结果可能大相径庭。

  - 大括号在初始化时会调用 std::initializer_list 构造函数，而小括号则调用非 std::initializer_list 构造函数：

    ```cpp
    std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                    //创建一个包含10个元素的std::vector，
                                    //所有的元素的值都是20
    std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                    //创建包含两个元素的std::vector，
                                    //元素的值为10和20
    ```

- 在模板类创建对象时，选择使用小括号和大括号是一个棘手的问题。

  - ```cpp
    template<typename T,            //要创建的对象类型
             typename... Ts>        //要使用的实参的类型
    void doSomeWork(Ts&&... params)
    {
        create local T object from params...
        …
    } 
    
    T localObjectA(std::forward<Ts>(params)...);             //使用小括号
    T localObjectB{std::forward<Ts>(params)...};             //使用大括号
    
    //假设以下面这样的参数进行调用，此时就会如上一条一样，如果使用小括号则调用非std::initializer_list构造函数，而使用大括号则调用std::initializer_list构造函数，出现两种结果。
    std::vector<int> v; 
    …
    doSomeWork<std::vector<int>>(10, 20);
    ```

    这也是 std::make_unique 与 std::make_shared 面临的问题，它们的解决方法是是使用小括号，并将这个决定记录在文档中，作为接口的一部分。




## 条款 8：优先选用 nullptr，而非 0 或 NULL

- 相对于 0 或者 NULL，优先选用 nullptr。

  - 在 C++ 中，0 是一个 int 而不是指针，同样 NULL 也只是一个整数类型（根据编译器的实现不同，可能会为 int 或 long）。它们都是整数类型而不是指针类型，如果采用 C++ 98 的写法使用它们作为空指针，可能会出现未定义的行为。

- 避免在整型和指针类型之间重载。

  - ```cpp
    void f(int);        //三个f的重载函数
    void f(bool);
    void f(void*);
    
    f(0);               //调用f(int)而不是f(void*)
    
    f(NULL);            //可能不会被编译，一般来说调用f(int)，
                        //绝对不会调用f(void*)
    
    f(nullptr);         //调用重载函数f的f(void*)版本
    ```




## 条款 9：优先选用 using 而非 typedef

- typedef 不支持模板化，但别名声明支持。

  - 假设我们需要定义一个链表，并且这个链表使用了我们自定义的内存分配器：

    ```cpp
    //别名模板
    template<typename T>                            //MyAllocList<T>是
    using MyAllocListA = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                    //的同义词
    
    //如果使用typedef，则只能在一个模板类中用typedef来为该类型起别名
    template<typename T>                            //MyAllocList<T>是
    struct MyAllocListB {                            //std::list<T, MyAlloc<T>>
        typedef std::list<T, MyAlloc<T>> type;      //的同义词  
    };
    ```

- 别名模板可以免写  [::类型名]  后缀，并且不会像 typedef 在模板中使用时还需要增加 typename 前缀。

  - 由于 typedef 是模板类中的某个类型的别名，因此还需要在作用域后面加上 [::类型名] 后缀 ：

    ```cpp
    MyAllocListA<Widget> lw;   	   
    MyAllocListB<Widget>::type lw; //typedef需要加上::type           
    ```

  - C++ 中规定，在使用依赖类型时必须要加上 typename 前缀，而 MyAllocListB\<T\>::type 依赖于模板参数 T，因此它是一个依赖类型。而别名模板 MyAllocListA\<T\> 是一个类型的名称，因此它是一个非依赖类型。

    ```cpp
    //typedef需要加上typename来告诉编译器这是一个类型的别名
    template<typename T>
    class Widget {                              //Widget<T>含有一个
    private:                                    //MyAllocLIst<T>对象
        typename MyAllocList<T>::type list;     //作为数据成员
        …
    }; 
    
    //而using声明的别名模板则不需要
    template<typename T> 
    class Widget {
    private:
        MyAllocList<T> list;                        //没有“typename”
    };
    ```

    为什么要有这个规定呢？因为对于编译器来说，它无法判断 MyAllocListB\<T\>::type 究竟代表的是一个类型的名字，还是一个变量的名字，又或是别的一些东西。




## 条款 10：优先选用限定作用域的枚举而非未限定作用域的枚举

- C++ 98 风格的枚举类型被称为未限定作用域的枚举。

  - ```cpp
    enum Color { black, white, red };       //C++98 未限域枚举
    enum class Color { black, white, red }; //C++11 限域枚举类
    ```

- 限定作用域的枚举类型仅能在枚举类型中可见。它们只能通过强制转换才可以转换到别的类型。

  - 未限域枚举会将枚举名泄露，从而导致命名污染，而限域枚举类仅能在枚举类型中可见。

    ```cpp
    //未限域枚举会导致枚举名泄露
    enum Color { black, white, red };   //black, white, red在
                                        //Color所在的作用域
    auto white = false;                 //错误! white早已在这个作用
                                        //域中声明
    
    //限域枚举类仅能在枚举类型中可见
    enum class Color { black, white, red }; //black, white, red
                                            //限制在Color域内
    auto white = false;                     //没问题，域内没有其他“white”
    
    Color c = white;                        //错误，域中没有枚举名叫white
    
    Color c = Color::white;                 //没问题
    auto c = Color::white;                  //也没问题（也符合Item5的建议）
    ```

  - 未限域枚举可能会隐式转换为整数类型或者浮点数类型，而限域枚举类则不会。

    ```cpp
    enum Color { black, white, red };       //未限域枚举  
    
    std::vector<std::size_t>             
      primeFactors(std::size_t x);
    
    Color c = red;
    …
    
    //未限域枚举在比较时会隐式类型转换
    if (c < 14.5) {
        auto factors =                   
          primeFactors(c);
        …
    }
    
    enum class Color { black, white, red }; //限域枚举类
    
    Color c = Color::red;                   
    ...                                   
    
    //只能通过强制类型转换
    if (static_cast<double>(c) < 14.5) {                                       
        auto factors =                                
          primeFactors(static_cast<std::size_t>(c));    
        …
    }
    ```

- 两种枚举类型都支持执行底层的存储类型。限定作用域的枚举类型默认是 int，而不限范围的枚举类型则没有默认底层类型。

  - ```cpp
    //默认情况下，限域枚举的底层类型是int：
    enum class Status;                  //底层类型是int
    
    //如果默认的int不适用，你可以重写它：
    enum class Status: std::uint32_t;   //Status的底层类型
                                        //是std::uint32_t
                                        //（需要包含 <cstdint>）
    
    //为了高效使用内存，编译器通常在确保能包含所有枚举值的前提下为enum选择一个最小的底层类型。在一些情况下，编译器将会优化速度，舍弃大小，这种情况下它可能不会选择最小的底层类型，而是选择对优化大小有帮助的类型。
    
    //只需要表示三个值，底层类型是int
    enum Color { black, white, red };   
    
    //这里值的范围从0到0xFFFFFFFF，编译器会选择一个能表示这个范围的的整型类型
    enum Status { good = 0,
                  failed = 1,
                  incomplete = 100,
                  corrupt = 200,
                  indeterminate = 0xFFFFFFFF
                };
    ```

- 限定作用域的枚举类型总是可以前置声明，而未限定作用域的枚举只有在制定了默认底层类型的时候才可以前置声明。

  - ```cpp
    enum Color;         //错误！
    enum class Color;   //没问题
    
    //当指定默认底层类型时，才可以前置声明
    enum Color: std::uint8_t;   //非限域enum前向声明
                                //底层类型为
                                //std::uint8_t
    ```

    




## 条款 11：优先使用 delete 函数而非 private 未定义函数

- 优先选择使用 delete 而非 private。

  - ```cpp
    // C++98写法，此时仍然可以通过成员函数或者类的友元调用它们，由于没有函数定义，在链接时会报错
    template <class charT, class traits = char_traits<charT> >
    class basic_ios : public ios_base {
    public:
        …
    
    private:
        basic_ios(const basic_ios& );           // not defined
        basic_ios& operator=(const basic_ios&); // not defined
    };
    
    // C++11写法，无法通过任何途径调用这些函数
    template <class charT, class traits = char_traits<charT> >
    class basic_ios : public ios_base {
    public:
        …
    
        basic_ios(const basic_ios& ) = delete;
        basic_ios& operator=(const basic_ios&) = delete;
        …
    };
    ```

- 任何函数都可以删除，包括非成员函数和模板实例。

  - 由于 C++ 继承了 C 的隐式类型转换，所有的数值类型在传参给一个参数为 int 的函数时都能够隐式转换为 int，为了避免这样的操作带来的一些错误行为，delete 允许删除非成员函数，因此我们可以 delete 它们的重载函数：

    ```cpp
    bool isLucky(int number);       //原始版本
    bool isLucky(char) = delete;    //拒绝char
    bool isLucky(bool) = delete;    //拒绝bool
    bool isLucky(double) = delete;  //拒绝float和double
    
    if (isLucky('a')) …     //错误！调用deleted函数
    if (isLucky(true)) …    //错误！
    if (isLucky(3.5f)) …    //错误！
    ```

  - delete 同样也能删除某个特化的模板类，甚至是类内部的模板函数的特化版本，这都是 private 无法做到的：

    ```cpp
    template<typename T>
    void processPointer(T* ptr);
    
    //删除void*和char*的特化版本
    template<>
    void processPointer<void>(void*) = delete;
    
    template<>
    void processPointer<char>(char*) = delete;
    
    //删除类内部的模板函数的某个特化版本
    class Widget {
    public:
        …
        template<typename T>
        void processPointer(T* ptr)
        { … }
        …
    };
    
    template<>                                          //还是public，
    void Widget::processPointer<void>(void*) = delete;  //但是已经被删除了
    ```



## 条款 12：为重写函数添加 override 

- 为需要重写的函数加上 override。

  - ```cpp
    class Base {
    public:
        virtual void mf1() const;
    };
    
    class Derived: public Base {
    public:
        virtual void mf1() const override;
    }; 
    ```

    override 用于确保派生类虚函数是否构成重写，当不构成重写时会编译错误。

- 成员函数引用饰词可以让我们区别对待左值和右值对象（针对 *this）。

  - ```cpp
    class Widget {
    public:
        using DataType = std::vector<double>;
        …
        DataType& data() &              //对于左值Widgets,
        { return values; }              //返回左值
        
        DataType data() &&              //对于右值Widgets,
        { return std::move(values); }   //返回右值
        …
    
    private:
        DataType values;
    };
    
    auto vals1 = w.data();              //调用左值重载版本的Widget::data，
                                        //拷贝构造vals1
    auto vals2 = makeWidget().data();   //调用右值重载版本的Widget::data, 
                                        //移动构造vals2
    ```




## 条款 13：优先选用 const_iterator 而非 iterator

- 优先选用 const_iterator 而非 iterator（对于不需要修改的操作，尽可能收缩权限，避免出现意外）。

- 若要实现通用的代码，优先选用非成员函数版本的 `begin`、`end` 和 `rbegin` 等，而不是成员函数版本。

  - 比起成员函数版本，非成员函数版本更加的通用，因为其能够支持原生数组，以及一些以自由函数提供接口的第三方库，例如下面这个通用版本的 findAndInsert：

    ```cpp
    template<typename C, typename V>
    void findAndInsert(C& container,            //在容器中查找第一次
                       const V& targetVal,      //出现targetVal的位置，
                       const V& insertVal)      //然后在那插入insertVal
    {
        using std::cbegin;
        using std::cend;
    
        auto it = std::find(cbegin(container),  //非成员函数cbegin
                            cend(container),    //非成员函数cend
                            targetVal);
        container.insert(it, insertVal);
    }
    ```




## 条款 14：如果函数不抛出异常，就使用 noexcept

- noexcept 是函数接口的一部分，这意味着调用方可能对它有依赖。

- 带有 noexcept 的函数更容易优化。 

  - 在运行期时，如果一个异常跳出函数的作用域，则说明函数的异常说明被违反。在 C++ 98 的异常说明中，调用栈会展开至函数的调用者，并在执行一系列动作后将程序终止。而 C++ 11 异常说明的运行时行为又有所不同，调用栈只是可能在程序终止前展开。

  - 展开调用栈和可能展开调用栈两者对于代码生成有很大的影响。在带有 noexcept 声明的函数中，优化器不需要在异常传出函数的前提下，保证运行时栈处于可展开的状态；也不需要在异常离开函数时保证 noexcept 函数中的对象按照构造的逆序析构。而那些以 throw() 异常声明的函数则享受不到这样的优化灵活性，和那些没有加上异常声明的函数一样。

    ```cpp
    RetType function(params) noexcept;  //C++11写法 极尽所能优化 
    RetType function(params) throw();   //C++98写法 较少优化
    RetType function(params);           //较少优化
    ```

- noexcept 对于移动操作、swap、资源释放函数和析构函数非常有用。

- 大多数函数都是异常中立的，不具备 noexcept。

  - 这类函数自身并不抛出异常，但是它们所调用的函数则可能会抛出异常。为了处理这种情况，异常中立的函数会允许那些抛出的异常就由它传至调用栈中更深的一层，直到遇到异常处理的程序。




## 条款 15：尽可能使用 constexpr

- constexpr 对象都具备 const 属性，并由编译期已知的值完成初始化。

  - ```cpp
    int sz;                             //non-constexpr变量
    …
    constexpr auto arraySize1 = sz;     //错误！sz的值在编译期不可知
    
    std::array<int, sz> data1;          //错误！一样的问题
    
    constexpr auto arraySize2 = 10;     //没问题，10是编译期可知常量
    
    std::array<int, arraySize2> data2;  //没问题, arraySize2是constexpr
    ```

  - 注意：const 不提供 constexpr 所能保证之事，因为 const 对象不需要在编译期初始化它的值。

    ```cpp
    int sz;                            //和之前一样
    …
    const auto arraySize = sz;         //没问题，arraySize是sz的const复制
    
    std::array<int, arraySize> data;   //错误，arraySize值在编译期不可知
    ```

    简而言之，所有 constexpr 对象都是 const，但不是所有 const 对象都是 constexpr。

- constexpr 函数在调用时如果传入的实参也是编译期已知的，则会产出编译期结果。

  - 当 constexpr 函数接受的参数为编译期已知的，则会在编译期间计算出结果。而如果参数是编译期未知的，则它的运作与普通函数一样，在运行期去执行计算。

    ```cpp
    constexpr int pow(int base, int exp) noexcept         //constexpr函数
    {
        //C++11写法，constexpr 函数的代码不超过一行语句
    	// return (exp == 0 ? 1 : base * pow(base, exp - 1));   
    
        //C++14放宽了限制
        auto result = 1;
        for (int i = 0; i < exp; ++i) result *= base;
        
        return result;
    }
    
    //传入编译器实参，生成编译器结果
    constexpr auto numConds = 5;               
    std::array<int, pow(3, numConds)> results;  
    
    //传入运行时参数，则在运行时进行计算
    auto base = readFromDB("base");     //运行时获取这些值
    auto exp = readFromDB("exponent"); 
    auto baseToExp = pow(base, exp);    //运行时调用pow函数
    ```

- 比起非 constexpr 的对象和函数，constexpr 可以用在作用域更广的语境（编译期、运行期）中。



## 条款 16：保证 const 成员函数的线程安全

- 保证 const 成员函数的线程安全性，除非可以确定他们不会用在并发语境中。
  - const 本身代表了只读的语义，但也有一些例外情况，例如存在 mutable 变量时，这些变量能够突破 const 的限制，此时在并发环境下就会出现数据竞争的情况。

- std::atmoic 类型的变量会比使用 std::mutex 性能更佳，但前者仅适用于单个变量或者内存区域的操作。



## 条款 17：特殊成员函数的生成机制

- 特殊成员函数指的是 C++ 中会自动生成的成员函数，如：默认构造函数、析构函数、拷贝操作、移动操作。
- 移动操作只有在类中未包含用户显式声明的拷贝操作、移动操作和析构函数时才会生成。
- 拷贝构造函数和拷贝赋值运算符仅会在用户没有显式声明它们的时候才会自动生成，且如果该类声明了它们的移动版本，则拷贝版本会被 delete。在新版本的 C++ 中，显式声明析构函数后，自动生成拷贝操作是被废弃的行为。
- 成员函数模板在任何情况下都不会抑制特殊成员函数的生成。

