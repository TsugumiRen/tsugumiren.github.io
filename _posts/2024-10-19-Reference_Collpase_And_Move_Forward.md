---
title: 利用 Reference Collapse 实现你自己的 std::move 和 std::forward
date: 2024-10-19 19:55:00 +0800
categories: [Programming Languages, C++]
tags: [C++, Reference Collapse, Universal Reference]
description: 神奇又好玩的 Reference Collapse
---

## Reference 的 Reference

在 C++ 里，我们可以通过指针的指针来指向一个链表头或是其他类似数据结构的操作：

```cpp
// insert a node in front of the list head
template<typename... Args>
void emplace_front(Node** head, Args&&... args){
    Node* node = new Node(std::forward<Args>(args)...);
    node->next = *head;
    head = &node;
}
```

既然有指针的指针，那么会不会有引用 (Reference) 的引用呢？像是这样：

```cpp
int&& r = 10;
int&& & rr = r; // error: 'rr' declared as a reference to a reference 
```

Oops，看来编译器不允许我们这样做。仔细想来这也没什么奇怪的，引用必须在声明时就与引用值进行绑定，而后也不能修改这份绑定，既然如此，那引用的引用也没有什么存在的必要。不过，在一定的场合也有例外。例如，在有类型推断 (Type Deduction) 的语境里，这种“引用的引用”就是允许的：

```cpp
auto&& & a = 10; // 就是(auto&&)&, 如果你觉得auto&& &看起来莫名其妙的话

templace<typename T>
void func(T&& t); // what if T is a int& or else
```

上面我们已经说了，引用的绑定关系是不允许改变的，那么这里的“引用的引用”到底是什么东西呢？

## Reference Collapse

揭晓答案：在上面语境中推断出来的类型仍然是一个引用，所以实际上我们还是没有“引用的引用”这种东西。引用的类型根据里面是否出现了左值引用来确定，如果在任何地方出现了一个左值引用，那么整一个都会被推导为左值引用；否则，则是一个右值引用。

```cpp
auto&& & a = 10; // lvalue reference: int&
auto& && b = 11; // the same
auto& & c = 12; // the same
auto&& && d = 13; // rvalue reference, congratulate!
```

这种引用的引用的推导被称为引用折叠 (Reference Collapse)，无论有多少级这样的"引用的引用“，最终都会被折叠成这个只有一级的引用。这不仅保证了在模板中有类似`T&&`这样参数时其能有正确的表现，我们还能利用这个特性实现更多的功能。

## 实现你自己的std::move 和 std::forward

C++ 11 可以算是 Modern C++ 的滥觞，与我们今天话题有关的最值得称道的或许就是引入了右值和移动语义 (Move Semantic) 的概念，其使得一些利用临时对象的构造和赋值操作变得显著地轻量化，避免了无意义的多次拷贝。

与这个特性相关的两个基础设施是`std::move`和`std::forward`，前者无条件地将一个左值转化为右值引用，后者则能在传递参数时保持参数的左右值特征。

### std::move

关于`std::move`的实现很简单，就是一个单纯的类型转换而已：

```cpp
template<typename T>
constexpr std::remove_refernce_t<T>&& std::move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>>&&(t);
}
```

关于这个实现有一点需要说明的是像`T&&`这样的类型参数的推导规则，它既可以接受左值作为参数，也可以接受右值，并且保留入参左右值的特征，因此被称作是 Universal Reference 。其推导规则是：

>1. 当`t`是左值且去掉引用部分的类型为`_T`时，`T`的类型是`_T&`，`T&&`在 Reference Collapse 后的类型是`_T&`
>2. 当`t`是右值且去掉引用部分的类型为`_T`时，`T`的类型是`_T`，`T&&`在 Reference Collapse 后的类型是`_T&&`

借助这个特性，当左值传入时，`T`的类型为`_T&`，去掉引用再加`&&`后为`_T&&`；右值传入时，`T`的类型为`_T`，去掉引用再加`&&`后也为`_T&&`；

### std::forward

进入正题之前有必要讲讲`std::forward`的功用，或者说相反，讲讲没有`std::forward`，我们参数的左右值特征是怎么在参数传递中消失的。其关键点很简单：左/右值引用是一个左值。只要我们经过 Universal Reference 一转手，无论我们传入的参数都会变成一个左值。

```cpp
void f(int& t){
    std::cout << "lvalue" << std::endl;
}

void f(int&& t){ // need a rvalue
    std::cout << "rvalue" << std::endl;
}

template<typename T>
void foo(T&& t){
    // suppose the type of t will be sometype& or sometype&&
    // however, cuz t is a reference, so it will always be a lvalue
    f(t) // will be dispatched to f(sometype& t), cause t is a lvalue
    f(std::forward<T>(t)) // will be dispatched according to t's type
}

int main(){
    int a = 10;
    foo(a) // lvalue, lvalue
    foo(10) // lvalue, rvalue
    return 0;
}
```

由于到`t`这里它已经变成一个引用了，所以我们`std::forward`的目标就明确了：

>1. 当`t`是左值引用时，`t`保持不变；
>2. 当`t`是右值引用时，把`t`转换为右值。

实现就是：

```cpp
template<typename T> 
T&& std::forward(std::remove_reference_t<T>& param) 
    return static_cast<T&&>(param);
}
```

放到我们上文提到的 Universal Reference 的语境中去看：

```cpp
template<typename T>
void foo(T&& t){
    f(std::forward<T>(t))
}
```

>1. 当`t`是左值且去掉引用部分的类型为`_T`时，`T`的类型是`_T&`，`T&&`的类型是`_T& &&` -> `_T&`
>2. 当`t`是右值且去掉引用部分的类型为`_T`时，`T`的类型是`_T`，`T&&`的类型是`_T&&`

完美！