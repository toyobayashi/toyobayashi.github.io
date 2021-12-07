---
title: 面向前端同学的 Emscripten WebAssembly 介绍（一）
date: 2021-12-07 17:31:33
tags: [技术分享]
---

面向前端或不熟悉 C/C++ 开发人员的 Emscripten WebAssembly 入门介绍。

<!-- more -->

# WebAssembly

MDN 传送门：[https://developer.mozilla.org/zh-CN/docs/WebAssembly](https://developer.mozilla.org/zh-CN/docs/WebAssembly)

WebAssembly 把服务端语言带进了浏览器，在需要大量 CPU 密集运算的场景（如前端加密算法，3D 图形渲染等等）可以弥补 JavaScript 的性能不足，也为 Web 开发生态打开了新的一扇窗。

支持编译到 WebAssembly 的主流编程语言包括但不限于：

- C/C++
- Rust
- Go
- AssemblyScript (阉割版的 TypeScript)

目前主流浏览器已支持 WebAssembly，浏览器外也有独立的 WebAssembly 运行时如 wasmtime / wasmer 等等。

# Emscripten

## 编译工具链

C/C++ 在不同平台有不同的编译工具链：

- Windows: Microsoft Visual C++ (`cl.exe`, `link.exe`, ...)
- Linux: GNU GCC (`gcc`, `g++`, ...)
- macOS: Clang (`clang`, `clang++`, ...)
- Android: NDK (`aarch64-linux-android23-clang`, `aarch64-linux-android23-clang++`, ...)

这些工具链所做的事情类比前端就相当于是 Webpack / Rollup / Vite + Babel + Terser，把 JS / JSX / TS / TSX / Vue 的模板 / Svelte 的模板转译到 ES5 再打包摇树压缩。

它们可以将 C/C++ 源码编译到不同 CPU 架构操作系统所支持的机器码，链接出来的可执行文件可以被操作系统直接运行，不像 Java / JavaScript 需要有 JVM / V8 那样的虚拟机来解释执行。

但是 WebAssembly 有自己的一套二进制标准，它的可执行文件（.wasm）并不直接由操作系统运行，而是由 WebAssembly 虚拟机来运行，WebAssembly 平台的 C/C++ 编译工具链是 [Emscripten](https://emscripten.org/)。

- WebAssembly: Emscripten (`emcc`, `em++`, `emar`, ...)

## 安装 Emscripten

官方提供了专门的安装工具 [emscripten-core/emsdk](https://github.com/emscripten-core/emsdk)，使用 emsdk 需要 Git 和 [Python 3](https://www.python.org/downloads/) 环境，安装过程中请保持**良好的网络环境**。

除此之外，使用 Emscripten 时也需要 Node.js 环境。

检查环境：

```sh
git --version
python -V
node -v
```

### 非 Windows

```txt
~/Projects$ git clone https://github.com/emscripten-core/emsdk.git
~/Projects$ cd emsdk
~/Projects/emsdk$ ./emsdk install latest
~/Projects/emsdk$ ./emsdk activate latest
~/Projects/emsdk$ source ./emsdk_env.sh
~/Projects/emsdk$ emcc -v
```

加环境变量

```sh
export EMSDK=~/Projects/emsdk
export PATH=~/Projects/emsdk:~/Projects/emsdk/upstream/emscripten:$PATH
```

### Windows

```txt
C:\Projects> git clone https://github.com/emscripten-core/emsdk.git
C:\Projects> cd emsdk
C:\Projects\emsdk> emsdk install latest
C:\Projects\emsdk> emsdk activate latest
C:\Projects\emsdk> emcc -v
```

编辑系统环境变量

```bat
set EMSDK=C:\Projects\emsdk
set PATH=%Path%;C:\Projects\emsdk;C:\Projects\emsdk\upstream\emscripten
```

# 编译 Hello World

以下是 C 语言的 Hello World 代码，命名为 `main.c`

```c
#include <stdio.h>

int main() {
  printf("Hello World\n");
  return 0;
}
```

编译命令：

```bash
emcc -o main.js main.c
```

生成了 `main.js` 和 `main.wasm`。

启动本地服务在浏览器中运行：

```html
<script src="main.js"></script>
```

可以看到 Console 输出了 `Hello World`。

## 编译过程

C/C++ 源码的编译过程分以下几个步骤：

1. 预处理（Preprocessing）
2. 编译（Compilation）
3. 汇编（Assemble）
4. 链接（Linking）

预处理阶段会处理源码中的 `#include`、`#ifdef`、`#define` 等等编译器指令，可以近似理解为是代码复制粘贴加替换。

```bash
emcc -E -o main.i main.c
```

编译阶段会把预处理得到的代码转换成文本格式的汇编代码（.s）。

```bash
emcc -S -o main.s main.i
```

汇编阶段会由上一步得到的汇编代码生成二进制格式的目标文件（.o）

```bash
emcc -c -o main.o main.s
```

每个编译单元都会经过这三个步骤生成一个目标文件，如果源文件中不 include 其他源文件，那么一个源文件就是一个编译单元。等价于 `emcc -c -o main.o main.c`。这三步类比前端就像是 Webpack 的 `DefinePlugin` 在编译时就处理了代码中的 `process.env.NODE_ENV`，根据配置的值自动删掉了 if else 走不到的分支，然后 Babel 转译 ESNext JS / JSX 源码到 Pure ES5。

链接阶段会把多个目标文件和需要用到的库文件（静态链接库或动态链接库）链接输出最终的可执行文件或动态链接库。不过 Emscripten 没有动态链接库。

```bash
emcc -o main.js main.o [xxx.o xxx.o xxx.a]
```

类比前端就像是 Webpack 把全部 JS 模块打包成一个 JS 文件，并且经过了 tree shaking 和压缩，把没有用到的代码都去掉了。

`emcc -o main.js main.c` 这个命令是一次性做完了所有步骤，一步到位生成了可执行文件。

如果要编译多个源文件，一般会用到 Makefile 或 CMake 来配置构建，本质上还是把所有编译单元编译成目标文件，最后再进行链接。

```bash
emcc -c -o file1.o file1.c
emcc -c -o file2.o file2.c
emcc -c -o file3.o file3.c
emcc -o out.js file1.o file2.o file3.o library1.a library2.a
```

更多 emcc 的参数用法在[这里](https://emscripten.org/docs/tools_reference/emcc.html)查看。

## 编译目标类型

C/C++ 最终的编译目标可以是可执行文件、静态链接库或动态链接库。

### 可执行文件

Windows 的可执行文件后缀是 `.exe`，Linux / macOS / Android 的可执行文件没有后缀，Emscripten 的可执行文件后缀是 `.wasm`。

可执行文件是链接时生成的包含机器码的二进制文件，操作系统的可执行文件可以由操作系统直接运行。

类比前端就像是 `<script>` 引入的 JS 文件。

### 静态链接库

Windows 的静态链接库后缀是 `.lib`，Linux / macOS / Android / Emscripten 的静态链接库后缀都是 `.a`。

静态链接库是由编译后的目标文件打包生成的结果，在链接时传给链接器，链接器会去目标文件里寻找最终可执行文件或动态链接库要用的函数符号。

类比前端就像是 node_modules 里的 package，Webpack 打包时静态解析 `import` 和 `require`，把 node_modules 里包打进了最终的 JS 里。

### 动态链接库

Windows 的动态链接库后缀是 `.dll`，Linux / Android 的动态链接库后缀是 `.so`，macOS 的动态链接库后缀是 `.dylib`，Emscripten 不存在真正意义上的动态链接库，有也只是把 `.wasm` 改个后缀名改成了 `.so`。

动态链接库和可执行文件类似，都经过链接生成，包含机器码，但是动态链接库不能直接运行，必须在可执行文件运行时动态装入内存再运行。

类比前端就像是 Webpack 动态 `import()` 分出来的包，可以按需加载。

# 从 C/C++ 导出函数给 JavaScript

一般来说大部分情况使用 WebAssembly 不会直接写 main 函数，而是暴露原生函数给 JS，在 JS 要调用的时候再去调用原生函数。这里我推荐几种做法。

## EMSCRIPTEN_KEEPALIVE

第一种最原始也是效率最好的办法，就是在函数签名上加上 `EMSCRIPTEN_KEEPALIVE`，它会告诉编译器这个函数会被用到，不要在“tree shaking”的时候删掉，并且会将函数名加上前缀 `_` 导出给 JS，就和编译器参数 `-sEXPORTED_FUNCTIONS` 一样。

```cpp
// lib.c

#include <emscripten.h>  // EMSCRIPTEN_KEEPALIVE

#ifdef __cplusplus
#define EXTERN_C extern "C"
#else
#define EXTERN_C
#endif

// 如果这个源码被 C++ 的编译器编译
// extern "C" 告诉 C++ 编译器不要修饰函数名称
// 按 C 的方式保留原函数名

// EMSCRIPTEN_KEEPALIVE 展开为 __attribute__((used))
// 告诉编译器不要删掉这个函数，并导出给 JS

EXTERN_C EMSCRIPTEN_KEEPALIVE
int add(int a, int b) {
  return a + b;
}
```

```bash
emcc -o lib.js lib.c
```

```html
<script src="lib.js"></script>
<script>
  Module.onRuntimeInitialized = function () {
    console.log(Module._add(3, 4)); // 7
  };
</script>
```

用这种方法导出的函数参数只能是数字类型或裸指针（指针也是数字），返回值也只能是数字类型或 `void`，在 JS 传的参数只能是 `number` 类型，返回值也只能是 `number` 或 `undefined`。

### JS 传字符串给 C

字符串在 C 中实际上是一个以 0 结尾的字符数组，内存是连续的。如果要把 JS 的 `string` 传到 C，首先要开辟一块 C 的内存，在把 JS 的字符串放进这段内存里，C 中就可以访问到了。

```c
#include <stdio.h>  // printf
#include <emscripten.h>  // EMSCRIPTEN_KEEPALIVE

EXTERN_C EMSCRIPTEN_KEEPALIVE
void log_js_string(const char* str) {
  printf("%s\n", str);
}
```

```bash
# 导出 C 的 malloc 和 free 在 JS 中分配和释放 C 的内存
emcc -sEXPORTED_FUNCTIONS=["_malloc","_free"] -o lib.js lib.c
```

```html
<script src="lib.js"></script>
<script>
  Module.onRuntimeInitialized = function () {
    var str = 'Hello World';
    // utf8 字符串 buffer
    var strBuffer = new TextEncoder().encode(str);
    /**
     * 分配字符串所需的内存空间
     * @type {number}
     */
    var strPointer = Module._malloc(strBuffer.length + 1);
    // 把字符串内容复制到 C 的内存中
    Module.HEAPU8.set(strBuffer, strPointer);
    Module.HEAPU8[strPointer + strBuffer.length] = 0; // 以 0 结尾
    Module._log_js_string(strPointer); // Hello World
    Module._free(strPointer); // 用完以后释放内存
  };
</script>
```

### C 传字符串给 JS

同理，传字符串指针，往 JS 分配的内存中写入字符串内容。

```c
#include <stdio.h>  // snprintf
#include <stddef.h>  // size_t
#include <emscripten.h>  // EMSCRIPTEN_KEEPALIVE

EXTERN_C EMSCRIPTEN_KEEPALIVE
int get_c_string(char* out_str, size_t size) {
  if (out_str == NULL) {
    return 12;
  }
  return snprintf(out_str, size, "Hello World");
}
```

```bash
emcc -sEXPORTED_FUNCTIONS=["_malloc","_free"] -o lib.js lib.c
```

```html
<script src="lib.js"></script>
<script>
  Module.onRuntimeInitialized = function () {
    // 第一次调用先获取需要的内存大小
    var size = Module._get_c_string(0, 0);
    var strPointer = Module._malloc(size);
    Module._get_c_string(strPointer, size);
    var strBuffer = new Uint8Array(Module.HEAPU8.buffer, strPointer, size - 1);
    var str = new TextDecoder().decode(strBuffer);
    console.log(str); // Hello World
    Module._free(strPointer);
  };
</script>
```

原则上内存是谁分配的就由谁来释放。

## embind

第二种办法是使用 Emscripten 官方提供的 [Embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#embind) 来绑定 C++ 的函数和类到 JavaScript 对象，写起来更自然，类似 Node.js 的 NAPI，没有了传参类型的限制。

使用这个特性时必须用 C++ 语言，编译命令必须加上链接器选项 `--bind`。

```cpp
#include <string>  // std::string
#include <iostream>  // std::cout
#include <emscripten/bind.h>

int Add(int a, int b) {
  return a + b;
}

void LogJsString(const std::string& str) {
  std::cout << str << "\n";
}

std::string GetCppString() {
  return "Hello World";
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("add", Add);
  emscripten::function("logJsString", LogJsString);
  emscripten::function("getCppString", GetCppString);
}
```

```bash
# -sDISABLE_EXCEPTION_CATCHING=0 启用 C++ 异常
# -sALLOW_MEMORY_GROWTH=1 内存超出初始化的大小时自动开辟新内存
# --bind 启用 embind
em++ -sDISABLE_EXCEPTION_CATCHING=0 \
     -sALLOW_MEMORY_GROWTH=1 \
     --bind \
     -o lib.js \
     lib.cpp
```

```html
<script src="lib.js"></script>
<script>
  Module.onRuntimeInitialized = function () {
    console.log(Module.add(3, 4)); // 7
    Module.logJsString('Hello World'); // Hello World
    var str = Module.getCppString();
    console.log(str); // Hello World
  };
</script>
```

### 类型映射

上面 `Add` 函数用到的类型 `int`，Embind 可以自动映射成 JS 的 `number` 类型，用 TypeScript 声明来描述的话相当于：

``` ts
export declare function add (a: number, b: number): number
```

也就是说 JS 调用的时候可以传 `number` 类型进来，如果传别的类型就会抛错。

下表是 Embind 支持的类型映射：

| C++ 类型   | JavaScript 类型   |
|:---------- | -----------------|
| `void` | `undefined` |
| `bool` | `boolean` |
| `char` | `number` |
| `signed char` | `number` |
| `unsigned char` | `number` |
| `short` | `number` |
| `unsigned short` | `number` |
| `int` | `number` |
| `unsigned int` | `number` |
| `long` | `number` |
| `unsigned long` | `number` |
| `float` | `number` |
| `double` | `number` |
| `std::string` | `string \| ArrayBuffer \| Uint8Array \| Uint8ClampedArray \| Int8Array` |
| `std::wstring` | `string (utf-16)` |
| `emscripten::val` | `any` |

值得注意的是 `emscripten::val` 这个类，定义在 `<emscripten/val.h>` 里面，它可以映射成任意 JS 类型，相当于是 NAPI 的 `Napi::Value`，可以用它来直接操作 JS 对象。

比如这样用：

```cpp
#include <string>
#include <emscripten/val.h>

std::string stringify(const emscripten::val& jsobj) {
  if (jsobj.isString()) {
    return jsobj.as<std::string>();
  }
  emscripten::val result = emscripten::val::global("JSON")
    .call<emscripten::val>("stringify", jsobj);
  
  if (result.isUndefined()) {
    return "";
  }
  return result.as<std::string>();
}
```

等价于：

```ts
function stringify (jsobj: any): string {
  if (typeof jsobj === 'string') {
    return jsobj
  }
  const result = JSON.stringify(jsobj)
  if (result === undefined) {
    return ''
  }
  return result
}
```

## Node-API Emscripten 实现

第三种方法是使用 Node.js 原生扩展的 API，官方没有提供，我自己实现了一套 [emnapi](https://github.com/toyobayashi/emnapi)，方便一套代码同时编译到 WebAssembly 和 Node 原生扩展。具体写法请参照代码仓库的 README 和 Node.js 官方文档。

# 从 C/C++ 调用 JavaScript 函数

使用 embind 或 Node-API 很容易做到，这里不做介绍。重点介绍 Emscripten 的 JS Library 写法。

1. 写一个 JavaScript 文件 `library_add.js`

    ```js
    mergeInto(LibraryManager.library, {
      add: function (a, b) {
        return a + b;
      }
    });
    ```

    这个文件是 Emscripten 编译时去运行的，只有函数体的内容会被内联进最终的运行时 JS，生成的内容：

    ```js
    function _add (a, b) {
      return a + b;
    }
    // _add 会被加入传入 WebAssembly 初始化对象中
    ```

2. 在 C/C++ 中声明函数后直接使用

    ```c
    // main.c

    #include <stdio.h>  // printf

    #ifdef __cplusplus
    #define EXTERN_C extern "C"
    #else
    #define EXTERN_C
    #endif

    EXTERN_C int add(int a, int b);

    int main() {
      printf("%d\n", add(3, 4));
      return 0;
    }
    ```

3. 编译命令

    ```bash
    # --js-library 可以重复多个，链接时要链接的 JavaScript library
    emcc --js-library=library_add.js -o main.js main.c
    ```

4. HTML

    ```html
    <script src="main.js"></script>
    ```
