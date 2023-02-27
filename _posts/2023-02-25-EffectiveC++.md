---
title: Effective C++ 笔记
date: 2023-02-25 01:55:00 -0500
categories: [笔记]
tags: [C++]
pin: false
author: 01

toc: true
comments: true
typora-root-url: ../../Sterben-01.github.io
math: false
mermaid: true
---

# 条款25 swap函数的实现方式

具体原因没啥好说的，都能理解。概括一下使用方式。

- 当标准库的`std::swap`对某种自定义类型效率很低的时候，提供一个`swap`成员函数，并且确定该函数不抛出异常。
  - 具体自定义类应该使用pimpl手法，也就是保有资源指针。所以成员`swap`仅仅需要交换指针。
- 由于通常来说，我们的保有资源指针是`private`的，所以应该再提供一个非成员函数的`swap`，让这个`swap`调用那个非成员函数的`swap`。
- 对于非模板类类型，可以考虑特化`std::swap`
- 针对类模板的非成员`swap`函数，应该使用`using std::swap`引入标准库的`swap`函数。
  - 这样在`T`类有合适的swap的时候，自然会使用`T`类自己的`swap`。如果`T`类没有自己的`swap`函数，则会匹配至`std::swap`标准库的`swap`函数。
- 最后，调用`swap`的时候不要加任何命名空间修饰符。

# 第五章

26条：延后变量的定义，并且尽量直接构造对象而不是先默认构造再赋值。

27条：少进行类型转换。因为很多时候可以通过重新设计避免类型转换。

28条：注意避免返回可以指向某一个对象内部的指针，引用，或迭代器（统一称之为句柄或handler）。因为很多成员变量可能是private的，我们在一个成员函数直接给这个东西返回了个引用，就破坏了封装性。一是本来不应被更改的可能被更改，二来本来可见的变得可见了。同时可能发生悬空指针或悬空引用。

31条：降低编译依赖。这个就是到底是用`include`还是用前向声明。这个杂记3里有。

# 第六章

整个第六章讲的都是类的设计相关。这里不分别叙述，写在一起

- 32条：明确`public`继承是is-a关系。意味着适用于基类每一件事情都适用于派生类身上。因为每一个派生类对象也都是一个基类对象。
  - 问题在于比如正方形继承自长方形，就会有问题。比如长方形的长宽可以不相等。正方形必须相等。假如我们在变更长方形的长或宽后检查了其长款不等，则这件事情不适用于正方形，因为正方形的长和宽永远相等。所以在这件事情上，就会出现问题，需要细细考虑。
- 33条：主要是讲的继承中的隐藏（重定义）。这个在`vptr`有提到。主要是使用`using`引入父类的函数或变量名使其在子类作用域中可见。
- 34条：主要是区分继承接口（抽象基类）和继承实现。
  - 主要是研究非虚函数，虚函数和纯虚函数的意义。
  - **非虚函数的目的是为了让所有派生类继承这个函数的接口和一份强制（可能是缺省）的实现。**
    - 因为非虚函数的意义是无论派生类多么特殊，或有多么不同的行为，这个非虚函数所展现出的事情是不应该被改变的。
  - 在public继承之下，派生类总是继承基类的接口（抽象基类）
  - **一个纯虚函数的目的是为了让派生类只继承其接口而不继承其实现。因为子类必须提供纯虚函数的一份实现。**
  - **至于提供某一个函数的缺省实现，既可以使用一个额外的非虚函数，也可以直接给纯虚函数提供一份实现。**
    - 这样在需要使用缺省设置的时候，可以直接使用`对象.抽象基类::函数()`来使用缺省的函数实现。

- 35条：很多时候满足多态需求并不一定需要依靠虚函数，也可以依靠某些设计模式。**（这里所谓的“不依靠虚函数”，不代表不使用虚函数。一定要注意。）**
  - 模板方法模式。这个和模板没有任何关联。详细会在设计模式笔记介绍。
    - 更具体点，模板方法模式的一个特殊形式是NVI手法，也就是non-virtual interface。让外部调用一个公有非虚函数。但是在非虚函数内对虚函数（可能是私有的）进行调用。
    - 优点是可以在调用虚函数的之前和之后进行一些预处理或善后工作。比如加锁解锁，记录日志，验证条件等等。
  - 策略模式。这个在模板中的19.2提到过。详细会在设计模式继续补充。
    - 模板版本的策略模式和普通的策略模式（保有一个策略类的指针）的最大区别是能否被替换。模板版本相当于把类型信息加入了对象当中，自然不可替换。但是普通的策略模式可以随着使用的时候进行替换。


- **36条：绝对不要重新定义（隐藏）继承而来的非虚函数。**

- **37条：绝对不要重新定义一个继承而来的虚函数的默认参数。因为默认参数是静态绑定，虚函数本身则是动态绑定。杂记2中有详细说明**
- 38条：has-a关系不仅表示包含，也可以表示：根据某物实现出



# 第七章

- 41条：显式接口基于函数签名，动态多态基于虚函数。隐式接口基于模板，基于有效表达式，静态多态基于函数重载。

- 43条：就是继承类模板产生的名称依赖问题。深度探索对象模型的7.1
- 44条：**由于模板会导致代码膨胀，**也就是`T<int,3>`，`T<int, 4>`，和`T<long, 4>`是3种类型，则会生成三份类或函数的实例。尤其是在类模板中，**假设某个针对这些类型（或所有类型）的操作是通用的，则应该以适当的方式把这部分相同的操作抽离出来。**

```c++
template<typename T>
struct same{ //抽离公共部分
    protected:
    void same_way(){
        cout <<"process" << endl;
    }
};

template<typename T, unsigned int N>
struct A : same<T>{
    void diff_way(){
        std::puts(__PRETTY_FUNCTION__);
    }
    void same_way_wrapper(){
        this->same_way(); //必须使用this或::引入依赖名。条款43
    }
};

int main(){
    A<int, 2> obj;
    A<int, 3> obj2;

    obj.diff_way();
    obj.same_way_wrapper();
    obj2.diff_way();
    obj2.same_way_wrapper();
}

```

如果这样写的话，`A`会照常实例化两份`A<int,2>` 和` A<int,3>`，但是`same`只会实例化一份。也就是`same<int>`。

尽管我们可能会说，`same_way_wrapper`依旧实例化了两份。但是我们要知道，真实场景下，`same_way`可能是几十几百行的函数。而`same_way_wrapper`只有一行，几百行和一行相比，这个膨胀率我们是可以接受的。

**继承只是一种实现方式。我们可以换成包含。**



## 46条：函数模板的类型推导不支持隐式类型转换。所以如果我们在写一个类模板，但是其中有函数需要类型转换的时候，需要写成类内部的`friend`函数。

这一条非常值的拿出来单独一说。这个东西的例子最常见在运算符重载

- 普通类的运算符重载

```c++
struct myclass{
    int val = 0;
    myclass(int x):val(x){};
    myclass operator+(const myclass& rhs){
            return myclass(val + rhs.val);
    }
};
//---类外版本----
// myclass operator+(const myclass& lhs, const myclass& rhs){
//         return myclass(lhs.val + rhs.val);
// }
int main(){
    myclass obj(200);
    myclass another = obj+400;
    cout << another.val << endl;
}
```

- 类模板的运算符重载

```c++
template<typename T>
struct myTclass{
    int val = 0;
    myTclass(int x):val(x){};
    myTclass operator+(const myTclass& rhs){ //类内版本 OK
        return myTclass(val + rhs.val);
    }
};
//----类外版本，注意这里是错的-----
// template<typename T>
// myTclass<T> operator+(const myTclass<T>& lhs, const myTclass<T>& rhs){
//     return myTclass<T>(lhs.val + rhs.val);
// }
int main(){
    myTclass<int> obj(200);
    myTclass<int> another = obj + 400;
    cout << another.val << endl;
}
```

至于运算符重载，尤其是当前是加法的时候，为了满足加法交换律，我们普遍会需要在类外写一个。但是问题来了。我们类内定义的没问题，因为不涉及类型转换。**但是类外的就出问题了。因为如`obj+400`如果和类外的运算符重载进行匹配，`400`是不能被隐式转换成`myTclass<int>(400)`的。**

- `operator+(obj, 400)`这个函数调用从第一个参数可以推导出`T`的类型是`int`，因为第一个参数的类型是`myTclass<int>`。由于在函数签名中，两个参数类型一致，所以期望第二个参数依旧是这个类型。但是突然出现了`int`，这两个参数类型对不上了。又不支持隐式转换。（想不通的话想一下最基础的例子：`max<T>(1,2.2)`为什么也不行）

**所以解决方式是以一个友元函数的身份写在类内即可。**

```c++
template<typename T>
struct myTclass{
    int val = 0;
    myTclass(int x):val(x){};
    friend myTclass<T> operator+(const myTclass<T>& lhs, const myTclass<T>& rhs){ //这里的三个<T>都是可以省略的。但是为了清晰，还是加上吧。
        return myTclass(lhs.val + rhs.val);
    }
};
int main(){
    myTclass<int> obj(200);
    myTclass<int> another = obj + 400;
    cout << another.val << endl;
}
```

**这么做之所以可行的原因是这个操作符重载是普通函数，并非函数模板。**它依托于整个类模板。一旦类模板被实例化，则函数也会被合成。所以当`myTclass<int> obj`的执行让`myTclass<int>`类被合成出来的时候，这个函数就已经可见了。然后由于是函数而非函数模板自然支持隐式类型转换。**注意不要因为看到模板类就觉得不可能发生隐式类型转换。编译器通过模板合成出来的类或函数与普通类或函数有同样的行为。**
