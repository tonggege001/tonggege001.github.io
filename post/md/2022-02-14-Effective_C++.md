---
layout: post  
title: 2022-02-14-Effective C++
date: 2022-02-14
categories: blog
tags: [C++]
description: 2022-02-14-Effective C++ 学习笔记
---

# 1. C++是一个语言联邦  
C++是一个语言联邦，包含四个主要的次语言：  
* C
* Object-Oriented C++，也就是C with classes, 包括构造函数、析构函数封装、继承、多态、virtual函数(动态绑定)
* Template C++，泛型编程，也就是模版元编程(Template MetaProgramming)
* STL，STL是一个Template程序库，它对容器、迭代器(iterator)、算法(algorithm)、函数对象(function object，也就是可调用对象)的规约有极佳的紧密配合

C++的四个层次有不同的规范，C++搞笑编程守则视状况而变化，取决于使用C++的哪一部分。

# 2. 尽量以const、enum、inline 替换 #define
因为#define不被视为语言的一部分，那么可能编译器开始处理源码之前它就被预处理器
移走了。比如`#define ASPECT_RATIO 1.653`，当出现bug后，它不会显示ASPECT_RATIO，而只会显示1.653，追踪bug时便很难确定。

```C++
const int* const authorName = "Scott Meyers";
第一个const： 底层const，authorName指向的内容不能更改
第二个const： 顶层const，authorName变量不能更改
```  

**声明**类的静态成员变量（在.h文件内）：  
```C++  
class GamePlayer {
private:
    static const int NumTurns = 5;      // 常量声明式
    int scores[NumTurns];               // 使用该常量
}
```
**定义**类的静态成员变量（在.cpp文件内）：
```C++
const int GamePlayer::NumTurns;             // NumTurns的定义
```
如果class专属常量且是static的整数类型，如果不取地址则不需要提供定义式，否则需要提供定义式。
初值可以在声明中提供，也可以在定义中提供。

# 3. const用法  
* 修饰常量、文件、函数、区块作用域中被声明为static的对象、内部static和non-static成员变量。

## 令函数返回一个常量会有效避免一些意外情况： 

```C++
const Rational operator * (const Rational & lhs, const Rational & rhs);
```  
上面的运算符重载可以有效避免下面的情况：  
```C++
Rational a,b,c;
...
if(a * b = c){      // 想做比较，但是笔误写成了赋值，这样const返回值就能有效报错

}
```

## 修饰成员函数  
将const实施在成员函数的目的是为了**确认该成员函数可用作于const对象身上**，而且能够确定class接口的该函数可以改动对象内容或者函数不可以改动对象内容。

## 两个成员函数如果只是常量性不同，可以被重载
```C++
class TextBlock {
public:
    const char & operator[](std::size_t position) const{}
    char & operator[](std::size_t position) {}
}
```

## mutable 允许成员变量在 `const成员函数` 内被修改  
```C++  
class Text {
private:
    mutable std::size_t textLength;

    std::size_t Text::length const {
        this->textLength = 10;
    }
}
```

## const_cast 移出常量修饰符变成变量修饰
const_cast能够将指针或者引用的常量修饰符移出，对原常量指针进行修改，则常量指针所指的内容和原始常量的结果会不一样，原始常量永远不会修改。

但是它们的地址不会发生任何变化。

const_cast的转换类型必须是指针或者引用，而不能是常量本身。

```C++
int main(){
    const int a = 5;
    const int * b = &a;
    cout<<"before cast: "<<endl;
    cout<<"a: "<<a<<", *b: "<<*b<<endl;
    cout<<"after cast: "<<endl;

    int * c = const_cast<int*>(b);
    *c = 6;
    cout<<"a: "<<a<<", *b: "<<*b<<", *c: "<<*c<<endl;
    return 0;
/*
    before cast: 
    a: 5, *b: 5
    after cast: 
    a: 5, *b: 6, *c: 6
*/
}
```

### 常用例子：non-const调用const成员函数以减少代码量，反过来绝对不可以  
>> 因为const成员函数保证了不会修改成员变量，但是在const内调用non-const打破了这一规则。

减少代码量例子：《Effective C++》 Page24

# 4. 确定对象使用前已经被初始化  
内置类型手动初始化，对象类型在进入构造函数前根据初始化列表初始化。  

初始化列表初始化顺序：是根据成员的声明顺序**而不是**初始化列表从左到右的顺序。  

如果成员是const 或references，则它们一定要被初始化，而不是在构造函数中被赋值。

## static修饰符  
static可以被定义在文件内、namespace内、classes内、函数内。函数内的static对象称为local static对象，其他的是non-local对象。程序结束时static对象会自己释放。

函数内的static函数，第一次调用的该函数的时候，会初始化内部static变量，后面调用的时候不会再初始化。

# 5. C++自动生成的函数
一个空类被C++自动生成一个copy构造函数（复制构造函数）、复制运算符（copy assignment）和一个析构函数。如果没有声明任何构造函数则会声明一个默认构造函数。  

对于类中的每一个成员变量，如果是class那么会调用其copy构造函数，如果是内置类型（比如int），则会复制每一个bit。若成员变量不包含复制构造函数（`A(const A &) = 0`），则编译失败。

内含const或者reference的成员，则必须自己定义copy构造函数和copy 运算符。

# 6. 阻止自动生成函数方法
1. 将copy构造函数、copy运算符等声明为private。缺点：成员函数和友元还是能够访问到。
2. 声明一个Uncopyable类，让其继承。Uncopyable类中将复制构造函数和复制运算符声明为private。
3. 令复制构造函数 = delete, 如：`public: A() = delete;`

# 7. 基类析构函数一般声明为virtual
* 因为基类指针如果指向一个派生类的对象，则析构的时候可能会出现局部销毁现象。

* 任何class只要带有virtual函数都几乎确定也有一个virtual 析构函数。
* 如果class不含有virtual，一般其析构函数不为virtual，因为出现virtual后，对象就会携带vtbl指针(虚函数入口地址表指针)。

# 8. 别让异常逃离析构函数

如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数）中执行该操作，比如close函数。

# 9. 不要在析构和构造函数中调用virtual函数  
> 注意：这一点C++与Java不同！  

因为基类的构造函数一定先于派生类的构造函数调用，那么在基类的构造函数中调用virtual函数func，本意是要调用派生类的func，然而派生类还未初始化，实际上调用的是基类的func。  

在基类的构造函数中还未下降到派生类里，所以不允许使用未初始化的成员。  

正确方法：在派生类的构造函数的初始化列表中，初始化基类，并把信息传递给基类，使其在基类中用到派生类的信息，例如：  
```C++  
class Base {
public: 
    explicit Base(param_b){
        ...
        // use information from derived class
    }
}

class Derived: public Base {
public:
        // pass information to base class
        // 先初始化基类，然后再初始化派生类
    Derived(param_d): Base(param_b){
        ...
    }
}
```  

# 10. 令operator= 返回一个reference to *this  
这是一个协议，也可以不遵守，但是建议遵守。  
```C++  
Widget & operator=(const Widget & rhs){
    ...
    return *this;
}
```  

# 11. operator= 中处理自我赋值  
`a[i] = a[j]     // i == j`
或者  
`w = w`

出现安全问题：  
```C++
Widget & operator=(const Widget & rhs){
    delete this->pointer;
    // 在自我赋值中，下面代码实际上rhs就是this，已经被释放了，但是还是用到了该pointer
    this->pointer = new Class(*rhs.pointer);     
    return * this;
}
```

解决方法有两种：
1. 重新排列代码或者提前判断二者是否等价
2. 先生成一个临时变量，使用移动语义交换内容，然后赋值  

方法一：  
```C++  
Widget & operator=(const Widget & rhs){
    Data * pOrig = this->pointer;
    this->pointer = new Data(*rhs.pointer);
    delete pOrig;
    return *this;
}
```

方法二：  
```C++  
Widget & operator=(const Widget & rhs){
    Widget temp(rhs);
    swap(temp);             // 创建一个交换函数
    return *this;
}
```

# 12. 复制对象时勿忘其每一个成分
当自己编写复制构造函数和复制运算符时，必须确保所有的成员变量都被复制，所有的基类都调用了复制构造函数初始化。  

如果基类没调用复制构造函数初始化，则会调用默认构造函数，此时基类就没有被复制过来。  

错误代码：
```C++  
class Derived: public Base {
private:
    int data;
public:
    Derived(const Derived & rhs){

    }
    Derived & operator=(const Derived & rhs){
        
    }
}
```  

正确代码：
```C++
class Derived: public Base {
private:
    int data;
public:
    Derived(const Derived & rhs){

    }
    Derived & operator=(const Derived & rhs){
        
    }
}
```  

# 13. 以对象管理资源  

旧有的资源管理方式：
```C++
void f(){
    Investment * pInv = createInvestment();
    ...
    delete pInv;
}
```  
缺点：...处如果提前return、出现异常，并且没有多次delete，那么资源就不会被释放。  

资源获得时机便是初始化时机(RAII)，运用析构函数确保资源被释放，用对象的析构来管理资源：  
```C++
void f(){
    std::shared_ptr<Investment> pInv(createInvestment());
}
```

智能指针调用的是`delete`而不是`delete[]`，所以一般使用vector或者array管理数组，而如果是非要使用`delete[]`，可以考虑使用Boost库的scoped_array和shared_array。


# 14. 在资源管理类中小心复制行为  
如果C++管理锁，有缺陷的版本：  
```C++  
class Lock {
public:
    explicit Lock(Mutex * pm)
    : mutexPtr(pm)
    {   
        lock(mutexPtr);
    }
    ~Lock(){unlock(mutexPtr);}
private:
    Mutex * mutexPtr;
}

// 调用时
Mutex m;
Lock m1(&m);
```

上面的问题是，如果用户调用了自动生成的复制构造函数或复制运算符：  
```C++
Lock ml1(&m);
Lock ml2(&ml1);
```
实际上上面只会调用一次lock，调用两次unlock。  
解决方法是：
1. 禁止复制
2. 对底层的资源也实现引用计数法，利用RAII class即可办到，例如下面的代码则只会调用一次lock，调用一次lock：  
```C++  
class Lock {
public:
    explicit Lock(Mutex * pm)
    :mutexPtr(pm, unlock)   // 以unlock作为删除器，share_ptr会自动调用
    {
        lock(mutexPtr.get());
    }

private:
    std::shared_ptr<Mutex> mutexPtr;
}
```

上面的代码Lock不再声明析构函数，而用一个RAII的shared_ptr管理计算引用计数，当mutexPtr被析构的时候，如果引用计数为0时则调用unlock。

# 15. 在资源管理类中提供对原始资源的访问
方法一：提供一个get()函数接口  
方法二：提供隐式类型转换。  
两个方法一个需要显式调用，降低bug风险，但是代码不易阅读，另一个相反。

隐式转换的两个方法：隐式构造函数和隐式类型转换运算符。  
```C++  
class A{
public:
    A(int a){}       // 只有一个实参并且不增加explicit关键字，则是隐式构造函数

    operator A(int a){} // 隐式类型转换 
}
```

# 16. new 和 delete一起出现，new typename[] 和 delete[] typename一起出现

不要对数组进行new，尽量使用vector。例子：  
```C++  
typedef string T[4];            // 类型T表示有4个元素的字符串数组

string * pointer = new T;

delete pointer;                 // 错误，只能释放一个string
delete[] pointer;               // 正确
```

# 17. 以独立的语句将new出来的对象放进智能指针  
```C++  
func(shared_ptr<Obj>(new Obj), other_func());           // 不推荐

shared_ptr<Obj> pt(new obj);                            // 推荐
func(pt, other_func());
```  
这是因为func的压栈调用顺序是不确定的，未定义的。所以如果先调用new Obj，然后调用other_func，最后调用shared_ptr的构造函数，则很可能
在other_func()出现异常，但是new Obj没有放进智能指针。此时就会出现内存泄漏。

# 18. 让接口容易被正确使用

# 19. 设计class要像设计type一样

# 20.宁以pass-by-reference-to-const 替换 pass-by-value
因为 pass-by-value 会调用复制构造函数。对于内置类型、迭代器和函数对象用pass-by-value，其余全部用pass-by-reference-to-const。  

C++ 参数切割问题：  
C++ virtual 的多态只能由指针或引用实现，不能由实参实现，也就是当把派生类函数传参给基类时，会将派生类的基类部分切下来传递，而不会把整个派生类传递。所以会出现下列问题：  
```C++
class A {
public:
    A(){};
    virtual void display(){
        cout<<"A: display()"<<endl;
    }

};

class B: public A{
public:
    B(){};
    virtual void display(){
        cout<<"B: display()"<<endl;
    }
};

void print(A a){
    a.display();
}

int main(){
    A a;
    B b;
    print(b);
}
```
上面打印输出的是：`A: display()`，但是改成：  
```C++
void display(const A & a){
    a.display()
}
```
此时才正确调用派生类的display，结果是：`B: display()`

# 21. 返回对象或返回引用要仔细斟酌  
比如`<<`符号返回引用 *this 是合理的，但是符号`*`, `-`等等，只能返回对象，不能返回引用。

# 22. 成员变量一般声明为 private

# 23. 用非成员函数非友元函数来代替成员函数，增加封装性

# 24. 如果所有参数都需要类型转换，请采用non-member函数  
一个 自定义类型 乘法的例子：  
```C++
class Rational {
public:
    Rational(int numerator = 0, int denominator=1){}
    int numerator() const;
    int denominator() const;
private:
    ...
}

// 如果乘法符号* 声明为成员函数，则2 * Rational(3) 不能编译通过
// 因此可以将其声明为non-member
// 是否声明为友元函数取决于lhs和rhs是否用到了private变量
const Rational operator *(const Rational & lhs, const Rational & rhs){

}
```
上面的函数可以编译通过`2 * Rational(3), Rational(4) * 5`两个语句。

# 25. 特例化一个模板  
一般的模板：  
```C++
namespace std {
    template<typename T>
    void swap(T & a, T & b){
        T temp(a);
        a = b;
        b = temp;
    }
}
```

对Widget模板进行特例化：  
`template<>`表示swap是一个全特化版本，函数名称后面`<Widget>`表示这一特化版本是针对T是Widget设计的。
```C++
namespace std{
    template<>
    void swap<Widget>(Widget & a, Widget & b){
        swap(a.pImpl, b.pImpl);
    }
}
```  

一般是在类里提供一个swap成员函数，在名称空间内再提供一个swap非成员函数，然后用户调用非成员函数的swap，非成员函数内的swap调用成员函数的swap。

# 26. 尽可能延后变量定义式的出现时间  
```C++
// 不好
T func(){
    T t();
    if(xxx){
        throw (xxx);
    }

    return t;
}

// 好
T func(){
    if(xxx){
        throw (xxx);
    }
    T t = xxx;
    return t;
}
```

# 27. 类型转换容易出现的错误

派生类在调用某些函数时，需要先调用基类的某些函数，下面是错误的写法：  
```C++
class Base {
public: 
    void onResize(){...}
}

class Derived: public Base {
public:
    void onResize(){
        static_cast<Base>(*this).onResize();
        ...
    }
}
```
错误的原因实际上 `*this` 被类型转换为Base，这个转换的Base是一个**临时变量**而不是*this本身的，所以做的修改都是在临时变量Base副本上，而不是在本身this上。
正确做法：  
```C++
class Base {
public: 
    void onResize(){...}
}

class Derived: public Base {
public:
    void onResize(){
        this->Base::onResize();
        ...
    }
}
```
**dynamic_cast 速度很慢**  
因为现版本的dynamic_cast需要使用字符串比较来比较类名。对于一个四深度的单继承体系，需要比较四次strcmp调用，用以比较class名称。深度继承和多继承的成本更高，某些实现版本这样做有其原因，因为要支持动态链接。

# 29. 为异常安全而努力
异常安全的两个基本保证：
1. 不泄漏任何资源
2. 不允许数据被破坏，保持对象内部的数据一致性

下面是一个可以并发访问的GUI更换背景的不好示例：
1. lock可能永远不会释放，因为前面可能new会出现异常。
2. new异常后，imageChanges已经改变，但是实际上却没有被改变。
```C++
class PrettyMenu {
private:
    Mutex mutex;
    Image * bgImage;
    int imageChanges;       // 背景图片更换次数
public:
    // ...
    void changeBackground(std::istream & imgSrc){
        lock(&mutex);                       // 加锁
        delete bgImage;                     // 移出
        ++imageChanges;
        bgImage = new Image(imgSrc);
        unlock(&mutex);
    }
    // ...
}
```

好代码一：利用条款13，让RAII自动管理资源，然后调整语句顺序保证状态一致性，要么更换成功，状态转移到新的状态，要么更换失败，相当于没有调用过这个函数。
```C++
class Lock{
private:
    shared_ptr<Mutex> mu_ptr;
public:
    Lock(Mutex * mu):mu_ptr(mu, unlock){
        lock(mu_ptr.get());
    }
};

class PrettyMenu {
private:
    Mutex mutex;
    shared_ptr<Image> bgImage;
    int imageChanges;       // 背景图片更换次数
public:
    // ...
    void changeBackground(std::istream & imgSrc){
        Lock(&mutex);                       // 加锁
        bgImage.reset(new Image(imgSrc));
        ++imageChanges;
    }
    // ...
}
```

另一种实现方法叫pimpl idiom，每次做改变时，先生成一个新对象，然后调用swap函数交换新对象。所以需要先抽象出来一个新对象的内容。
代码如下：  
```C++  
struct PMImpl {
    shared_ptr<Image> bgImage;
    ing imageChanges;
};

class PrettyMenu{
private:
    Mutex mutex;
    shared_ptr<PMImpl> pImpl;
public:
    void changeBackground(istream& imgSrc){
        using std::swap;
        Lock ml(&mutex);
        shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));
        pNew->bgImage.reset(new Image(imgSrc));
        ++pNew->imageChanges;
        seap(pImpl, pNew);
    }
};

```



