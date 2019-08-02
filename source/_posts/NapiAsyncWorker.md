---
title: 重写 node-addon-api 的 AsyncWorker 类
date: 2019-08-02 15:08:55
tags: ['学习笔记']
---

AsyncWorker 不支持在子线程中多次回调 JS 传过来的回调函数，所以，嗯，撸起袖子就是干。

<!-- more -->

# 为什么要重写

在使用 node-addon-api 开发原生模块时，涉及到异步回调 JS 函数的操作一般都是用 `Napi::AsyncWorker` 类去做，但是 `Napi::AsyncWorker` 并不支持在子线程中多次执行 JS 传过来的回调函数，在某些场景下（比如 JS 要知道异步操作的进度并更新进度条）就有了一定的限制。

为了能够满足这些场景下的需求，就必须要在异步耗时的操作中多次执行 JS 回调。node-addon-api 官方提供了 `Napi::ThreadSafeFunction` 这个类来满足在多个线程中执行 JS 回调的需求，但是它仅仅是 `napi_threadsafe_function` 的 C++ Wrapper，没有与 `Napi::AsyncWorker` 配合得很好，所以还是决定参考 `Napi::AsyncWorker` 的实现另外写一个 `ThreadSafeAsyncWorker`。

# 类声明

注释很详细。

``` cpp ThreadSafeAsyncWorker.h
// ThreadSafeAsyncWorker.h
#include <napi.h>

class ThreadSafeAsyncWorker {
public:
  // 虚析构，可继承
  virtual ~ThreadSafeAsyncWorker();

  // 这个类只能移动不能复制，所以关掉编译器默认的构造函数和赋值重载
  ThreadSafeAsyncWorker(ThreadSafeAsyncWorker&& other);
  ThreadSafeAsyncWorker& operator =(ThreadSafeAsyncWorker&& other);
  ThreadSafeAsyncWorker(const ThreadSafeAsyncWorker&) = delete;
  ThreadSafeAsyncWorker& operator =(ThreadSafeAsyncWorker&) = delete;

  // 转换为 C 类型，方便配合 C 风格的 NAPI 使用
  operator napi_async_work() const;

  // 获取私有 V8 环境
  Napi::Env Env() const;

  // 任务入队执行，调用 napi_queue_async_work()，由底层的 libuv 开子线程
  void Queue();

  // 取消任务的执行，回调函数也不会被执行
  void Cancel();

  // 禁止实例自杀，必须由调用者手动 delete
  void SuppressDestruct();

  // 获取私有的 JS 回调引用
  Napi::FunctionReference& Callback();
  Napi::FunctionReference& ProgressCallback();

protected:
  // 显式构造函数，只能由子类继承时构造，进度回调可传可不传
  explicit ThreadSafeAsyncWorker(const Napi::Function& callback);
  explicit ThreadSafeAsyncWorker(const Napi::Function& callback, const Napi::Function& _progressCallback);

  // 纯虚的任务执行函数，子类必须给出实现，入队后在子线程中执行
  // 需要异步操作的逻辑写在这里面
  virtual void Execute() = 0;

  // 在 Execute() 中调用 EmitProgress(void* data) 后
  // napi_threadsafe_function 会把 CallJS 推进队列执行
  // CallJS 会在主线程调用这个函数
  // 无重写的默认行为是无参调用 JS 传过来的进度回调函数
  // 如果构造时没有传进度回调，这个函数不会执行
  // 重写这个函数把告诉 JS 进度的逻辑写在这里
  virtual void OnProgress(void* data);

  // 子线程执行完 Execute() 后的成功回调
  // 无重写的默认行为是无参调用 JS 传过来的回调函数
  // 私有成员 _error 有值时这个函数不会执行
  virtual void OnOK();

  // 子线程执行完 Execute() 后的失败回调
  // 无重写的默认行为是把 e 作为 JS 回调函数的第一个参数并执行 JS 回调
  // 私有成员 _error 无值时这个函数不会执行
  virtual void OnError(const Napi::Error& e);

  // 自杀
  virtual void Destroy();

  // 可以在 Execute() 中调用，赋值私有成员 _error
  // 子线程执行完 Execute() 后把错误传入 OnError()
  // 并且 OnOK 不会执行
  void SetError(const std::string& error);

  // 可以在 Execute() 中调用
  // napi_threadsafe_function 会把 CallJS 推进队列执行
  // CallJS 会在主线程调用 OnProgress 并传入 data
  // 如果构造时没有传进度回调，这个函数不做任何事
  void EmitProgress(void* data);

private:
  // 被传入 napi_create_async_work()
  // 这个函数在子线程执行
  static void OnExecute(napi_env env, void* this_pointer);

  // 被传入 napi_create_async_work()
  // 当 OnExecute() 执行完后这个函数在主线程执行
  static void OnWorkComplete(napi_env env, napi_status status, void* this_pointer);
  
  // 如果构造时传了进度回调
  // 这个函数会被传入 napi_create_threadsafe_function()
  // 当调用 EmitProgress() 时这个函数会入队执行（在主线程）
  // 这个函数里会调用 OnProgress()
  static void CallJS(napi_env env, napi_value jsCallback, void* this_pointer, void* data);

  // 存 V8 环境
  napi_env _env;

  // 存 NAPI 的 napi_async_work
  napi_async_work _work;

  // 存 NAPI 的 napi_threadsafe_function
  napi_threadsafe_function _tsfn;

  // 存进度回调，如果构造时不传 _progressCallback.IsEmpty() 会返回 true
  Napi::FunctionReference _progressCallback;

  // 存完成时的回调
  Napi::FunctionReference _callback;

  // 存错误信息，有值的话会执行 OnError() 不执行 OnOK()
  std::string _error;

  // 是否由调用者来释放内存
  bool _suppress_destruct;
};
```

# 关键实现

## 构造函数

两个构造函数，带进度回调和不带进度回调。

``` cpp
// 不带进度回调的构造函数，只创建 napi_async_work
inline ThreadSafeAsyncWorker::ThreadSafeAsyncWorker(const Napi::Function& callback)
  : _env(callback.Env()), _suppress_destruct(false), _callback(Napi::Persistent(callback)), _progressCallback(), _tsfn(nullptr) {
  napi_value resource_id;
  napi_status status = napi_create_string_latin1(
      _env, "generic", NAPI_AUTO_LENGTH, &resource_id);
  NAPI_THROW_IF_FAILED_VOID(_env, status);

  status = napi_create_async_work(
    _env,
    Napi::Object::New(callback.Env()), // resource 空对象
    resource_id, // resource id
    OnExecute, // 子线程跑
    OnWorkComplete, // 子线程跑完
    this, // 把实例对象的指针传给 OnExecute 和 OnWorkComplete
    &_work // 保存结果
  );
  NAPI_THROW_IF_FAILED_VOID(_env, status);
}

// 带进度回调的构造函数，创建 napi_async_work 和 napi_threadsafe_function
inline ThreadSafeAsyncWorker::ThreadSafeAsyncWorker(const Napi::Function& callback, const Napi::Function& progressCallback)
  : _env(callback.Env()), _suppress_destruct(false), _callback(Napi::Persistent(callback)), _progressCallback(Napi::Persistent(progressCallback)) {
  napi_value resource_id;
  napi_status status = napi_create_string_latin1(
      _env, "generic", NAPI_AUTO_LENGTH, &resource_id);
  NAPI_THROW_IF_FAILED_VOID(_env, status);

  status = napi_create_async_work(_env, Napi::Object::New(callback.Env()), resource_id, OnExecute,
                                  OnWorkComplete, this, &_work);
  NAPI_THROW_IF_FAILED_VOID(_env, status);

  status = napi_create_threadsafe_function(
    _env,
    _progressCallback.Value(), // 要在子线程执行的 JS 回调函数
    nullptr, // async resource
    Napi::String::New(_env, "N-API Thread-safe Call from Async Work Item"), // async resource name
    0, // 队列最大限制，设置为 0 则无限制.
    1, // 线程初始值，包括主线程在内，有几个线程会用到这个函数
    NULL, // 可选的 data，会被传给 thread_finalize_cb
    NULL, // 可选的 thread_finalize_cb，napi_threadsafe_function 被销毁时执行的回调函数
    this, // 可选的 data，会被传给自定义的数据处理回调函数
    CallJS, // 可选的自定义数据处理回调函数，这个函数会在主线程执行，如果不传，JS回调函数会被无参调用
    &_tsfn // 保存结果
  );
  NAPI_THROW_IF_FAILED_VOID(_env, status);
}
```

## 子线程执行与完成

需要注意执行是跑在子线程里的，完成是跑在主线程里的。

``` cpp
inline void ThreadSafeAsyncWorker::OnExecute(napi_env /*DO_NOT_USE*/, void* this_pointer) {
  ThreadSafeAsyncWorker* self = static_cast<ThreadSafeAsyncWorker*>(this_pointer);
  napi_status status;
  if (self->_tsfn != nullptr) {
    // 在当前线程中请求使用线程安全函数
    status = napi_acquire_threadsafe_function(self->_tsfn);
    if (status != napi_ok) {
      self->SetError("napi_acquire_threadsafe_function() failed.");
      return;
    }
  }
  
#ifdef NAPI_CPP_EXCEPTIONS
  try {
    self->Execute();
  } catch (const std::exception& e) {
    self->SetError(e.what());
  }
#else // NAPI_CPP_EXCEPTIONS
  self->Execute();
#endif // NAPI_CPP_EXCEPTIONS
  if (self->_tsfn != nullptr) {
    // 当前线程中不再需要使用线程安全函数，必须释放
    status = napi_release_threadsafe_function(self->_tsfn, napi_tsfn_release);
    if (status != napi_ok) {
      self->SetError("napi_release_threadsafe_function() failed.");
      return;
    }
  }
}

inline void ThreadSafeAsyncWorker::OnWorkComplete(
    napi_env /*env*/, napi_status status, void* this_pointer) {
  ThreadSafeAsyncWorker* self = static_cast<ThreadSafeAsyncWorker*>(this_pointer);
  if (status != napi_cancelled) {
    Napi::HandleScope scope(self->_env);
    // 包的这一层仅仅为了 catch 异常然后抛给 JS
    Napi::details::WrapCallback([&] {
      if (self->_error.size() == 0) {
        self->OnOK();
      } else {
        self->OnError(Napi::Error::New(self->_env, self->_error));
      }
      return nullptr;
    });
  }
  // 子线程执行完后，实例默认会自杀
  if (!self->_suppress_destruct) {
    self->Destroy(); // delete this;
  }
}
```

## 析构函数

析构函数释放底层的 NAPI 对象。

``` cpp
inline ThreadSafeAsyncWorker::~ThreadSafeAsyncWorker() {
  if (_work != nullptr) {
    napi_delete_async_work(_env, _work);
    _work = nullptr;
  }
  if (_tsfn != nullptr) {
    // 注意每个线程都必须释放线程安全函数，光子线程释放了还不行
    napi_release_threadsafe_function(_tsfn, napi_tsfn_release);
    _tsfn = nullptr;
  }
}
```

## 自定义线程安全数据处理

这个函数负责调用 OnProgress，调用者重写 OnProgress，在 OnProgress 中处理数据并多次回调 JS。需要注意如果没有进度回调函数这个函数什么也不做，如果 data 指向的内存是动态分配的，会无法释放造成内存泄漏。

``` cpp
inline void ThreadSafeAsyncWorker::CallJS(napi_env env, napi_value jsCallback, void* this_pointer, void* data) {
  if (env == nullptr && jsCallback == nullptr) {
    return;
  }

  ThreadSafeAsyncWorker* self = static_cast<ThreadSafeAsyncWorker*>(this_pointer);

#ifdef NAPI_CPP_EXCEPTIONS
  try {
    self->OnProgress(data);
  } catch (const std::exception& e) {
    self->SetError(e.what());
  }
#else // NAPI_CPP_EXCEPTIONS
  self->OnProgress(data);
#endif // NAPI_CPP_EXCEPTIONS
}
```

## Worker 的入队与取消

这没啥，就是调用 NAPI。

``` cpp
inline void ThreadSafeAsyncWorker::Queue() {
  napi_status status = napi_queue_async_work(_env, _work);
  NAPI_THROW_IF_FAILED_VOID(_env, status);
}

inline void ThreadSafeAsyncWorker::Cancel() {
  napi_status status = napi_cancel_async_work(_env, _work);
  NAPI_THROW_IF_FAILED_VOID(_env, status);
}
```

## 默认的事件处理

这几个函数可以由调用者重写，用于相关的业务逻辑。

``` cpp
inline void ThreadSafeAsyncWorker::OnProgress(void* data) {
  if (!_progressCallback.IsEmpty()) {
    _progressCallback.Call({});
  }
}

inline void ThreadSafeAsyncWorker::OnOK() {
  if (!_callback.IsEmpty()) {
    _callback.Call({});
  }
}

inline void ThreadSafeAsyncWorker::OnError(const Napi::Error& e) {
  if (!_callback.IsEmpty()) {
    _callback.Call(std::initializer_list<napi_value>{ e.Value() });
  }
}
```

## 子线程中可用的函数

SetError 用来指定错误信息，如果错误信息不为空字符串，子线程结束后会触发 OnError。

EmitProgress 用来触发 CallJS，CallJS 里会调用 OnProgress，同上使用时需要注意 data 指针内存泄漏。

``` cpp
inline void ThreadSafeAsyncWorker::SetError(const std::string& error) {
  _error = error;
}

inline void ThreadSafeAsyncWorker::EmitProgress(void* data) {
  if (_tsfn == nullptr) {
    return;
  }
  napi_status status = napi_call_threadsafe_function(_tsfn, data, napi_tsfn_blocking);
  if (status != napi_ok) {
    SetError("napi_call_threadsafe_function() failed.");
  }
}
```

## 禁止实例自杀

new 了指针以后默认不需要调用者 delete，可以用 SuppressDestruct 关掉这个特性。

``` cpp
inline void ThreadSafeAsyncWorker::SuppressDestruct() {
  _suppress_destruct = true;
}

inline void ThreadSafeAsyncWorker::Destroy() {
  delete this;
}
```

# 用法

1. 写一个类继承 `ThreadSafeAsyncWorker`，实现 Execute，重写 OnProgress，OnOK，OnError

    ``` cpp
    // MyAsyncWorker.h
    #include "ThreadSafeAsyncWorker.h"

    using namespace Napi;

    class MyAsyncWorker : public ThreadSafeAsyncWorker {
      public:
        MyAsyncWorker(const std::string&, Napi::Function&, Napi::Function&);
        ~MyAsyncWorker();
        void Execute();
        void OnProgress(void* data);
        void OnOK();
        void OnError(const Napi::Error&);
      private:
        // 保存一些数据
        size_t _length;
    };

    MyAsyncWorker::MyAsyncWorker(
      size_t l,
      Napi::Function& callback,
      Napi::Function& progressCallback): ThreadSafeAsyncWorker(callback, progressCallback), _length(l) {/* 构造 */}

    MyAsyncWorker::~MyAsyncWorker() {/* 析构 */}

    void MyAsyncWorker::Execute() {
      // 异步耗时操作，这里是子线程
      // ...

      for (size_t i = 0; i < _length; i++) {
        // ...
        double* progress = new double(number);
        EmitProgress((void*)progress);
        // ...
      }
      
      // 有错
      // SetError("有错");

      // ...
    }

    void MyAsyncWorker::OnProgress(void* data) {
      // EmitProgress() 后会来这里（不是同步的）
      Napi::Env env = Env();
      HandleScope scope(env);
      double* value = (double*)data;
      Object res = Object::New(env);
      res["percentage"] = Number::New(env, value);
      // 相当于 JS 的 progressCallback({ percentage: value })
      ProgressCallback().Call({ res });
      delete value; // 用完了记得释放
    }

    void MyAsyncWorker::OnOK() {
      HandleScope scope(Env());
      // 相当于 JS 的 callback(null)
      Callback().Call({ Env().Null() });
    }

    void MyAsyncWorker::OnError(const Napi::Error& err) {
      HandleScope scope(Env());
      // 相当于 JS 的 callback(err)
      Callback().Call({ err.Value() });
    }
    ```

2. new 出实例来调用 Queue

    ``` cpp
    #include "MyAsyncWorker.h"

    // 相当于
    // function threadSafeAsyncCall (length, onComplete, onProgress) {
    //   
    // }

    static Value threadSafeAsyncCall(const CallbackInfo& info) {
      // ...
      MyAsyncWorker *w = new MyAsyncWorker(info[0].As<Number>().Uint32Value(), info[1].As<Function>(), info[2].As<Function>());
      w->Queue();
      // ...
    }

    static Object _index(Env env, Object exports) {
      // module.exports = threadSafeAsyncCall
      return Function::New(env, threadSafeAsyncCall, "threadSafeAsyncCall");
    }

    NODE_API_MODULE(NODE_GYP_MODULE_NAME, _index)
    ```

3. 用 JS 来调用

    ``` js
    const threadSafeAsyncCall = require('./build/Release/${node-gyp模块名}.node')
    threadSafeAsyncCall(100000, function (err) {
      // 跑完了
    }, function (percentage) {
      // 执行中
    })
    ```
