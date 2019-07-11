---
title: Electron使用Webpack后remote.require的痛点
date: 2019-07-11 10:19:23
tags: ['学习笔记']
---

记录一次 Electron 开发的坑，在使用 Webpack 打包主进程代码后如何在渲染进程中正常使用 `remote.require()`。

<!-- more -->

# 问题背景

Electron 5 出来也不短时间了，5.x 版本最明显的大改动就是出于安全性考虑渲染进程默认禁用 Node.js 集成，这意味着在渲染进程中不能像 5.x 以下的版本中那样直接使用 Node.js 的 API，没有 require，除非手动指定 `webPreferences.nodeIntegration = true`。Webpack 在最近的版本发布中也指出，5.x 的 Electron 渲染进程构建目标应该设置成 `web` 而不是 `electron-renderer`，同时还新增了一个 `electron-preload` 的构建目标（看源码其实就是 `electron-renderer` 的别名）。

按照官方的意思，简单说就是渲染进程和浏览器没什么区别了，就是个破网页，想用 Node？，你得先写个 preload 脚本，preload 的运行环境还是渲染进程，但是是在渲染进程加载之前跑的代码，里面还是可以使用所有 Node API，借助 preload 脚本把一些涉及 node 的东西暴露到全局可访问的地方，渲染进程内就照样可以正常用。这个不展开说。

现在思考这样一个场景，假如某块业务涉及的计算量很大，又要用到 Node API，就不适合把它放到渲染进程内处理，这样会导致页面卡顿假死，体验不好，所以必须把这部分业务代码挪到主进程内跑。这其中可能就会涉及到要实时在渲染进程的页面上反映处理的进度。

举个例子来说，假如有个需求要弄音频解码，渲染进程里要做一个进度条，实时反映解码的进度。解码是个计算量超大又很耗时的活，无论是用 JS 做还是 C++ 做都不应该放在渲染进程里。好，那就放在主进程里嘛，常规的做法就是用 Electron 内置的 IPC 模块，渲染进程发个消息给主进程，主进程就开始解码，解码的过程中不断给渲染进程发消息告诉渲染进程进度，渲染进程收到消息更新进度条。

这样做是可以做，但是渲染进程内的代码会被分成两部分，一部分是获取配置以及让主进程解码：

``` js
ipcRenderer.send('你给我解码', 音频信息和一些配置)
```

另一部分是监听解码进度的消息更新进度条：

``` js
ipcRenderer.on('告诉我解码进度', (e, 进度信息) => { /* 更新进度条逻辑及其它逻辑 */ })
```

搞不好这两块代码还在不同的文件里，难维护，这还只是解码一个的音频的情况，如果要同时解码多个，这解码进度的更新就需要更多的实现才能把不同音频的状态分开。有没有更好的处理方法？当然有，__传回调__！想当然的话，可能会这样写：

``` js
ipcRenderer.send('你给我解码', 音频信息和一些配置, (进度信息) => { /* 更新进度条逻辑及其它逻辑 */ })
```

看起来没什么问题，解码的过程中不断地调用渲染进程传过去的回调函数，更新进度条。这里致命的问题在于，__跨进程不能传函数__，主进程接收不到这个回调函数！__跨进程只能传递原始值__，如果传对象或数组，Electron 会帮忙拷贝一份，而不是原对象或原数组的引用，而且挂在数组上的自定义属性也会丢失。

怎么办呢？Electron 为开发者铺好了路，两个办法（必须要在 preload 里用，因为渲染进程拿不到 Electron 的 `remote` 对象）：

1. `remote.getGlobal`：主进程中把解码的函数挂在 `global` 上，通过这个 API 获取主进程 `global` 上的解码函数，在渲染进程中可以得到这个函数得“假引用”，这样可以从渲染进程传回调。
2. `remote.require`：把解码的函数单独写在一个 JS 里并导出，在渲染进程中调用主进程的 `require` 函数，得到主进程中解码模块的“假引用”，这样也可以传回调。

痛点又来了。

第一种方法超级方便，但是会污染主进程的 `global` 对象，有洁癖的人比如我就很抗拒这样写。

第二种方法，如果主进程的代码不用 Webpack 打包，也很方便，没有什么障碍。接下来要讲的，就是用 Webpack 打包主进程代码的前提下，这个方法要怎么用。

# 问题分析

为什么说用了 Webpack 这个方法就很麻烦呢？就拿 Electron 官方文档的例子来说，我稍微改一改。

```
project/
├── main
│   ├── decode.js
│   └── index.js
├── package.json
└── renderer
    └── index.js
```

``` js
// main process: main/index.js
const { app } = require('electron')
const decode = require('./decode')
app.on('ready', () => { /* 这里要用到解码 decode(input, (progress) => {}) */ })
```

``` js
// some relative module: main/decode.js
// 导出解码功能
module.exports = function decode (input, callback) {
  /* 这里做解码 */
}
```

``` js
// renderer process: renderer/index.js
const decode = require('electron').remote.require('./decode') // 相对主进程入口的路径
decode('文件路径', (progress) => {
  /* 这里更新进度条 */
})
```

不用 Webpack 打包 `main/index.js`，一点问题都没有。但是一旦打包，`main/index.js` 和 `main/decode.js` 里面的代码都会被打进 bundle，渲染进程就不能 require 到原始的 `main/decode.js`，`main/decode.js`不能独立存在，因为主进程的其它代码里要用到它，它被打进了包里。

# 问题解决

既然不能把 `main/decode.js` 分出来，那就不分，我们可以给主进程和渲染进程建立新的桥梁，就好像 `global` 对象一样。

具体的做法是，新弄一个 webpack config，这个配置单独打出一个 JS 来，作为要被 remote.require 的模块。

``` js
// webpack.config
module.exports = {
  /* ... */
  entry: {
    export: [toAbsolute('./main/export.js')]
  },
  output: {
    /* path */
    filename: '[name].js',
    libraryTarget: 'commonjs2'
  },
  /* ... */
}
```

`libraryTarget: 'commonjs2'` 的意思是这个包会被打成一个遵循 CommonJS 2 模块规范的库，可以被 require。

这个模块中实现一个超级简单的缓存。

``` js
// main process: main/export.js
let cache = {}

export function getCache (name) {
  return cache[name]
}

export function setCache (name, value) {
  cache[name] = value
}

export function clearCache () {
  cache = {}
}

export function removeCache (name) {
  if (cache.hasOwnProperty(name)) {
    delete cache[name]
  }
}
```

主进程利用 `__non_webpack_require__` 把 decode 函数塞进这个模块缓存里：

``` js
// main process: main/index.js
const { app } = require('electron')
const decode = require('./decode')

// 这个特殊的 require 是 webpack 内置的，会被转码成 Node.js 原生的 require 函数，而不会被打进包里
const { setCache } = __non_webpack_require__('./export')
// => var 某个名字 = require(export.js的模块id).setCache

app.on('ready', () => {
  /* 这里要用到解码 decode(input, (progress) => {}) */
  setCache('decode', decode)
})
```

渲染进程 preload 中通过 `remote.require` 拿到缓存，挂在 window 对象上

``` js
// renderer process: renderer/preload.js
const { remote } = require('electron')

process.once('loaded', function () {
  // 这里 process 对象已经可用
  window.preload = {
    decode: remote.require('./export.js').getCache('decode') // 相对主进程入口包的路径
  }
})
```

渲染进程加载以后禁用包括 require 在内的 Node API

``` js
// renderer process: renderer/index.js
const decode = window.preload.decode
decode('文件路径', (progress) => {
  /* 这里更新进度条 */
})
```


完。
