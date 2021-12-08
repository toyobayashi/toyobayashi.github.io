---
title: 面向前端同学的 Emscripten WebAssembly 介绍（二）
date: 2021-12-08 17:39:34
tags: [技术分享]
---

面向前端或不熟悉 C/C++ 开发人员的 Emscripten WebAssembly 入门介绍。补一下 C/C++ 基础。

<!-- more -->

# 数据类型与内存

C/C++ 不像 JS 有垃圾回收，内存需要开发者自己管理，所以要知道数据是怎么放在内存中的。

## 内存

Emscripten wasm 默认的初始内存是 16M，在 JavaScript 中可以通过以下方式访问：

- `Module.HEAPU8` (Uint8Array)
- `Module.HEAP8` (Int8Array)
- `Module.HEAPU16` (Uint16Array)
- `Module.HEAP16` (Int16Array)
- `Module.HEAPU32` (Uint32Array)
- `Module.HEAP32` (Int32Array)
- `Module.HEAPF32` (Float32Array)
- `Module.HEAPF64` (Float64Array)

这些 `TypedArray` 共享同一个 `ArrayBuffer`。

## 数字

```c
#include <stddef.h>  // size_t
#include <stdint.h>  // *int*_t

// 字符，有符号 8 位整数，占用 1 字节内存，范围 [-128, 127]
char c = 'a';
int8_t i8 = 97;

// 无符号 8 位整数，占用 1 字节内存，范围 [0, 255]
unsigned char uc = 255u;
uint8_t u8 = 255u;

// 有符号 16 位整数，占用 2 字节内存，范围 [-32768, 32767]
short s = 256;
int16_t i16 = 256;

// 无符号 16 位整数，占用 2 字节内存，范围 [0, 65535]
unsigned short us = 65535u;
uint16_t u16 = 65535u;

// 有符号 32 位整数，占用 4 字节内存，范围 [-2147483648, 2147483647]
int i = -1;
int32_t i32 = -1;

// 无符号 32 位整数，占用 4 字节内存，范围 [0, 4294967295]
unsigned int ui = 4294967295u;
uint32_t u32 = 4294967295u;

// 有符号 64 位整数，占用 8 字节内存
// 范围 [-9223372036854775808, 9223372036854775807]
long long ll = -1ll;
int64_t i64 = -1ll;

// 无符号 64 位整数，占用 8 字节内存
// 范围 [0, 18446744073709551615]
unsigned long long ull = 18446744073709550592ull;
uint64_t u64 = 18446744073709550592ull;

// 单精度浮点数，占用 4 字节内存
float f = 3.14f;

// 双精度浮点数，占用 8 字节内存
double d = 3.14;

// 四精度浮点数，占用 16 字节内存
long double ld = 3.14l;
```

数字类型在内存中以小端序存储，比如 int 类型的 61183 的 16 进制是 `0x0000EEFF`，在内存中存的顺序就是

```txt
FF EE 00 00
```

例子：

```c
#include <emscripten.h>

EMSCRIPTEN_KEEPALIVE int* get_int32_value() {
  static int n = 61183;
  return &n;
}
```

```js
Module.onRuntimeInitialized = function () {
  var intPointer = Module._get_int32_value();
  var intMemory = Module.HEAPU8.subarray(intPointer, intPointer + 4);
  console.log(intMemory); // [0xFF, 0xEE, 0x00, 0x00]
  var intValue = Module.HEAP32[intPointer >> 2];
  console.log(intValue); // 61183
}
```

## 布尔值

在 C 中以 0 表示 false，非 0 表示 true。函数返回值通常返回错误码，返回 0 表示成功。

在 C++ 中有 `bool` 类型和字面量 `true` 和 `false`。

## 数组

连续的一段内存。

```c
// 创建 5 个 int 元素的数组，在内存中占用 20 字节
int arr1[5];

// 创建 5 个 int 元素的数组，并把每个元素初始化为 0
int arr2[5] = { 0 };

// 创建数组并初始化
int arr3[5] = { 0, 1, 2, 3, 4 };

// 省略数组大小
int arr4[] = { 0, 1, 2, 3, 4 };

// 访问元素
arr4[1];

// 修改元素
arr4[1] = 5;

// 不可越界
arr[5];
```

## 指针

内存地址，JS 中表现为 `Module.HEAPU8` 的下标。数组也可以当指针使，指向数组第一个元素的地址。

```c
int a = 1;
int* a_ptr = &a; // 取地址
int b = *a_ptr; // 访问地址

int arr[3] = { 1 };
int* arr_ptr = arr;
int a1 = *arr; // arr[0]
int a2 = *(arr + 1); // arr[1]
*(arr + 1) = 2; // arr[1] = 2;

void alert(const char* str) {
  // ...
}

void (*alert_ptr)(const char*) = alert; // 函数也是指针
alert_ptr("abc");
```

C 的空指针是 `<stddef.h>` 里的 `NULL`，一般展开为 `((void*)0)`。

```c
#include <stddef.h>

int* a = NULL;

typedef struct my_struct my_struct;

my_struct* create_my_struct() {
  // ...
  return NULL;
}
```

C++ 中不要用 `NULL`，直接使用 `nullptr`。

```cpp
int* a = nullptr;

class MyClass;

MyClass* CreateMyClass() {
  // ...
  return nullptr;
}
```

## 字符串

C 风格的字符串是以 0 结尾的字符数组，一般指这个字符数组的首地址指针。

```c
// 创建字符数组，内容可以修改
char str1[4] = "abc";
char str2[4] = { 'a', 'b', 'c', '\0' };
char str3[] = "abc";

// 指向字符串字面量的指针，内容不可修改，必须要 const
const char* str4 = "abc";
```

C++ 字符串使用标准库 `std::string`

```cpp
#include <string>

std::string str1;
std::string str2 = "abc";
std::string str3 = str2;
```

## 栈内存与堆内存

变量在栈上分配内存，离开作用域后自动释放内存

```c
{
  int a = 1;
  // 使用 a
}
// a 不可用
```

堆内存由开发者负责分配和释放，运行时不会自动释放内存，C 语言使用 `<stdlib.h>` 里的 `malloc` 和 `free` 库函数，C++ 语言使用 `new` 和 `delete` 关键字。

```c
#include <stdlib.h>  // malloc free
#include <string.h>  // memset

{
  int* a = (int*)malloc(sizeof(int));
  *a = 1;
  // 使用 a

  // 释放堆内存
  free(a);

  // 堆分配数组
  int* heap_arr = (int*)malloc(3 * sizeof(int));
  // 填内存
  memset(heap_arr, 0, 3 * sizeof(int));
  free(heap_arr);
}
// 如果忘记 free 将导致内存泄漏
```

```cpp
{
  int* a = new int(1);
  delete a;

  int* heap_arr = new int[3];
  delete[] heap_arr;
}
```

## 结构体与类

C 没有类，只有结构体

```c
struct s {
  int a;
  float b;
};
struct s s1 = { 1, 3.14f };

// 声明类型别名
typedef struct s s;
s s2 = { 1, 3.14f };

typedef struct s {
  int a;
  float b;
} s;
s s3 = { 1, 3.14f };
```

C++ 的 struct 是默认 public 成员的 class。

```cpp
struct S {
  int a_;
  float b_;

  // 可以写构造函数
  S() noexcept: a_(0), b_(0.0f) {}
  S(int a, float b) noexcept: a_(a), b_(b) {}
  // 默认拷贝构造
  S(const S& other) = default;
  // 移动构造
  S(S&& other) noexcept: a_(other.a_), b_(other.b_) {
    other.a_ = 0;
    other.b_ = 0.0f;
  }

  // 可以写成员函数
  S& operator=(const S& other) = default;
  S& operator=(S&& other) noexcept {
    a_ = other.a_;
    other.a_ = 0;
    b_ = other.b_;
    other.b_ = 0.0f;
    return *this;
  }

  int GetA() const noexcept {
    return a_;
  }

  S& SetA(int a) noexcept {
    a_ = a;
    return *this;
  }
};

S s1;
S s2(1, 3.14f);
S s3{1, 3.14f};
```

C 中两个结构体不推荐直接赋值，结构体成员指针指向的内存不会复制，C++ 通过 `operator=` 赋值运算符重载来控制具体行为。

# 声明与定义分离

调用函数必须在前面能找到函数声明（类似 JS 的变量提升），声明处可以没有函数定义（函数体）。

```c
void a(int x) {
  // ...
}

void b() {
  a(1); // ok
}

int main() {
  b();
  return 0;
}
```

```c
void b() {
  a(1);  // 编译不通过
}

void a(int x) {
  // ...
}

int main() {
  b();
  return 0;
}
```

```c
// 先声明
void a(int x);

void b() {
  a(1);  // ok
}

// 后定义
void a(int x) {
  // ...
}

int main() {
  b();
  return 0;
}
```

# 多个编译单元

代码写多了不可能全塞在一个源文件中。可以把函数定义放在不同的源文件中，每个源文件当作一个编译单元编译成目标文件，链接时只要能在这些目标文件中找到函数定义就没有问题。

```c
// api.c

int add(int a, int b) {
  return a + b;
}
```

```c
// main.c

#include <stdio.h>

int add(int a, int b);

int main() {
  printf("%d\n", add(3, 4));
  return 0;
}
```

```sh
emcc -o main.js api.c main.c

# 或
emcc -c -o api.o api.c
emcc -c -o main.o main.c
emcc -o main.js api.o main.o
```

如果有很多源文件都要用到同一个函数，每个源文件都要写一次函数声明，就比较麻烦，可以把函数声明、类型别名、宏定义等等东西放在头文件中，然后在源文件 `#include` 头文件。

```c
// api.h

// 头文件保护，确保头文件内容只编译一次
#ifndef SRC_API_H_
#define SRC_API_H_

typedef int i32;
i32 add(i32 a, i32 b);

#endif
```

```c
// api.c

#include "api.h"

i32 add(i32 a, i32 b) {
  return a + b;
}
```

```c
// main.c

#include <stdio.h>
#include "api.h"

int main() {
  printf("%d\n", add(3, 4));
  return 0;
}
```

头文件不需要传给 `emcc`，因为头文件的内容会被 `#include` 到源文件中。

# 生成与链接静态库

上面的例子，加一个乘法，把加法和乘法函数编译成一个静态库，链接时链接静态库文件。

```c
// api.h

#ifndef SRC_API_H_
#define SRC_API_H_

typedef int i32;

i32 add(i32 a, i32 b);
i32 multiply(i32 a, i32 b);

#endif
```

```c
// api2.c

#include "api.h"

i32 multiply(i32 a, i32 b) {
  return a * b;
}
```

```c
// main.c

#include <stdio.h>
#include "api.h"

int main() {
  printf("%d\n", add(3, 4));
  printf("%d\n", multiply(3, 4));
  return 0;
}
```

```sh
emcc -c -o api.o api.c
emcc -c -o api2.o api2.c
emar rcs libapi.a api.o api2.o

emcc -c -o main.o main.c
emcc -o main.js libapi.a main.o
```
