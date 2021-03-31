---
title: 在 Android 应用中嵌入 Node.js
date: 2021-03-29 17:12:23
tags: ['学习笔记', 'C++']
---

本文介绍如何把 Node.js 编译成安卓平台的动态链接库以及如何通过 JNI 调用 Node.js 的 C++ 接口。

<!-- more -->

# 编译成动态库

* 编译环境：Ubuntu 20.04
* Node.js 版本：目前最新的 LTS v14.16.0
* NDK 版本：21.0.6113669
* Python 版本：3.8.5

去 Github 上把这个版本的源码下载下来解压，打开 `android-configure` 文件，在倒数第三行加一个反斜杠 `\`，倒数第二行加上 `--shared` 参数：

```sh
if [ -f "configure" ]; then
    ./configure \
        --dest-cpu=$DEST_CPU \
        --dest-os=android \
        --without-snapshot \
        --openssl-no-asm \
        --cross-compiling \
        --shared
fi
```

在终端 cd 进源码目录，运行编译脚本：

```bash
# ./android-configure <NDK目录> <目标架构> <安卓SDK版本>
./android-configure ~/ndk/path arm64 23
make
```

编译要花点时间，耐心点等编译完了会输出 `out/Release/libnode.so`。

在源码根目录运行以下命令，会把头文件生成输出在 `headers` 目录：

```bash
HEADERS_ONLY=1 python3 ./tools/install.py install headers /
```

# Android 工程以及 CMake 配置

创建工程选 `Native C++` 模板。

App 模块的 Gradle 配置加上 `externalNativeBuild`：

```gradle
android {
    // ...
    defaultConfig {
        // ...
        externalNativeBuild {
            cmake {
                cppFlags ""
                // 必须要选 c++_shared，才会把 libc++_shared.so 打包进 apk里
                arguments "-DANDROID_STL=c++_shared"
            }
        }
    }

    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
            version "3.10.2"
        }
    }
}
```

把编译好的动态库和生成的头文件丢进安卓项目里，比如

把动态库放在 `src/main/cpp/lib/libnode.so`

把头文件放在 `src/main/cpp/include/node`

CMakeLists.txt 就这样写：

```cmake
# 创建动态库编译目标 native-lib
add_library(native-lib SHARED
    # 下面是源文件列表，空格回车都可以分割
    xxx.cpp
    yyy.cpp
    zzz.cpp)

# native-lib 目标的私有包含目录
target_include_directories(native-lib PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include/node)

# 安卓的 logcat 库，保存在 log-lib 变量中
find_library(log-lib log)

# 创建预编译动态库目标，名字是 imported-node-lib
add_library(imported-node-lib
    SHARED
    IMPORTED)

# 设置这个预编译库的路径
set_target_properties(imported-node-lib
    PROPERTIES IMPORTED_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/lib/libnode.so)

# 把要用的库链接到 native-lib
target_link_libraries(native-lib
    imported-node-lib
    ${log-lib})
```

# Node.js C++ API

头文件和库都有了，可以开始写代码了。

嵌入 C++ 代码可以参考[官方文档](https://github.com/nodejs/node/blob/v14.16.0/doc/api/embedding.md)。

## 进程初始化

要使用 Node.js，首先要初始化 libuv & Node.js & v8 引擎，每个**进程**只能初始化一次。

```cpp
// 全局假的命令行参数和 v8 platform 单例
namespace {
  std::vector<std::string> args = { "node" };
  std::vector<std::string> exec_args;
  std::vector<std::string> errors;
  std::unique_ptr<node::MultiIsolatePlatform> platform;
}

// std::vector<std::string> 转 char**，这里不给出实现
class Args {
public:
  Args() noexcept;
  ~Args();
  Args(const Args&) = delete;
  Args& operator=(const Args&) = delete;
  void parse(const std::vector<std::string>& arr);
  char** data() const noexcept;
  size_t size() const noexcept;
private:
  char** p;
  size_t len;
};

void _register_android(); // 后面说
int initializeNode() {
  Args cliArgs;
  cliArgs.parse(args);
  uv_setup_args(cliArgs.size(), cliArgs.data());

  // 在初始化前注册原生模块
  // JS 通过 process._linkedBinding() 访问
  _register_android();

  int exit_code = node::InitializeNodeWithArgs(&args, &exec_args, &errors);
  for (const std::string& error : errors)
    fprintf(stderr, "%s: %s\n", args[0].c_str(), error.c_str());
  if (exit_code != 0) {
    return exit_code;
  }

  platform = node::MultiIsolatePlatform::Create(4);
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();
  return 0;
}
```

JNI 胶水：

```cpp
// JNI 相关，用于 JS 的原生函数里调 Java
struct JNIContext {
  JNIEnv* env;
  jobject ctx;
  JNIContext(): env(nullptr), ctx(nullptr) {}
};

JNIContext jnictx;

extern "C" JNIEXPORT jint JNICALL
Java_com_github_toyobayashi_nodeexample_NodeJs_setupNode(JNIEnv *env, jclass clazz, jobject ctx) {
  if (jnictx.ctx != nullptr) {
    env->DeleteGlobalRef(jnictx.ctx);
  }
  jnictx.env = env;
  jnictx.ctx = env->NewGlobalRef(ctx);
  return initializeNode();
}
```

```java
public class NodeJs {
  static {
    System.loadLibrary("native-lib");
  }

  public native static int setupNode(Context ctx);
  // ...
}

public class App extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    // 初始化 Node.js
    NodeJs.setupNode(this);
  }
}
```

```xml
<!-- AndroidManifest.xml -->
<application
  android:name=".App" />
```

## 运行 Node.js 实例

初始化完了以后，就可以创建 Node.js 实例，执行 JavaScript 代码，实现差不多是下面这样：

```cpp
using EvalCallback =
  std::function<void(const v8::Local<v8::Context>&, const v8::Local<v8::Value>&, void*)>;

std::mutex mu;

int runNodeInstance(
  const std::string& script,
  EvalCallback callback,
  void* data,
  std::string* errout
) {
  // 线程安全  上锁
  const std::lock_guard<std::mutex> lock(mu);
  // 初始化事件循环
  int exit_code = 0;
  uv_loop_t loop;
  int ret = uv_loop_init(&loop);
  if (ret != 0) {
    fprintf(stderr, "%s: Failed to initialize loop: %s\n",
            args[0].c_str(),
            uv_err_name(ret));
    return 1;
  }

  // 创建 v8 虚拟机
  std::shared_ptr<node::ArrayBufferAllocator> allocator = node::ArrayBufferAllocator::Create();
  v8::Isolate* isolate = node::NewIsolate(allocator, &loop, platform.get());
  if (isolate == nullptr) {
    fprintf(stderr, "%s: Failed to initialize V8 Isolate\n", args[0].c_str());
    return 1;
  }

  {
    v8::Locker locker(isolate);
    v8::Isolate::Scope isolate_scope(isolate);

    std::unique_ptr<node::IsolateData, decltype(&node::FreeIsolateData)> isolate_data(
        node::CreateIsolateData(isolate, &loop, platform.get(), allocator.get()),
        node::FreeIsolateData);

    v8::HandleScope handle_scope(isolate);
    v8::Local<v8::Context> context = node::NewContext(isolate);
    if (context.IsEmpty()) {
      fprintf(stderr, "%s: Failed to initialize V8 Context\n", args[0].c_str());
      return 1;
    }

    // 要先进入 v8 上下文作用域
    v8::Context::Scope context_scope(context);

    // 创建 Node.js 环境实例
    std::unique_ptr<node::Environment, decltype(&node::FreeEnvironment)> env(
        node::CreateEnvironment(isolate_data.get(), context, args, exec_args),
        node::FreeEnvironment);

    v8::TryCatch trycatch(isolate); // 捕获 JS 异常

    // 设置 Node.js 环境实例
    // 重写 console.log 方法，打印到 logcat，并把 require 函数挂到 global 上
    // androidLogd 是前文 _register_android() 加进去的，后面讲
    v8::MaybeLocal<v8::Value> loadenv_ret = node::LoadEnvironment(env.get(), 
      "(function () {"
        "const androidLogd = process._linkedBinding('android').androidLogd;"
        "const log = function (...args) { androidLogd(require('util').format(...args)) };"
        "console.log = log;"
        "console.info = log;"
        "console.debug = log;"
        "console.warn = log;"
        "console.error = log;"
      "})();"
      "globalThis.require = require('module').createRequire(process.cwd() + '/');");

    if (loadenv_ret.IsEmpty()) {
      // JS 抛错了
      if (trycatch.HasCaught()) {
        v8::String::Utf8Value err(isolate, trycatch.Exception());
        if (errout) *errout = ToCString(err);
      }
      return 1;
    }

    // 使用 v8 引擎编译运行传进来的 JS 脚本字符串
    v8::Local<v8::String> runScript = v8::String::NewFromUtf8(isolate, script.c_str()).ToLocalChecked();
    auto maybe_script = v8::Script::Compile(context, runScript);
    if (maybe_script.IsEmpty()) {
      return 1;
    }

    auto script_result = maybe_script.ToLocalChecked()->Run(context);
    if (script_result.IsEmpty()) {
      if (trycatch.HasCaught()) {
        v8::String::Utf8Value err(isolate, trycatch.Exception());
        if (errout) *errout = ToCString(err);
      }
      return 1;
    }

    if (callback) {
      callback(context, script_result.ToLocalChecked(), data);
    }

    {
      // 开启事件循环，处理 JS 的异步任务
      v8::SealHandleScope seal(isolate);
      bool more;
      do {
        uv_run(&loop, UV_RUN_DEFAULT);

        platform->DrainTasks(isolate);
        more = uv_loop_alive(&loop);
        if (more) continue;

        node::EmitBeforeExit(env.get());
        more = uv_loop_alive(&loop);
      } while (more == true);
    }

    exit_code = node::EmitExit(env.get());

    node::Stop(env.get());
  }

  // 代码跑完了，销毁 v8 虚拟机，关闭事件循环
  bool platform_finished = false;
  platform->AddIsolateFinishedCallback(isolate, [](void* data) {
    *static_cast<bool*>(data) = true;
  }, &platform_finished);
  platform->UnregisterIsolate(isolate);
  isolate->Dispose();

  while (!platform_finished)
    uv_run(&loop, UV_RUN_ONCE);
  int err = uv_loop_close(&loop);
  assert(err == 0);

  return exit_code;
}
```

# Java 调用 JS

运行 Java 传过来的 JS 脚本字符串，得到一个 double 值：

```cpp
std::string runInThisContext(const std::string& script) {
  return "(function(){ return require('vm').runInThisContext('" + std::regex_replace(script, std::regex("'"), "\\'") + "') })()";
}

extern "C"
JNIEXPORT jdouble JNICALL
Java_com_github_toyobayashi_nodeexample_NodeJs_evalDouble(JNIEnv *env, jobject instance, jstring script) {
  std::string errmsg;
  double result = 0;
  const char* scriptString = env->GetStringUTFChars(script, JNI_FALSE);
  int r = runNodeInstance(
    runInThisContext(scriptString),
    [](const v8::Local<v8::Context>& context, const v8::Local<v8::Value>& value, void* data) {
      if (data != nullptr && value->IsNumber()) {
        *static_cast<double*>(data) = value->NumberValue(context).ToChecked();
      }
    }, &result, &errmsg);
  env->ReleaseStringUTFChars(script, scriptString);
  if (r != 0) {
      env->ThrowNew(env->FindClass("java/lang/Exception"), errmsg.c_str());
      return 0;
  }
  return result;
}
```

```java
public class NodeJs {
  static {
    System.loadLibrary("native-lib");
  }

  public native static int setupNode(Context ctx);
  public native double evalDouble(String script) throws Exception;
}
```

运行 Java 传过来的 JS 脚本文件路径：

```cpp
std::string moduleLoadEntry(const std::string& path) {
  return "(function(){ const require = globalThis.require; delete globalThis.require; return require('module')._load('" + std::regex_replace(path, std::regex("'"), "\\'") + "', null, true) })()";
}

extern "C"
JNIEXPORT void JNICALL
Java_com_github_toyobayashi_nodeexample_NodeJs_runMain(JNIEnv *env, jobject instance, jstring path) {
  std::string errmsg;
  const char* pathString = env->GetStringUTFChars(path, JNI_FALSE);
  int r = runNodeInstance(
    moduleLoadEntry(pathString),
    EvalCallback{},
    nullptr, &errmsg);
  env->ReleaseStringUTFChars(path, pathString);
  if (r != 0) {
    env->ThrowNew(env->FindClass("java/lang/Exception"), errmsg.c_str());
  }
}
```

```java
public class NodeJs {
  static {
    System.loadLibrary("native-lib");
  }

  public native static int setupNode(Context ctx);
  public native void runMain(String path) throws Exception;
}
```

# JS 调用 Java

v8 引擎可以把 C++ 原生函数编译成 JS 函数，然后把它注册到 Node.js 的 linked binding，在 JS 中就可以使用 `process._linkedBinding()` 访问到原生函数，原生函数中通过 JNI 调用 Java 的类。

```cpp
JNIContext jnictx; // 回顾前文

// function toast (string) { Toast.makeText(context, string, Toast.LENGTH_SHORT) }
void toast(const v8::FunctionCallbackInfo<v8::Value>& args) {
  // 拿到当前的 v8 虚拟机实例
  v8::Isolate* isolate = args.GetIsolate();

  // 判断 JS 传参数量
  if (args.Length() < 1) {
    isolate->ThrowException(v8::Exception::TypeError(
        v8::String::NewFromUtf8(isolate, "missing message").ToLocalChecked()));
    return;
  }

  // 判断 JS 传参类型
  if (!args[0]->IsString()) {
    isolate->ThrowException(v8::Exception::TypeError(
        v8::String::NewFromUtf8(isolate, "message is not a string").ToLocalChecked()));
    return;
  }

  // 调 Java
  JNIEnv* env = jnictx.env;
  jclass Toast = env->FindClass("android/widget/Toast");
  jmethodID show = env->GetMethodID(Toast, "show", "()V");

  jmethodID makeText = env->GetStaticMethodID(Toast, "makeText", "(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;");
  jobject toastObject = env->CallStaticObjectMethod(Toast, makeText, jnictx.ctx, env->NewStringUTF(*v8::String::Utf8Value(isolate, args[0])), 0);
  env->CallVoidMethod(toastObject, show);
  args.GetReturnValue().Set(v8::Undefined(isolate));
}

// function logd (string) { Log.d(TAG, string) }
void logd(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  if (args.Length() < 1) {
    isolate->ThrowException(v8::Exception::TypeError(
        v8::String::NewFromUtf8(isolate, "missing message").ToLocalChecked()));
    return;
  }

  if (!args[0]->IsString()) {
    isolate->ThrowException(v8::Exception::TypeError(
        v8::String::NewFromUtf8(isolate, "message is not a string").ToLocalChecked()));
    return;
  }

  v8::String::Utf8Value str(isolate, args[0]);
  __android_log_write(ANDROID_LOG_DEBUG, LOG_TAG, *str);
  args.GetReturnValue().Set(v8::Undefined(isolate));
}

void init(
  v8::Local<v8::Object> exports,
  v8::Local<v8::Value> module,
  v8::Local<v8::Context> context,
  void* priv
) {
  v8::Isolate* isolate = context->GetIsolate();

  // exports.toast = toast;
  v8::Local<v8::FunctionTemplate> toastTemplate = v8::FunctionTemplate::New(isolate, toast);
  v8::Local<v8::Function> toastFunction = toastTemplate->GetFunction(context).ToLocalChecked();
  v8::Local<v8::String> toastName = v8::String::NewFromUtf8(isolate, "toast").ToLocalChecked();
  toastFunction->SetName(toastName);
  exports->Set(context, toastName, toastFunction).Check();

  // exports.androidLogd = logd;
  v8::Local<v8::FunctionTemplate> logTemplate = v8::FunctionTemplate::New(isolate, logd);
  v8::Local<v8::Function> logFunction = logTemplate->GetFunction(context).ToLocalChecked();
  v8::Local<v8::String> logName = v8::String::NewFromUtf8(isolate, "androidLogd").ToLocalChecked();
  logFunction->SetName(logName);
  exports->Set(context, logName, logFunction);
}

enum {
  NM_F_BUILTIN = 1 << 0,
  NM_F_LINKED = 1 << 1,
  NM_F_INTERNAL = 1 << 2,
  NM_F_DELETEME = 1 << 3,
};

// 注册 Node.js 原生绑定
static node::node_module _module = {
  NODE_MODULE_VERSION,
  NM_F_LINKED,
  nullptr,
  __FILE__,
  nullptr,
  (node::addon_context_register_func)(init),
  NODE_STRINGIFY(android),
  nullptr,
  nullptr
};
void _register_android() { node::node_module_register(&_module); }
```

这样 JS 就可以从 `process._linkedBinding('android')` 访问这两个原生函数了，从而实现调用到了 Java 的类。

# 源码仓库

所有源码在[这里](https://gitee.com/toyobayashi/NodeExample)
