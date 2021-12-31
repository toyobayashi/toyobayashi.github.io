---
title: Node.js 线程
date: 2021-12-23 15:08:21
tags: [学习笔记]
---

跑 JavaScript 确实是单线程在跑的，但 Node.js 不是单线程的应用程序。

<!-- more -->

# libuv 线程池

理清楚：

1. 主线程 V8 引擎在负责解释执行 JavaScript 代码，且有一个 libuv 默认 `uv_loop_t` 处理 JavaScript 的异步事件
2. 文件操作、网络等 API 底层由 libuv 库提供，通过 V8 C++ Binding 暴露给 JavaScript
3. libuv 的网络 I/O 是单线程的，利用操作系统的原生事件系统，比如 IOCP、kqueue、epoll
4. libuv 的文件 I/O 是多线程的，libuv 开了线程池，在其他线程中调用操作系统的同步文件 API，完了再通知主线程的 loop
5. 除了文件 I/O，DNS Functions 和用户自定义的 `uv_queue_work`（即 napi 的 `AsyncWorker`）也是放在线程池的其他线程中执行的

通过环境变量 `UV_THREADPOOL_SIZE` 设置 libuv 的线程池大小，Show 出代码更直观

```js
// threadpool-size.js

const { pbkdf2 } = require('crypto')

const start = Date.now()

const doExpensiveHashing = () => {
  pbkdf2('pwd', 'salt', 100000, 512, 'sha512', () =>
    console.log(`Done in ${Date.now() - start}ms`)
  )
}

for (let i = 0; i < 10; ++i) {
  doExpensiveHashing()
}
```

Windows 上运行

```
> node threadpool-size.js
Done in 508ms
Done in 554ms
Done in 617ms
Done in 673ms
Done in 1077ms
Done in 1174ms
Done in 1249ms
Done in 1290ms
Done in 1604ms
Done in 1692ms

> set UV_THREADPOOL_SIZE=5&&node threadpool-size.js
Done in 508ms
Done in 514ms
Done in 615ms
Done in 636ms
Done in 681ms
Done in 1069ms
Done in 1074ms
Done in 1172ms
Done in 1177ms
Done in 1240ms

> set UV_THREADPOOL_SIZE=1&&node threadpool-size.js
Done in 480ms
Done in 966ms
Done in 1450ms
Done in 1927ms
Done in 2405ms
Done in 2885ms
Done in 3365ms
Done in 3847ms
Done in 4325ms
Done in 4802ms
```

看运行结果可以得知 Node.js 默认的线程池大小是 4

# V8 线程池

`worker_threads` 模块用于开多线程来跑 JavaScript，更有利于 JavaScript 处理 CPU 密集型计算，对于 I/O 密集型场景不适用，libuv 的异步 I/O 处理比 `worker_threads` 更高效。

工作线程实际上用的是 V8 线程池，即 V8 初始化的时候构造 `v8::Platform` 的子类 `node::MultiIsolatePlatform` 设置线程池数量才能开工作线程。
