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
        swap(pImpl, pNew);
    }
};

```

# 30. inline 功能
1. 免除函数调用成本，直接进行代码替换，但是会造成代码膨胀。
2. 浓缩不含函数调用的代码，对其进行优化。

inline 只是对编译器的一个申请，不是强制命令。inline是在编译期被替换的。inline这项申请可以隐式提出也可以明确提出，**隐式提出**的方法是将函数定义在类内。**显示提出**是在定义式前增加inline声明。

隐式：  
```C++
class Person {
public:
    int age() const {return theAge;}    
}
```
显式：  
```C++
template<typename T>
inline const T & std::max(const T & a, const T & b){
    return a < b ? b : a;
}
```
函数带有virtual或者带有循环和递归的inline申请，都不会被满足。

inline缺点：
1. 会导致代码膨胀
2. 代码升级不方便，所有使用过inline的代码都需要重新编译。而不使用inline的代码只需要重新链接即可。
3. inline函数的debug不方便，难以找到break point。

所以inline一般只用于小型、被频繁调用的函数上。

# 32. 降低文件间的编译依存关系

能使用object reference或者object pointer，就不用object。
尽量以class声明式替换class定义式。

接口继承的实现方法：  
```C++
class Person {
public:
    static std::tr1::shared_ptr<Person> create(const std::string & name,
        const Date & birthday, const Address & addr);
}

class RealPerson: public Person {
public:
    RealPerson(const std::string & name,
        const Date & birthday, const Address & addr){
            // init RealPerson
        }
    virtual ~RealPerson(){}
}

// 为接口函数提供具体实现
std::tr1::shared_ptr<Person> Person::create(const std::string & name, 
    const Date & birthday, const Address & addr){
        return std::tr1::shared_ptr<Person>(new RealPerson(name, birthday, addr));
    }
```

# 32. Public继承是is-a关系
父类具有的功能，子类必须具有。  
例如鸟和企鹅，鸟能飞，企鹅不能飞，所以鸟和企鹅不是is-a的关系。

# 33. 避免遮掩继承而来的名称
下面代码没法使用基类的被覆盖的函数：  
```C++
class A {
public:
    void mf3(){
        cout<<"A::mf3()"<<endl;
    };

    void mf3(double x){
        cout<<"A::mf3("<<x<<")"<<endl;
    }
};

class B: public A {
public:
    void mf3(){
        cout<<"B::mf3()"<<endl;
    }
};

int main(){
    B b;
    b.mf3(1.0);         // error, 基类的mf3被覆盖
}
```
正确做法需要使用using重新将被覆盖的函数显示出来：
```C++

class A {
public:
    void mf3(){
        cout<<"A::mf3()"<<endl;
    };

    void mf3(double x){
        cout<<"A::mf3("<<x<<")"<<endl;
    }
};

class B: public A {
public:
    using A::mf3;
    void mf3(){
        cout<<"B::mf3()"<<endl;
    }
};

int main(){
    A a;
    B b;
    a.mf3();
    a.mf3(1.0);

    b.mf3();
    b.mf3(1.0);
}
```

当不想继承基类的一些函数或成员时，可以考虑使用**private继承**，然后要么使用using 重新声明作用于，要么使用隐式内联，如下：
```C++
class Derived: private Base {
public: 
    virtual void mf3(){
        Base::mf3();            // 暗自成为inline，条款30
    }
}
```

# 34. 接口继承和实现继承

## 34.1 成员函数的接口总是会被继承的。
public继承意味着 **is-a** 的关系，对base class 做的所有为真的事情一定对derived class也为真。

## 34.2 声明一个纯虚函数的目的是为了让派生类只继承函数接口

## 34.3 声明非纯虚函数的目的是为了让派生类继承函数的接口和缺省实现

## 34.4 使用普通成员函数的目的是为了使派生类继承函数的接口及一份强制性实现。（基类不变性和凌驾特异性）
非虚函数一般代表的意义是不变性和凌驾特异性，所以它**绝不该在派生类中被重新定义**。

非虚析构函数会带来非常大的问题，比如资源泄露。

当然，如果这个类不被设计为基类，那么不声明虚函数也是没问题的。

## 34.5 非虚函数实现Template Method 设计模式

虚函数继承为private，然后暴露给外部的是非虚函数，非虚函数调用继承来的虚函数。暴露给外部的是一个wrapper，该函数可以初始化、清理现场、参数检查等等。

Strategy 模式与Template Method模式不同，它有一个成员变量的指针函数指向特异化的计算方法。例如：计算一个游戏人物的hp值，不同人物有不同的计算方法，并且还有缺省方法：  
```C++
class GameCharacter;            // 前置声明
int defaultHealthCalc(const GameCharacter & gc);    // 缺省方法

class GameCharacter {
public:
    typedef int (*HealthCalc)(const GameCharacter & gc);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc):headlFunc(hcf){}

    int headlthValue() const {
        return healFunc(*this);
    }

private: 
    HealthCalcFunc healthFunc;      // 实际上是一个函数指针
}
```

另一个方法是使用std::function对象代替函数指针。
std::function 传入一个函数指针、可调用对象、lambda等等，生成一个被包装好的可调用对象。

# 36. 绝不重新定义继承来的non-virtual函数
也就是不覆盖任何基类的函数。

# 37. 绝不重新定义继承而来的缺省参数值
因为non-virtual函数不被覆盖，所以实际上本条款是针对virtual的。

函数缺省值是静态绑定的，但是virtual函数是动态绑定的，所以实际上的缺省值是基类的而非派生类的。

# 38. 通过复合实现 has-a 关系 和 “根据某物实现出”的关系

has-a关系：一个类拥有另一个类的xxx

is-implemented-in-terms-of: 通过某个类实现出另一个新类。
例如：通过list实现set，这不是一种继承 is-a的关系，因为基类list可以重复，派生类set不可以重复。所以list是set的一个成员。

# 39. 私有继承
1. 如果类间的继承关系是私有继承，那么编译器不会自动将一个派生类对象转换成基类对象。并且基类的所有public、protected成员全部变成私有的。

private继承实际上是一种 implemented-in-terms-of 的实现方法。

一般用复合关系实现 implemented-in-terms-of ，除非因为空间或者protected成员或/和 virtual 函数牵扯进来时，才用private继承。

# 40. 多重继承和虚继承
多重继承如果时菱形继承，则可能出现对同一对象的多重内存占用，所以在合适的时候需要使用虚继承。  

虚继承内存模型：[链接](https://yqlab.me/archives/51.html)


虚继承会增加内存的开销（虚类表指针），同时其初始化责任由继承体系中的最低层 class 负责，这意味着派生自 virtual 基类而需要初始化的类，必须首先判断它所有的virtual bases，先初始化它们。

所以对于virtual 基类，非必要不使用。如果使用，则尽量不要在其中放置数据（类比Java Interface不允许声明成员变量一样）。


# C++ 模版和泛型变成
C++ template机制自身是一部完整的图灵机：他可以被用来计算任何可计算的值，于是导出了模版元编程，创造出“在C++编译器内执行并于编译完成时停止执行”的程序。

# 41. 隐式接口和编译期多态

运行时多态：由virtual 函数实现。
编译期多态：由函数重载 或 模版template实现。

举个例子：
```C++
// 根据T的不同做不同的事儿
template<typename T>
void doProcess(T & w){
    // do ...
    return T.size();        // 只要T支持size成员函数，那么这样调用都没问题
}

template<>
void doProcess<Base>(Base & a){
    // do ...
    // 专门支持Base类的doPrzocess函数
}
```

# 42. 模板中 typename 与class 的区别
1. 模板中的嵌套类型需要增加typename 关键字，比如：  
```C++
// 错误，嵌套类型 const_iterator可能会被当作一个变量，而不是类型
template<typename C>
void print2nd(const C & container){
    if(container.size() >= 2){
        C::const_iterator iter(container.begin());
        iter++;
        ...
    }
}

// 正确，C::const_iterator被正确当成一个类型
template<typename C>
void print2nd(const C & container){
    if(container.size() >= 2){
        typename C::const_iterator iter(container.begin());
        iter++;
        ...
    }
}
```

另一个例子：  
只有嵌套类型需要增加 typename，container前面不需要增加。
```C++
template<typename C>
void func(const C & container, typename C::iterator iter);
```

typename声明类型时，不能在继承列表和初始化列表中增加。  
```C++
template<typename T>
class Derived: public Base<T>::Nested{      // Nested不加 typename
public:
    // 初始化列表不加typename
    explicit Derived(int x): Base<T>::Nested(x){
        typename Base<T>::Nested temp;      // 需要增加
    }

}
```

# 44. 将参数无关的代码抽离templates
将参数无关代码抽离成一个非模板的函数，然后才模板代码中调用它。因为模板会为每个类都生成一个代码版本。  

具有代码膨胀问题的版本：
```C++
// 大小信息作为参数传递
template<typename T, std::size_t n>
class SquareMatrix {
public:
    ...
    void invert();              // 求逆矩阵
};

// 下面的调用会生成两份参数不同的代码
SquareMatrix<double, 5>sm1;
sm1.invert();
SquareMatrix<double, 5>sm2;
sm2.invert();
```

可改进的方法，将参数无关的代码抽离，让模板参数n变为函数调用参数:  
```C++
template<typename T>
class SquareMatrixBase{
protected:
    SquareMatrixBase(std::size_t n, T * pMem):size(n), pData(pMem){}
    void setDataPtr(T * ptr){pData = ptr;}
    ...
private:
    std::size_t size;
    T * pData;
}

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
public:
    SquareMatrix():SquareMatrixBase<T>(n, data){}
private:
    T data[n*n];
}
```

另一方面，所有指针类型都有相同的二进制表述，因此凡是template持有指针者（例如list<int*>, list<SquareMatrix<long, 3> *），往往应该对每一个成员函数使用唯一一份底层实现，也就是如果你实现某些成员函数而它们操作强型指针(int *, long *)，你应该令它们调用另一个操作无类型指针(void *)的函数，由后者完成实际工作。

# 45. 泛化构造函数和泛化成员函数  
```C++
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U> & other);    // 生成泛化的拷贝构造函数
}
```

# 46. template 类型推导时不会考虑隐式类型转换
所以在需要类型转换的函数中最好将其声明为非成员函数或者友元函数。

# 47. traits classes 表现类型信息  
trait calss 的功能就是**允许你在编译期间取得某些类型信息**。它不是一个关键字，而是一种技术或者需要遵守的协议。

例如编译期间判断一个对象的类型是否是void：  
```C++
template<typename T>
struct is_void{
    static const bool value = false;
}

template<>          // 特例化模板
struct is_void<void> {
    static const bool value = true;
}

// 在使用时，用下面方法能够在编译期就获得类型：
is_void<false>::value;
is_void<int>::value;
```
**一个迭代器使用 traits 的例子：**
迭代器种类：
* input 迭代器, istream_iterator: 只能向前移动，一次一步，客户只能读取它所指的东西，只能读取一次。
* output 迭代器, ostream_iterator: 只能向前移动，一次一步，客户只能写它们所指的东西，只能写一次。
* forward 迭代器, 可以做多次读写，可以向前移动。
* bidirectional 迭代器, 可以向前向后移动(++运算符和--运算符),可以多次读写。
* random access 迭代器, 可以在常量时间内向前或向后移动任意距离

上述五种分类，C++标准程序库分别提供专属的卷标结构(tag struct) 加以识别和确认：
```C++
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag: public input_iterator_tag{};
struct bidirectional_iterator_tag: public forward_iterator_tag{};
struct random_access_iterator_tag: public bidirectional_iterator_tag{};
```

现在我们想实现一个 `advance` 函数，给定一个迭代器，使其移动一个固定的距离 d。那么如果是随机访问迭代器可以直接 iter + d，但是如果时其他迭代器则需要通过++和--方法来实现。这就出现了需要根据iter的类型来判断具体选择的方法。

运行时的解决方法: 
```C++
template<typename IterT, typename DistT>
void advance(IterT & iter, DistT & d){
    if(iter is a random access iterator){
        iter += d;
    }
    else {
        if(d >= 0){while(d--) ++iter;}
        else{while(d++) --iter;}
    }
}
```

但是实际上我们可以在编译期间就将其确定下来，因为编译时是知道iter的类型的。解决方法是额外声明一个模板 iterator_traits（标准库也有），针对每一个类型IterT，在`struct iterator_traits<IterT>` 内一定声明某个名为iterator_category。这个typedef 用来确认IterT的迭代器分类

所以对于任意一个容器的迭代器class内，必须首先声明一个 xxx_iterator_tag 的类为iterator_category，然后再将标准库的iterator_traits进行偏特化，然后内部包含相关信息。

**编译时做if判断的方法**：利用重载来实现编译时判断，因为重载会在编译期间自动选择调用哪个函数。

例如：
```C++
template<...>
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        ...
    };
};

// 偏特化iterator_traits;
template<typename IterT>
struct iterator_traits {
    typedef random_access_iterator_tag iterator_category;
    ...
};

// 现在有了iterator_traits 后，可以实现编译期间的advance了：
template<typename IterT, typename DistT>
void doAdvance(IterT & iter, DistT d, std::random_access_iterator_tag){
    iter += d;
}

template<typename IterT, typename DistT>
void doAdvance(IterT & iter, DistT d, std::bidirectional_iterator_tag){
    if(d >= 0){while(d--) ++iter;}
    else{while(d++) --iter;}
}

template<typename IterT, typename Dist>
void doAdvance(IterT & iter, DistT d, std::input_iterator_tag){
    if(d < 0){
        throw std::out_of_range("Native distance");
    }
    while(d--) ++iter;
}

// 总的函数
template<typename IterT, typename DistT>
void advance(IterT & iter, DistT d){
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}
```

Traits 广泛用于标准程序库，除了iterator_traits 提供的iterator_category外，还提供了其他信息如value_type。此外还有char_traits用来保存字符类型的相关信息、numeric_limits用来保存数值类型的相关信息，比如某数值类型的可表达的最大值和最小值。

TR1还刀入了许多新的traits classes用以提供类型信息，包括`is_fundamental<T>`判断是否是内置类型，`is_array<T>`（判断T是否为数组类型), 以及`is_base_of<T1, T2>`判断T1和T2是否相同或者T1是T2的基类。

# 48 模板元编程 (Template MetaProgramming)

1.模板元编程是编写template-based C++程序并执行于编译期间的过程。
2.模板元编程是于1990年初期被发现的，而不是被发明出来的，它是一个图灵完备的计算，能够解决任何可计算问题。
3.它能够让计算工作从运行期转移到编译期，这样某些错误可以在编译期被找出来。
4.部分情况下，它能有较小的可执行文件、较短的运行期、较少的内存需求。但是它却增加了编译时间。
5.模板元编程实际上是一种函数式编程。

前面提到，重载能够实现 if-else 逻辑，而循环则依靠递归来实现，因为其是函数式编程。

下面是一个斐波那契数列的例子：
```C++
template<unsigned n>
struct Factorial {
    // 用 enum hack方法声明静态常量，见条款2
    // enum 实际上是一个类，所以这其实是静态常量
    enum{value = n * Factorial<n-1>::value};        
};

template<>
struct Factorial<0> {
    enum {value = 1};
};

int main(){
    std::cout<<Factorial<5>::value;
    std::cout<<Factorial<10>::value;
}
```

另外，还有表达式template，它能够实现typename是表达式的模板类。

# 49 了解new-handler行为
注意：STL管理的对象是由allocator 管理，而不是new/delete管理。

new_handler是在 `operator new`无法继续分配内存抛出异常时调用的一个函数，它的函数签名是：
```C++
namespace std{
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw(); // C98式的不抛出异常
}

// 示例
int main(){
    std::set_new_handler(func);
    int * big_data = new int[100000000000LL];
}
```
当new引发异常时，如果new_handler不为none时，bad_alloc不会被抛出，只有new_handler为null时才会抛出，new 引发异常时无限循环分配-调用new_handler-分配-调用new_handler……，直到在new_handler里通过某种方法退出（安装另一个new_handler，卸载nwe_handler，清理内存使其足够可被使用，抛出bad_alloc）。

一个设计良好的new_handler必须完成以下事情：
1. 让更多内存可以被使用，也就是在这里释放一些内存。
2. 安装另一个new_handler，当目前的new_handler不能处理时，调用set_new_handler安装另一个，那么下一次再调用new时，将会调用其他的new_handler。
3. 卸载new_handler，设置为null指针即可
4. 抛出bad_alloc的异常，这样的异常不会被operator new捕获，会传导到代码调用处。
5. 不返回，调用abort或者exit。

# 50. 替换 new 和 delete  
重载的new和delete的原因如下：
1.**用来检测运用上的错误**，例如多次delete、new完delete失败内存泄漏、定制化内存比如在分配对象内存的前几个字节放上签名。
2.**定制自己的内存管理**，因为有些小内存频繁被申请和释放，可以自己用桶或者内存池来管理，用来增加分配内存和归还内存的速度，例如Boost的Pool库。也就是说operator new / operator delete 如果自己管理好的话，性能会提升很多。
3.**收集使用上的统计数据**，分配区块大小分布如何，寿命分布如何，运用形态是否随时间变化，最大动态分配量最高点是多少等等这些都需要定制化new收集。
4.**能够将相关对象集中**，分配一个大的heap，这样就能让对象尽可能在一页上，减少内存缺页错误，提升效率。

写自己的内存池需要注意字节对齐，比如double是8字节，那么double的地址必须是8的倍数，同理int是4字节，那么int的地址必须是4的倍数。这是因为逻辑上内存存取是按照字节来存的，但是底层可能是一次取四个字节，也就是说存储单元最小是四字节。这样假设double不是8的倍数，很可能会出现跨字节存取，前4字节取一半，后4字节取一半，极大地降低了存储效率。甚至有些机器上还会引发异常。


## operator new 的成员函数可能会被派生类继承
这样会导致Base class 的operator new 会被调用来分配Derived 对象。
解决方法：给出申请量错误：
```C++
void * Base::operator new(std::size_t size) throw(std::bad_alloc) {
    if(size != sizeof(Base))
        return ::operator new(size);    // 交给标准operator new处理
    ...
}
```
基类的delete同理也会被派生类调用，用同样的方法判断size是否一样，不一样就交给标准operator delete处理。

下列输出结果是:
hello A  
A constructor  
```C++
class A {
public:
    A(){
        cout<<"A constructor"<<endl;
    }

    void * operator new(size_t size) {
        cout<<"hello A"<<endl;
        return malloc(sizeof(size));
    }
};

int main(){
    A * a = new A();
}
```

# 52. 写了placement new 也要写 placement delete
如果operator new接受的参数除了一定会有的size_t之外，还有其他的参数，那么就是placement new，一般而言，其他的参数就是一分配好空间的地址，在这个地址上进行初始化。例如：

```C++
void operator new(size_t size, ostream & logStream);
void operator new(size_t size, Object * obj);

// 调用时：
Widget * pw = new (std::cerr)Widget;
Widget * pw = new (&addr)Widget;
```

如果内存分配成功，但是调用构造函数出现异常，则运行期系统有责任取消operator new的分配并恢复原样。然而运行期调用的delete必须寻找**参数个数和类型**都与operator new 相同的那个delete。所以当提供了placement new时必须也要提供对应的placement delete。

注意，运行期的delete是自动调用的，当分配异常后会自动调用。所以一半有placement new时，要提供placement delete和正常版本的delete。

另外，成员函数会覆盖其外围作用于中相同的名称，也就是说会覆盖正常函数，也就是说当只有唯一的一个placement new时，正常的new函数就被覆盖了，所以要么将placement 声明为global版本而不是成员函数版本，要么使用using 防止其被屏蔽。

# 53. 请不要忽视编译期警告
```C++
class B {
public:
    virtual void f() const;
};
class D: public B{
public:
    virtual void f();
}
```
上面会给出D覆盖了B的f函数的警告，而没有达到运行时多态的效果。
