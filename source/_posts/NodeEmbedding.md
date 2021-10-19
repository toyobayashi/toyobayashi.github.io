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
  std::vector<std::string> args;
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

int initializeNode() {
  Args cliArgs;
  args = { "node" };
  cliArgs.parse(args);
  uv_setup_args(cliArgs.size(), cliArgs.data());

  int exit_code = node::InitializeNodeWithArgs(&args, &exec_args, &errors);
  for (const std::string& error : errors)
    fprintf(stderr, "%s: %s\n", args[0].c_str(), error.c_str());
  if (exit_code != 0) {
    return exit_code;
  }

  platform = node::MultiIsolatePlatform::Create(4);
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();
  NodeInstance::Get(); // 创建 NodeInstance 单例
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

class NodeInstance {
public:
  uv_loop_t loop_;
  std::shared_ptr<node::ArrayBufferAllocator> allocator_;
  v8::Isolate* isolate_;
  node::IsolateData* isolate_data_;
  node::Environment* env_;
  v8::Global<v8::Context> context_;
  std::mutex mu_;

  NodeInstance() noexcept:
    loop_(),
    allocator_(nullptr),
    isolate_(nullptr),
    isolate_data_(nullptr),
    env_(nullptr),
    context_(),
    mu_() {}

  int Eval(const std::string& script,
           const EvalCallback& callback,
           void* data,
           std::string* errout);

  void SpinEventLoop();

  static NodeInstance* instance;
  static NodeInstance* Get();
  static int Free();
};

NodeInstance* NodeInstance::instance = nullptr;

void NodeInstance::SpinEventLoop() {
  v8::SealHandleScope seal(isolate_);
  bool more;
  do {
    uv_run(&loop_, UV_RUN_DEFAULT);

    platform->DrainTasks(isolate_);
    more = uv_loop_alive(&loop_);
    if (more) continue;
    node::EmitBeforeExit(env_);
    more = uv_loop_alive(&loop_);
  } while (more);
}

NodeInstance* NodeInstance::Get() {
  if (instance != nullptr) return instance;
  std::unique_ptr<NodeInstance> node_instance = std::make_unique<NodeInstance>();
  int ret = uv_loop_init(&node_instance->loop_);
  if (ret != 0) {
    fprintf(stderr, "%s: Failed to initialize loop: %s\n",
            args[0].c_str(),
            uv_err_name(ret));
    return nullptr;
  }
  node_instance->allocator_ = node::ArrayBufferAllocator::Create();

  node_instance->isolate_ = node::NewIsolate(node_instance->allocator_, &node_instance->loop_, platform.get());
  if (node_instance->isolate_ == nullptr) {
    fprintf(stderr, "%s: Failed to initialize V8 Isolate\n", args[0].c_str());
    return nullptr;
  }

  v8::Locker locker(node_instance->isolate_);
  v8::Isolate::Scope isolate_scope(node_instance->isolate_);
  node_instance->isolate_data_ = node::CreateIsolateData(node_instance->isolate_, &node_instance->loop_, platform.get(), node_instance->allocator_.get());

  v8::HandleScope handle_scope(node_instance->isolate_);
  v8::Local<v8::Context> context = node::NewContext(node_instance->isolate_);
  node_instance->context_.Reset(node_instance->isolate_, context);

  if (context.IsEmpty()) {
    fprintf(stderr, "%s: Failed to initialize V8 Context\n", args[0].c_str());
    return nullptr;
  }

  v8::Context::Scope context_scope(context);
  node_instance->env_ = node::CreateEnvironment(node_instance->isolate_data_, context, args, exec_args);
  node::AddLinkedBinding(node_instance->env_, "android", init, nullptr); // 后文讲

  v8::TryCatch trycatch(node_instance->isolate_);
  v8::MaybeLocal<v8::Value> loadenv_ret = node::LoadEnvironment(node_instance->env_,
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
    // There has been a JS exception.
    if (trycatch.HasCaught()) {
      v8::String::Utf8Value err(node_instance->isolate_, trycatch.Exception());
      const char* errmsg = *err;
      fprintf(stderr, "%s\n", errmsg);
    }
    return nullptr;
  }
  node_instance->SpinEventLoop();
  instance = node_instance.release();
  return instance;
}

int NodeInstance::Free() {
  if (!instance) return 0;
  NodeInstance *node_instance = instance;
  v8::Isolate* isolate = node_instance->isolate_;
  bool platform_finished = false;
  int exit_code;

  {
    v8::Locker locker(isolate);
    v8::Isolate::Scope isolate_scope(isolate);
    exit_code = node::EmitExit(node_instance->env_);
    node::Stop(node_instance->env_);
    node::FreeEnvironment(node_instance->env_);

    platform->AddIsolateFinishedCallback(isolate, [](void *data) {
      *static_cast<bool *>(data) = true;
    }, &platform_finished);
    platform->UnregisterIsolate(isolate);
    node_instance->context_.Reset();
    node::FreeIsolateData(node_instance->isolate_data_);
  }

  isolate->Dispose();
  node_instance->allocator_.reset();
  while (!platform_finished)
    uv_run(&node_instance->loop_, UV_RUN_ONCE);
  int err = uv_loop_close(&node_instance->loop_);
  assert(err == 0);
  instance = nullptr;
  return exit_code;
}

int NodeInstance::Eval(const std::string& script,
                       const EvalCallback& callback,
                       void* data,
                       std::string* errout) {
  const std::lock_guard<std::mutex> lock(mu_);
  v8::Locker locker(isolate_);
  v8::Isolate::Scope isolate_scope(isolate_);
  v8::HandleScope handle_scope(isolate_);
  v8::Local<v8::String> runScript = v8::String::NewFromUtf8(isolate_, script.c_str()).ToLocalChecked();
  auto context = context_.Get(isolate_);
  v8::Context::Scope context_scope(context);
  auto maybe_script = v8::Script::Compile(context, runScript);
  if (maybe_script.IsEmpty()) {
    return 1;
  }

  v8::TryCatch trycatch(isolate_);
  auto script_result = maybe_script.ToLocalChecked()->Run(context);
  if (script_result.IsEmpty()) {
    if (trycatch.HasCaught()) {
      v8::String::Utf8Value err(isolate_, trycatch.Exception());
      if (errout) *errout = ToCString(err);
    }
    return 1;
  }

  if (callback) {
    callback(context, script_result.ToLocalChecked(), data);
  }
  SpinEventLoop();
  return 0;
}
```

# Java 调用 JS

运行 Java 传过来的 JS 脚本字符串，得到一个 double 值：

```cpp
extern "C"
JNIEXPORT jdouble JNICALL
Java_com_github_toyobayashi_nodeexample_NodeJs_evalDouble(JNIEnv *env, jobject instance, jstring script) {
  std::string errmsg;
  double result = 0;
  const char* scriptString = env->GetStringUTFChars(script, JNI_FALSE);
  int r = NodeInstance::Get()->Eval(
    scriptString,
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
  return "(function(){ return require('module')._load('" + std::regex_replace(path, std::regex("'"), "\\'") + "', null, true) })()";
}

extern "C"
JNIEXPORT void JNICALL
Java_com_github_toyobayashi_nodeexample_NodeJs_runMain(JNIEnv *env, jobject instance, jstring path) {
  std::string errmsg;
  const char* pathString = env->GetStringUTFChars(path, JNI_FALSE);
  int r = NodeInstance::Get()->Eval(
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

v8 引擎可以把 C++ 原生函数编译成 JS 函数，然后通过 `node::AddLinkedBinding` 把它注册到 Node.js 的 linked binding，在 JS 中就可以使用 `process._linkedBinding()` 访问到原生函数，原生函数中通过 JNI 调用 Java 的类。

`NodeInstance::Get()` 中有一处：

```cpp
node::AddLinkedBinding(node_instance->env_, "android", init, nullptr);
```

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
```

这样 JS 就可以从 `process._linkedBinding('android')` 访问这两个原生函数了，从而实现调用到了 Java 的类。

### Node-API

Linked Binding 也支持用 Node-API（以前称作 NAPI）来写

```cpp
napi_value NapiModuleInit(napi_env env, napi_value exports) {
  napi_value world;
  napi_create_string_utf8(env, "world", NAPI_AUTO_LENGTH, &world);
  napi_set_named_property(env, exports, "hello", world);
  return exports;
}

node::AddLinkedBinding(node_instance->env_, napi_module {
  NAPI_MODULE_VERSION,
  node::ModuleFlags::kLinked,
  __FILE__,
  NapiModuleInit,
  "napibinding",
  nullptr,
  {0}
});
```

# 源码仓库

所有源码在[这里](https://gitee.com/toyobayashi/NodeExample)
