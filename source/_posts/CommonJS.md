---
title: 从零开始实现 CommonJS
date: 2020-01-02 16:36:51
tags: ['JS']
---

搞清 `require` 的原理，自己动手在浏览器中实现 `require`。

<!-- more -->

开始实现之前，先来看一下 Node.js 中的 CommonJS。

# Node.js 中的 CommonJS 规范

在 Node.js 中，一个文件就是一个模块。

## 模块的导出与导入

使用 `exports` 导出。

``` js
// a.js

exports.add = function add (a, b) {
  return a + b
}
```

使用 `module.exports` 导出。

``` js
// b.js

module.exports = function mult (a, b) {
  return a * b
}
```

区别：

模块初始化时 `exports` 和 `module.exports` 引用同一个对象，相当于是 `let exports = module.exports = {}`，最后导出的是 `module.exports`。

如果把 `exports` 的引用改掉，那就有问题了。

``` js
// c.js

// let exports = module.exports = {}
const c = { key: 'value' }
exports = c
exports.neverExport = 0
// 导出 module.exports，即 {}，不是 a
// 由于 exports 与 module.exports 不再引用同一个对象
// neverExport 也不会被导出
```

把 `module.exports` 的引用改掉，`exports` 也会没用。

``` js
// d.js

// let exports = module.exports = {}
module.exports = { key: 'value' }
exports.neverExport = 0
// 导出 module.exports，即 { key: 'value' }
// 由于 exports 与 module.exports 不再引用同一个对象
// neverExport 不会被导出
```

导入模块。

``` js
// index.js
const add = require('./a.js').add
add(1, 2) // => 3

const mult = require('./b.js')
mult(3, 4) // => 12

const c = require('./c.js') // => {}
c.key // => undefined
c.neverExport // => undefined

const d = require('./d.js') // => { key: 'value' }
d.key // => 'value'
d.neverExport // => undefined
```

## module exports require 的秘密

`module`，`exports`，`require` 这三个东西为什么在文件最开头就可以使用？它们是从哪来的？

全局变量上？试试输出 `global.module`，你会发现是 `undefined`

实际上每个 JS 文件里的代码都会被包在一个函数里面：

``` js
function (exports, require, module, __filename, __dirname) {
  // JS 文件里的代码在这里
}
```

Node.js 在加载一个 JS 文件时，会先读取文件内容，再把内容用这个函数模板包一层，然后由内部的实现来调用这个函数，传入这些参数，所以我们才能够使用 `module`，`exports`，`require`。

在开头输出一下这三个对象：

``` js
// index.js
console.log(exports)
console.log(require)
console.log(module)
```

输出为：

```
{}
[Function require]
Module {
  id: '.',
  loaded: false,
  filename: '/path/to/index.js',
  exports: {},
  parent: undefined,
  children: [],
  paths: [
    '/path/to/node_modules',
    '/path/node_modules',
    '/node_modules',
    ...
  ]
}
```

# 实现

可以看到 Node.js 中实现了一个 `Module` 类，里面存了一些模块的信息，`module` 就是 `Module` 类的实例。这里我们就不搞那么多花里胡哨的东西了，来一个最简单的实现。

``` js
(function (window) {
  // 把注册的模块存起来，键是模块 ID，值是包裹层函数
  var registeredModules = {};
  // 把加载过的模块存起来，键是模块 ID，值是 module 对象
  var installedModules = {};

  function require (moduleId) {
    // 先看有没有加载过，加载过就直接拿导出的对象，防止模块多次加载
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }

    // 先看有没有注册过这个模块，没有注册就报错
    if (!registeredModules[moduleId]) throw new Error('Cannot find module "' + moduleId + '".');

    // 构造一个 module 对象，并预先存起来，防止模块加载死循环
    var module = installedModules[moduleId] = {
      id: moduleId,
      loaded: false,
      exports: {}
    };

    // 调用模块包裹层函数，传入 module exports require
    registeredModules[moduleId].call(module.exports, module.exports, require, module);

    // 改成已加载状态
    module.loaded = true;

    // 返回 module.exports 对象
    return module.exports;
  }

  // 注册一个模块
  function register (moduleId, fn) {
    // 第二个参数必须是函数
    if (typeof fn !== 'function') throw new TypeError('Module body must be a function.');

    // 如果已经存在这个模块，就不要动了
    if (registeredModules[moduleId]) {
      return;
    }
    registeredModules[moduleId] = fn;
  }

  // 运行入口模块
  function runAsMain (moduleId) {
    require(moduleId);
  }

  window.cjs = {
    register: register,
    runAsMain: runAsMain
  };
})(window)
```

引用这段 JS 后，就可以和 Node 里一样用了。注册模块可以不考虑顺序，但是 runAsMain 必须在所有模块都注册了以后调用，还有 require 的不是文件路径。

``` js
cjs.register('entry', function (exports, require, module) {
  var add = require('a').add;
  console.log(add(1, 2)); // 3

  var mult = require('b');
  console.log(mult(3, 4)); // 12

  var c = require('c');
  console.log(c); // {}
  console.log(c.key); // undefined
  console.log(c.neverExport); // undefined

  var d = require('d');
  console.log(d); // { key: 'value' }
  console.log(d.key); // 'value'
  console.log(d.neverExport); // undefined
});

cjs.register('a', function (exports, require, module) {
  exports.add = function add (a, b) {
    return a + b;
  };
});

cjs.register('b', function (exports, require, module) {
  module.exports = function mult (a, b) {
    return a * b;
  };
});

cjs.register('c', function (exports, require, module) {
  var c = { key: 'value' };
  exports = c;
  exports.neverExport = 0;
});

cjs.register('d', function (exports, require, module) {
  module.exports = { key: 'value' }
  exports.neverExport = 0;
});

cjs.runAsMain('entry');
```

更完善的实现： [bommon-ts](https://github.com/toyobayashi/bommon-ts)
