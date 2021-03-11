---
title: 如何写好一个 C++ 类
date: 2021-03-11 16:02:16
tags: ['学习笔记']
---

什么三原则五原则六原则，统统不需要！

<!-- more -->

# 类的关注点与特殊成员函数

代码写出来除了运行，也是给人看的。拿到一个类的代码，我们阅读时关注的是（按优先级）：

* 包含哪些资源（看成员变量、析构函数）

* 是否可以无参构造（看默认构造函数）

* 是否可以拷贝或移动（看拷贝构造、拷贝赋值运算符、移动构造、移动赋值运算符）

* 除了拷贝和移动，还可以如何构造（看其它构造函数）

* 能够做什么其它事（看成员函数）

正好对应了类的特殊成员函数：

* 析构函数

* 默认构造函数

* 拷贝构造函数

* 拷贝赋值运算符重载

* 移动构造函数

* 移动赋值运算符重载

所以要按这样的顺序写：

```cpp
class MyClass {
private:
  // 成员变量，一眼看出这个类大概有什么东西
  A a_;
  B b_;
  C* c_;

public:
  // 析构函数里可以看出这个类有没有指针成员需要释放
  // 有 virtual 就可以继承
  virtual ~MyClass() noexcept;

  // 如果声明了其它构造函数，编译器不会提供默认构造函数
  // 所以类如果需要默认构造，要跟在析构函数后面出现
  // 这样可以一眼看出能不能默认构造、默认构造在干什么
  MyClass() noexcept;

  // 拷贝构造和拷贝赋值
  MyClass(const MyClass&);
  MyClass& operator=(const MyClass&);

  // 移动构造和移动赋值
  MyClass(MyClass&&) noexcept;
  MyClass& operator=(MyClass&&) noexcept;

  // 其它构造函数和成员函数
  // ...
};
```

# 编译器行为

一张表说明一切

| 用户声明 | 默认构造函数 | 析构函数 | 拷贝构造函数 | 拷贝赋值运算符重载 | 移动构造函数 | 移动赋值运算符重载 |
| :-----: | :----: | :----: | :----: | :----: | :----: | :----: |
| 不声明 | 默认 | 默认 | 默认 | 默认 | 默认 | 默认 |
| 任意构造函数 | 未声明 | 默认 | 默认 | 默认 | 默认 | 默认 |
| 默认构造函数 |  | 默认 | 默认 | 默认 | 默认 | 默认 |
| 析构函数 | 默认 |  | 默认（危） | 默认（危） | 未声明 | 未声明 |
| 拷贝构造函数 | 未声明 | 默认 |  | 默认（危） | 未声明 | 未声明 |
| 拷贝赋值运算符重载 | 默认 | 默认 | 默认（危） |  | 未声明 | 未声明 |
| 移动构造函数 | 未声明 | 默认 | 删除 | 删除 |  | 未声明 |
| 移动赋值运算符重载 | 默认 | 默认 | 删除 | 删除 | 未声明 |  |

# 套餐选择

1. 不写特殊成员（不存在指针释放的简单类）

    ``` cpp
    class Normal {
    public:
    };
    ```

2. 像上面那样按优先级顺序写全（容器）

3. 只可移动不可拷贝（系统资源封装，如文件描述符）

    ```cpp
    class ResourceHandle {
    public:
      ~ResourceHandle() noexcept;
      ResourceHandle() noexcept;

      // ResourceHandle(const ResourceHandle&) = delete;
      // ResourceHandle& operator=(const ResourceHandle&) = delete;

      ResourceHandle(ResourceHandle&& other) noexcept;
      ResourceHandle& operator=(ResourceHandle&& other) noexcept;
    };
    ```

4. 不可拷贝不可移动

    ```cpp
    class Immovable {
    public:
      Immovable(const Immovable&) = delete;
      Immovable& operator=(const Immovable&) = delete;

      // Immovable(Immovable&& other) noexcept;
      // Immovable& operator=(Immovable&& other) noexcept;
    };
    ```

特别注意：析构函数会使编译器提供的的拷贝变危险，使编译器不提供移动构造和移动赋值。所以如果写了析构函数最好把拷贝和移动都显式地写全。
