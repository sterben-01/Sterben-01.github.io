---
title: Effective Modern C++ 笔记
date: 2022-07-20 01:55:00 -0500
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

# Effective Modern C++ 笔记

# 条款18需要显式所有权的资源管理时，用`std::unique_ptr`

# 条款26 避免重载万能引用

**2023.2.21：直接看模板的6.2**

第一次看书给我看乐了。核心就一句话。根据C++的重载决议规则，万能引用版本总会被优先匹配。万能引用很jb贪。。它们会在具现过程中，和几乎任何实参型别都会产生精确匹配。而且在重载过程当中，万能引用模板还会和构造函数拷贝构造函数竞争。例子不写了，看书吧。反正别重载万能引用就行。

**要点:**

- 把万能引用作为重载候选型别，几乎总会让该重载版本在始料未及的情况下被调用到
- （完美转发）构造函数(使用了万能引用）的问题尤其严重，因为对于非常量的左值型别而言，它们一般都会形成相对于复制构造函数的更佳匹配，并且它们还会劫持子类中对父类的复制和移动构造函数的调用

# 条款27：26的解决方案

## 1. 使用常量左值引用做为形参（`const &`)

常量左值引用可以接受任意类型的参数。（常量左值，左值，常量右值，右值）。虽然效率低一些，但是可以正确使用。

## 2. 传值

有些人忽略一点。以值传递是可以接受右值的

```c++
void num(int a){
    cout << a << endl;
}
int main(){
    int digit = 5;
    num(digit);	//	OK 
    num(800); 	//	OK
    return 0;
}
```

所以：

```c++
class Person {
public:
    explicit Person(std::string n) //值传递
    : name(std::move(n)) {}

    explicit Person(int idx)
    : name(nameFromIdx(idx)) {}
    ...
private:
    std::string name;
};

```

没有效率损失的原因：如果实参是左值，那么实参到形参是一次复制，形参到`name`是一次移动，相比普适引用只多了一次移动；如果实参是右值，那么实参到形参是一次移动，形参到`name`还是一次移动，相比普适引用还是只多一次移动，可以认为没有效率损失。

## 3.使用标签。

**2023.2.21：直接看模板的标记派发。**

这里很有意思。侯捷老师讲STL的时候提到过，当时没有理解。现在有点理解了。

我们重新实现` logAndAdd`把它委托给另外两个函数，**一个接受整型值，另一个接受其他所有型别**。而` logAndAdd` 本身则接受所有型别的实参，无论整型和非整型都来者不拒。

改动前的原始版本：

```c++
std::multiset<std::string> names;
template <typename T>
void logAndAdd(T&& name) {
    names.emplace(std::forward<T>(name)); //具体代码细节无须在意
    //...
}
```
接近实现正确的版本：

```c++
template <typename T>
void logAndAdd(T&& name) {
    logAndAddImpl(std::forward<T>(name), 
                  std::is_integral<T>()//检查T的类型是否为整型
    );
}
```

上面的问题是：当实参是左值时，`T`会被推导为左值引用，即如果实参类型是`int`，那么`T`就是`int&`，（杂记的完美转发推导有写）`std::is_integral<T>()`就会返回false（此函数判断是否为整型。但是**所有的引用型别都不是整型**）。**这里我们需要把`T`可能的引用性去掉**：

```c++
template <typename T>
void logAndAdd(T&& name) {
    logAndAddImpl(
        std::forward<T>(name),
        std::is_integral<typename std::remove_reference<T>::type>() //检查T的类型是否为整型
    );
}
```
然后`logAndAddImpl`提供两个特化版本：
```c++
template <typename T>
void logAndAddImpl(T&& name, std::false_type) { //如果不是整型
    names.emplace(std::forward<T>(name)); //具体代码细节无须在意
}

template <typename T>
void logAndAddImpl(T&& name, std::true_type) {	//如果是整型
    logAndAdd(nameFromIdx(idx)); //具体代码细节无须在意
}
```

为什么用`std::true_type`/`std::false_type`而不用`true/false`？前者是编译期值，后者是运行时值。

注意这里我们都没有给`logAndAddImpl`的第二个参数起名字，说明它就是一个Tag。这种方法常用于模板元编程。它们在运行期不起任何作用。



# 条款35 优先选用基于任务（std::async）而非基于线程（std::thread）的程序设计。

关于这一点其实我也困惑了比较久，尤其是基于`std::thread`，`std::async`这样的高级函数，和`POSIX`系列的底层接口在多线程中的区别。

一般来说，在使用现代C++的高级库的时候，异步操作可以选用`std::thread`和`std::async`。

**基于任务的方法通常比基于线程实现的对应版本要好**。**主要原因是`async`可以让你获取到异步执行的返回值。**因为我们要有一个`future`对象做为句柄。而`thread`则不会给你机会直接返回一个返回值。

线程在C++软件的世界里有三种含义：

- 硬件线程是实际执行计算的线程。现代计算机体系结构会为每个CPU内核提供一个或多个硬件线程。
- 软件线程(又称操作系统线程或系统线程)是操作系统用以实施跨进程的管理，以及进行硬件线程调度的线程。通常，能够创建的软件线程会比硬件线程要多，因为当一个软件线程阻塞了（例如，阻塞在I/O操作上，或者需要等待互斥量或条件变量等)，运行另外的非阻塞线程能够提升吞吐率。
- **`std::thread`是C++进程里的对象，用作底层软件线程的句柄。**有些`std::thread`对象表示为“`null`”句柄，对应于“无软件线程”，可能的原因有: 
  - 它们处于默认构造状态（因此没有待执行的函数)
  - 被移动了（作为移动目的的`std::thread`对象成为了底层线程的句柄)
  - 被联结了(`join`。待运行的函数已运行结束)，
  - 被分离了(`detach` 。`std::thread`对象与其底层软件线程的连接被切断了)。

**那么什么时候使用`std::thread`更好呢？**

- 需要访问底层线程实现的API。
  - `std::thread`的 `native_handle`函数提供了一个返回底层线程句柄的方法。
- 需要且有能力优化线程用法
- 需要实现超越C++并发API的线程技术。比如细化线程池。

**注意到我们必须时刻关注线程运行的数量。所以`async`的默认启动方式是`std::launch::async | std::launch::deferred`。也就是到底是真正的异步执行还是推迟执行是交给系统去选择的。这是下一条的讨论重点**。



# 条款36，如果异步是必要的，则指定std::launch::async

我们在杂记3中已经详细讨论过了async的启动方式。我感觉大多数情况下，只要使用了`async`，应该都会使用异步方式启动。但是为了避免意外情况，在这里详细讨论一下，使用默认方式启动可能会遇到的问题。

比如我们有

```c++
auto fut1 = std::async(f);
```

- 无法预知`f`是否会和调用线程并发运行，因为`f`可能会被调度为推迟运行。
- 无法预知`f`是否运行在与调用`fut`的`get`或`wait`函数的线程不同的某线程之上。
- 连`f`是否会运行这件起码的事情都是无法预知的，这是因为无法保证在程序的每条路径上，`fut`的 `get`或`wait`都会得到调用。

**尤其是针对有`thread_local`变量的情况下，我们无法确认f到底是使用调用线程的`thread_local`变量还是新线程的。**

## 基于future 的 wait_untill /wait_for 的循环以超时为条件的情况下可能导致永远无法退出循环

假设我们有如下代码

```c++
void f(){
    //一些操作
}
auto fut = async(f); //默认启动条件。表面上的异步运行

while(fut.wait_for(100ms) != std::future_status::ready){ //注意这里
    //其他操作
}
```

- 假设如果`f`和调用线程是并发执行的，也就是`async`真的异步启动了，那么这个操作没什么问题。
- 但是假设`f`并没有被`async`异步启动，而是采用了推迟执行。由于我们从未对fut调用过`get`或`wait`，则`f`操作永远不会被执行。该`future`对象的状态将永远是`std::launch::deferred`。所以这个`while`永远都不会退出。

### 修正技巧：调用wait_untill 或 wait_for 并检查返回值

比如上面的代码可以修改为这样

```c++
void f(){
    //一些操作
}
auto fut = async(f); //默认启动条件。表面上的异步运行

if(fut.wait_for(0s) == std::future_status::deferred){ //实际上我们不需要等待任何事情，0s即可。然后查看是否被推迟
    fut.get();//如果被推迟，就显式的调用get或wait
}
else{
    while(fut.wait_for(100ms) != std::future_status::ready){ //如果未被推迟，确为异步执行。则正确执行。
    	//其他操作
	}
}

```

- 调用`wait_for`中，实际上我们不需要等待任何事情，所以参数为`0s`即可。然后查看是否被推迟。如果被推迟，就显式的调用`get`或`wait`



# 条款37 使thread对象在所有路径皆不可联结 -- 也就是让thread对象在离开其作用域范围之前被join或detach掉。

**换句话说就是：thread对象在离开其作用域的时候，必须确保它的状态不是joinable的。**

非joinable的thread对象包括：

- 默认构造的（本来就无效）
- 已被移动的（所有权被转移）
- 已被`join`的（废话）
- 已被`detach`的（thread对象已和底层执行线程分离）

我们在并发笔记中提到了，如果一个thread对象在离开作用域（调用析构函数）的时候，仍然是joinable的状态，则会直接调用`std::terminate`。

**解决这个问题的方式是使用RAII管理线程对象。我们在并发笔记中写到了。此处不赘述。**

