---
title: 使用VSCode混合调试C++与Node.js
date: 2018-10-19 19:20:44
tags: ['学习笔记']
---

VSCode中C++和JS的混合调试。

<!-- more -->

# 效果图

二话不说先上图。

![混合调试](demo.gif)

# VSCode  

必须先吹一波VSCode，巨硬少有的良心作，用来写前端代码速度快，有弹性，体位好，手感棒，谁用谁知道，尤其是对JS和TS的支持非常棒。

其本身也是由自家TS写的Electron跨平台编辑器，没有Visual Studio的笨重，虽然它不是IDE，但是耍起来堪比IDE好使。

下面进入正题。

# Node.js C++ 原生模块  

Node.js允许我们用C++来开发原生模块。理论上来说C++能做的事大部分JS都能做，为什么还要用C++来写Node.js，我认为有以下几点：

1. 可以使用C++中JS所没有的特性，比如类的私有成员、多继承，大数运算等等
2. 可以提高性能，编码解码、加密解密等运算量大的工作C++的效率要比JS快得多
3. 可以使用现有的C++轮子，没有必要用JS再造一遍
4. 需要做的东西的的确确用JS不好写，用C++反而好写，又必须用上Node.js
5. 瞎玩

而我就是出于第5种原因。

其实让Node.js和C++搞起基来是非常容易的，我们先来点前戏。

# 准备

具体可以参考[node-gyp文档](https://www.npmjs.com/package/node-gyp)

* Visual Studio Code 1.22+
* Visual Studio Code C/C++ 扩展（Microsoft官方的，打开VSCode扩展页搜一下第一个应该就是）
* Node.js 8+ （这里我使用8版本以上才有的NAPI来写）
* node-gyp （原生模块跨平台构建工具，通过npm安装）
* Python 2.7 （node-gyp需要，Python 3不行）
* .NET 4.5.1 (非Win7忽略这一项)
* Visual Studio 2015/2017，或单独安装MSBuild和VC++工具集140/141（非Windows系统忽略这一项）
* gcc （非Linux系统忽略这一项）
* Xcode （非Mac系统忽略这一项）

准备好了就开始搞起。拿Windows平台做个示范，全平台通用。

# 正片

## package.json

新建个目录，创建`package.json`，`npm install`走起

``` json
{
  // ...
  "scripts": {
    "build:debug": "node-gyp configure && node-gyp build --debug",
    "build:release": "node-gyp configure && node-gyp build",
    "clean": "node-gyp clean"
  },
  "devDependencies": {
    "node-gyp": "^3.8.0"
  },
  "dependencies": {
    "node-addon-api": "*"
  }
}
```

`build`脚本执行构建完会在根目录生成build目录，`clean`用于清理生成，`node-addon-api`这个库用C++封装了C风格的NAPI，写起来容易点。

## binding.gyp  

`node-gyp`会根据根目录下的`binding.gyp`配置来生成项目，`binding.gyp`看起来是json格式，其实是Python的字典和列表。

``` gyp
{
  # 构建目标集合
  "targets": [
    {
      # 模块最终生成的二进制文件名
      "target_name": "VscodeCppJsDebugDemo",
      # 要编译的源文件
      "sources": [
        "./src/main.cpp",
        "./src/DemoAsyncWorker.cpp"
      ],
      # 头文件包含目录，!是执行shell命令取输出值，@是在列表中展开输出的每一项
      "include_dirs": ["<!@(node -p \"require('node-addon-api').include\")"],
      # 外部依赖项
      "dependencies": ["<!(node -p \"require('node-addon-api').gyp\")"],
      # 以下是编译器选项，启用node-addon-api的集成C++和JavaScript的异常处理
      "cflags!": [ "-fno-exceptions" ],
      "cflags_cc!": [ "-fno-exceptions" ],
      "xcode_settings": {
        "GCC_ENABLE_CPP_EXCEPTIONS": "YES",
        "CLANG_CXX_LIBRARY": "libc++",
        "MACOSX_DEPLOYMENT_TARGET": "10.7"
      },
      "msvs_settings": {
        "VCCLCompilerTool": {
          "ExceptionHandling": 1
        }
      },
      # 预定义宏，禁用NAPI的C++异常处理和node-addon-api废弃的API
      "defines": ["NAPI_DISABLE_CPP_EXCEPTIONS", "NODE_ADDON_API_DISABLE_DEPRECATED"]
    }
  ]
}
```

简单说，有了这个配置文件，`node-gyp`就会分别对不同操作系统下的编译器做不需要我们关心的跨平台配置，编译`src`目录下的两个源文件，最终生成`VscodeCppJsDebugDemo.node`二进制文件，其实就是个动态链接库。

## index.js

既然都要用到C++了，那么在大多数情况需求都应该是耗时的异步操作，如数据库的读写，音频流的处理等等，JS调用的时候应该是异步的，那就用C++写个异步函数吧，模仿耗时操作。

根目录下新建`index.js`。

``` javascript
const asyncDemo = require('./build/Debug/VscodeCppJsDebugDemo.node') // 打断点

asyncDemo((str) => { // 打断点
  const msg = 'asyncDemo() ' + str // 打断点
  console.log(msg) // 打断点
})

console.log('[JavaScript] Call console.log()') // 打断点
```

引入原生模块，原生模块暴露的是一个异步函数，执行它后回调函数将在1秒后执行并且不阻塞JS线程，回调函数的参数是一个字符串。为了更直观的调试效果，在这5行都打上断断点。

## src/main.cpp

在`src`目录下新建`main.cpp`，看注释。

``` cpp
// 包含DemoAsyncWorker类的头文件，实现将在后面给出
#include "./DemoAsyncWorker.h"

using namespace Napi;

// 要暴露给JS的函数
static Value _asyncDemo(const CallbackInfo& info) {
  Env env = info.Env(); // 类似V8中的Isolate

  if (info.Length() != 1) { // 检查是否只传入一个参数
    Error::New(env, "arguments.length !== 1").ThrowAsJavaScriptException();
    return env.Undefined();
  }

  if (!info[0].IsFunction()) { // 检查第一个参数是否为函数
    Error::New(env, "typeof arguments[0] !== 'function'").ThrowAsJavaScriptException();
    return env.Undefined();
  }

  Function callback = info[0].As<Function>(); // 把Napi::Value强转成Napi::Function类型
  DemoAsyncWorker *w = new DemoAsyncWorker(callback); // 把回调传给另外写好的DemoAsyncWorker类
  w->Queue(); // 入队执行，这里打个断点

  return env.Undefined();
}

// 模块入口，被require或者被当作Node.js入口文件执行的时候首先进入这里
Object init(Env env, Object exports) {
  Object global = env.Global(); // 获取JS的global对象
  Object console = global.Get("console").As<Object>(); // 获取global对象下的console对象
  Function log = console.Get("log").As<Function>(); // 获取console对象下的log函数
  log.Call(console, { String::New(env, "[C++] Call console.log()") }); // 执行log函数，console为this，参数为'[C++] Call console.log()'，这里打个断点
  Function module = Function::New(env, _asyncDemo); // 把C++函数包装成JS函数
  return module; // 暴露JS函数。Napi::Function继承自Napi::Object，所以可以直接返回
}

// NODE_API_MODULE是个宏，用于注册原生模块
// NODE_GYP_MODULE_NAME宏由node-gyp预定义，就是binding.gyp里的target_name
// 把模块入口函数传进去，注意没有分号
NODE_API_MODULE(NODE_GYP_MODULE_NAME, init)
```

注意到21行，为什么在堆内存上new了一个Worker对象，在`w->Queue()`完了以后不用`delete w`，点进源码里面去看会发现，这个Worker对象在执行完异步操作后会自杀，即使不自杀，我们也不可以直接把它delete掉，因为排队等待是异步的，直接delete后回调根本不能执行，甚至由于指针操作不当程序会崩溃。附上对象自杀代码：

``` cpp napi-inl.h
inline void AsyncWorker::OnWorkComplete(napi_env /*env*/, napi_status status, void* this_pointer) {
  AsyncWorker* self = static_cast<AsyncWorker*>(this_pointer);
  if (status != napi_cancelled) {
    HandleScope scope(self->_env);
    details::WrapCallback([&] {
      if (self->_error.size() == 0) {
        self->OnOK();
      }
      else {
        self->OnError(Error::New(self->_env, self->_error));
      }
      return nullptr;
    });
  }
  delete self;
}
```


## src/DemoAsyncWorker.h

在`src`目录下新建`DemoAsyncWorker.h`，声明DemoAsyncWorker类，看注释。

``` h
#ifndef __DEMO_ASYNC_WORKER_H__
#define __DEMO_ASYNC_WORKER_H__

// 包含node-addon-api的头文件
#include <napi.h>

// 要实现异步必须继承Napi::AsyncWorker类，该类的内部会调用NAPI开启子线程
class DemoAsyncWorker : public Napi::AsyncWorker {
  public:
    // 构造函数传入JS的回调函数
    DemoAsyncWorker(Napi::Function&);
    // 析构函数
    ~DemoAsyncWorker();
    // 子线程下执行异步操作
    void Execute();
    // 异步操作执行完成的回调
    void OnOK();
};

#endif // ! __DEMO_ASYNC_WORKER_H__
```

## src/DemoAsyncWorker.cpp

在`src`目录下新建`DemoAsyncWorker.cpp`，实现DemoAsyncWorker类，看注释。

``` cpp
#include "./DemoAsyncWorker.h"
#include <thread>
#include <chrono>

using namespace Napi;

// 构造函数中把JS回调传给基类构造函数
DemoAsyncWorker::DemoAsyncWorker(Function& callback): AsyncWorker(callback) {}
// 析构函数啥也不干
DemoAsyncWorker::~DemoAsyncWorker() {}
// 子线程中等待1秒
void DemoAsyncWorker::Execute() {
  std::this_thread::sleep_for(std::chrono::milliseconds(1000)); // 打断点
}
// 异步操作完成，执行JS回调，传入'callback.'字符串，JS中将接收到这个字符串
void DemoAsyncWorker::OnOK() {
  Callback().Call({ String::New(Env(), "callback.") }); // 打断点
}
```

## .vscode/launch.json

万事预备，只差靓妹。接着就要配置VSCode的Debugger了。

在`.vscode`目录下新建`launch.json`，这个json文件就是VSCode Debugger的配置文件。

``` json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "JS Debug Build",
      "console": "integratedTerminal",
      "program": "${workspaceFolder}/index.js",
      "preLaunchTask": "npm: build:debug"
    },
    {
      "name": "Windows Attach",
      "type": "cppvsdbg",
      "request": "attach",
      "processId": "${command:pickProcess}"
    }
  ]
}
```

重点看`configurations`数组，这里面的每一项，都是一个启动项。`request`可以是`launch`（启动）或`attach`（附加），`console`是终端选项，设置为`integratedTerminal`则使用VSCode的内部集成终端显示调试结果，`preLaunchTask`是启动前执行的任务，就是`package.json`中的`scripts['build:debug']`。先编译生成项目，再启动node调试进程，入口是根目录的`index.js`。

启动起来以后，调试器会断在第一行，这时先不要急，切换到第二个启动项再点一下启动。关键就在这里，第二项配置的是Windows的C++调试器，其它平台也差不多，设置为附加到另一个调试进程，启动它后会弹出一个列表，输入`node`就会出现刚刚启动的调试进程，选择它后就可以让C++代码也进入调试，可以看到调试器下拉菜单中多了一项，我们可以随时切换调试。[效果演示GIF图](#效果图)在本文最开始。

到此为止，就实现了用VSCode混合调试C++和JS代码。

## .vscode/tasks.json

设置快捷键`Ctrl + Shift + B`为Release生成，只要在`.vscode/tasks.json`做如下配置：

``` json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "script": "build:release",
      "problemMatcher": [],
      "group": {
        "kind": "build",
        "isDefault": true
      }
    }
  ]
}
```

# Git仓库

本文只讲解了JS的调试法，还有TS、Electron的调试配置，具体可以来看[这个仓库](https://github.com/toyobayashi/vscode-cpp-js-debug-demo)。

# 参考

* [node-addon-api](https://github.com/nodejs/node-addon-api)
* [gyp文件输入格式参考](https://itbilu.com/nodejs/npm/By7L5p3ff.html)
