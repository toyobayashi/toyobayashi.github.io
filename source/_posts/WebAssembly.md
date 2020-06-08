---
title: 用 Emscripten 把 C++ 编译成 WebAssembly
date: 2020-06-08 15:49:01
tags: [学习笔记]
---

简要记录如何使用 Emscripten 把 C++ 编译成 WebAssembly。

<!-- more -->

虽然 WebAssembly 还处在发展中阶段，但是已经可以提前玩起来了。把 C++ 编译成 WASM 需要 [Emscripten](https://emscripten.org/) 编译器。

# 安装

需要 Git 和**良好的网络环境**。

非 Windows：

``` bash
$ git clone https://github.com/emscripten-core/emsdk.git
$ cd emsdk
$ ./emsdk install latest
$ ./emsdk activate latest
$ source ./emsdk_env.sh
```

Windows：

``` bat
> git clone https://github.com/emscripten-core/emsdk.git
> cd emsdk
> emsdk install latest
> emsdk activate latest
> emsdk_env
```

为了方便也可以顺带配一下 PATH 环境变量。

```
C:\path\to\emsdk
C:\path\to\emsdk\upstream\emscripten
```

如果不用 emsdk_env 初始化环境变量而是手动配置，Python 所在的目录也需要在 PATH 变量中。

# C++ 胶水

## 例子

Emscripten 提供了 [Embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#embind) 来绑定 C++ 的函数和类到 JavaScript 对象，写起来更自然，类似 Node.js 的 NAPI。

使用这个特性时必须加上链接器选项 `--bind`。

原生代码：

``` cpp
// add.cpp
#include <emscripten/bind.h>

int add(int a, int b) {
  return a + b;
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("add", add);
}
```

编译链接生成 js 和 wasm 文件：

``` bash
$ em++ -std=c++11 -s DISABLE_EXCEPTION_CATCHING=0 -s ALLOW_MEMORY_GROWTH=1 -O3 --bind -o add.js add.cpp
```

gcc 的参数基本都可以用，这里的 `-s` 是 Emscripten 额外的选项，`DISABLE_EXCEPTION_CATCHING=0` 可以正常 catch 到 C++ 异常，`ALLOW_MEMORY_GROWTH=1` 可以让 WebAssembly 内存超出初始化的大小时自动开辟新内存。

文档在[这里](https://emscripten.org/docs/tools_reference/emcc.html)。

JS 调用：

``` html
<!doctype html>
<html>
<script>
/**
 * JS 胶水会取全局的 Module 对象，
 * 在原生代码初始化完成以后把暴露的接口挂在 Module 上，
 * 并调用 onRuntimeInitialized 回调
 */
var Module = {
  onRuntimeInitialized: function () {
    console.log(Module.add(1, 2)); // => 3
  }
};
</script>
<script src="add.js"></script>
<script>
// 同步直接调用是不行的，此时 add 并不是 Module 的属性
// console.log(Module.add(1, 2));
</script>
</html>
```

默认输出的 JS 也支持 Node.js 运行环境，但是最好不要直接用在 Webpack 里，因为它里面用到了很多 Node.js 变量，Webpack 会自动导入 Node Polyfill 导致生成的包体积超大。

``` js
const Module = require('./add.js')

Module.onRuntimeInitialized = function () {
  console.log(Module.add(1, 2))
}
```

如果需要输出 ES6 模块格式的 JS，需要指定 `-o add.mjs`，然后这样用

``` js
import main from './add.mjs'

const Module = {
  onRuntimeInitialized () {
    console.log(Module.add(1, 2))
  }
}

main(Module)
```

ES6 模块可以用在 Webpack 里，但是要注意 `import.meta.url` 的处理。

## 类型映射

上面 `add` 函数用到的类型 `int`，Embind 可以自动映射成 JS 的 `number` 类型，用 TypeScript 声明来描述的话相当于：

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

值得注意的是 `emscripten::val` 这个类，定义在 `<emscripten/val.h>` 里面，它可以映射成任意 JS 类型，相当于是 NAPI 的 `Napi::Value`，很好用，可以用它来直接操作 JS 对象。

比如这样用：

``` cpp
#include <string>
#include <emscripten/val.h>

std::string stringify(const emscripten::val& jsobj) {
  if (jsobj.isString()) {
    return jsobj.as<std::string>();
  }
  emscripten::val result = emscripten::val::global("JSON").call<emscripten::val>("stringify", jsobj);
  
  if (result.isUndefined()) {
    return "";
  }
  return result.as<std::string>();
}
```

等价于：

``` ts
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

# 用 CMake 构建

非 Windows：

``` bash
$ mkdir -p ./build
$ cd ./build
$ cmake -DCMAKE_TOOLCHAIN_FILE=<EmscriptenRoot>/cmake/Modules/Platform/Emscripten.cmake
        -DCMAKE_BUILD_TYPE=<Debug|RelWithDebInfo|Release|MinSizeRel>
        -G "Unix Makefiles"
$ cmake --build .
```

Windows:

需要安装 [Make for Windows](http://gnuwin32.sourceforge.net/packages/make.htm) 跑 CMake 生成的 Makefile。

``` bat
> mkdir build
> cd build
> cmake -DCMAKE_TOOLCHAIN_FILE=<EmscriptenRoot>\cmake\Modules\Platform\Emscripten.cmake
        -DCMAKE_BUILD_TYPE=<Debug|RelWithDebInfo|Release|MinSizeRel>
        -G "MinGW Makefiles" -DCMAKE_MAKE_PROGRAM=make ..
> cmake --build .
```

# 调试

不要用 `-O` 参数，加上 `-g4 --source-map-base http://<host>:<port>/`，host 和 port 自己填。把 map 放在正确的位置即可在浏览器开发者工具中给 C++ 代码打断点调试。
