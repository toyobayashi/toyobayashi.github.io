---
title: __esModule 的作用
date: 2020-06-29 10:07:19
tags: [学习笔记]
---

简要说明 Webpack 和 TypeScript 编译器对 `__esModule` 的处理方式 

<!-- more -->

# JS 模块化的历史包袱

最开始的 JS 没有模块化这么一说，在同一个页面中多个 JS 脚本跑在同一个全局上下文中，从上到下顺序跑，污染全局变量，依赖顺序混乱，难维护等等问题就暴露出来了。

随着 JS 不断发展和 Node.js 的出现，JS 慢慢有了模块化方案。在 ES6 之前，最有名的就是 CommonJS / AMD，AMD 就不提了现在基本不用。CommonJS 被 Node.js 采用至今，与 ES 模块共存。由于 Node.js 早期模块化方案选择了 CommonJS，导致现在 NPM 上仍然存在大量的 CommonJS 模块，JS 圈子一时半会儿是丢不掉 CommonJS 了。

与此同时，前端工程化的发展也是突飞猛进，先有 Grunt 后有 Gulp，继 Webpack 出世后又来了个“零配置”Parcel，还有 Vue 老爹尤雨溪的 Vite，类似的东西层出不穷，但是现阶段 Webpack 几乎已经是最被接受和认可的一个打包器了，短时间内不会突然被其他类似的工具取代。

Webpack 也同样实现了一套 CommonJS 模块化方案，支持打包 CommonJS 模块，同时也支持打包 ES 模块。但是两种模块格式混用的时候问题就来了，ES 模块和 CommonJS 模块并不完全兼容，CommonJS 的 `module.exports` 在 ES 模块中没有对应的表达方式，和默认导出 `export default` 是不一样的。

# CJS 与 ESM 混用的问题

考虑下面的场景：

``` js
// ESM mod.js
function foo () {}
export function bar () {}
export default foo
```

ES 模块使用侧：

``` js
// ESM index.js
import defaultExport, { bar } from './mod.js'
```

CommonJS 模块使用侧：

``` js
// CJS index.js
const { default: defaultExport, bar } = require('./mod.js')
```

ES 的默认导出可以对应 CommonJS 模块导出对象的 default 属性，但是反过来就麻烦了。

``` js
// CJS mod.js
function foo () {}
function bar () {}
module.exports = foo
module.exports.bar = bar // foo.bar === bar
```

CommonJS 模块使用侧：

``` js
// CJS index.js
const foo = require('./mod.js')
const bar = foo.bar
// 或 const { bar } = require('./mod.js')
```

ES 模块使用侧：

``` js
// ESM index.js
import { bar } from './mod.js'
import foo from './mod.js'

console.log(bar)
console.log(foo)
console.log(foo())
```

可以发现 CommonJS 的 `module.exports` 没法对应 ES 模块。

# __esModule 标识

然后为了解决这个问题，不知道是 JS 圈子里的谁最先提出了 `__esModule` 这个解决方案，现在市面上的打包器都非常默契地遵守了这个约定。

表面上看就是把一个导出对象标识为一个 ES 模块：

``` js
exports.__esModule = true
```

或

``` js
Object.defineProperty(exports, '__esModule', { value: true })
```

## Webpack 的处理方法

上面 ES 模块中导入 CommonJS 模块的例子，在 Webpack 4.43.0 打包后变成了这样（去掉所有注释）：

``` js
(function(modules) {
  // ...
  function __webpack_require__ (moduleId) {
    // ...
  }

  // ...

  __webpack_require__.d = function(exports, name, getter) {
    if(!__webpack_require__.o(exports, name)) {
      Object.defineProperty(exports, name, { enumerable: true, get: getter });
    }
  };

  __webpack_require__.r = function(exports) {
    if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
    }
    Object.defineProperty(exports, '__esModule', { value: true }); // <-- 重点
  };

  __webpack_require__.n = function(module) {
    var getter = module && module.__esModule ?
      function getDefault() { return module['default']; } :
      function getModuleExports() { return module; }; // <-- 兼容处理
    __webpack_require__.d(getter, 'a', getter);
    return getter;
  };

  return __webpack_require__(__webpack_require__.s = 0);
})({
  "./mod.js": function (module, exports) {
    function foo () {}
    function bar () {}
    module.exports = foo
    module.exports.bar = bar
  },
  "./index.js": function (module, __webpack_exports__, __webpack_require__) {
    "use strict";
    __webpack_require__.r(__webpack_exports__); // <-- 标识 ES 模块
    var _mod_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("./mod.js");
    var _mod_js__WEBPACK_IMPORTED_MODULE_0___default = __webpack_require__.n(_mod_js__WEBPACK_IMPORTED_MODULE_0__);

    console.log(_mod_js__WEBPACK_IMPORTED_MODULE_0__["bar"])
    console.log(_mod_js__WEBPACK_IMPORTED_MODULE_0___default.a)
    console.log(_mod_js__WEBPACK_IMPORTED_MODULE_0___default()())
  },
  0: function (module, exports, __webpack_require__) {
    module.exports = __webpack_require__("./index.js");
  }
  // ...
})
```

可以看到在使用侧导入的默认导出实际上是一个 Getter 函数，读取值的时候访问了其自身的 `a` 属性，如果 __esModule 为 `true` 那么 `a` 就是 `module.exports.default`，Getter 调用也返回 `module.exports.default`，否则 `a` 的值和 Getter 返回值就是 `module.exports`。所以在 Webpack 中这样用是没有问题的，Webpack 会根据 __esModule 标识来自动处理 CommonJS 的模块导出对象，兼容 ES 模块中的导入。

## TypeScript 的处理方法

同样的例子，在 TypeScript 3.9.5 中：

``` js
// CJS mod.js
function foo () {}
function bar () {}
module.exports = foo
module.exports.bar = bar
```

``` ts
// mod.d.ts
declare function foo(): void;

declare namespace foo {
  export function bar(): void;
}

export = foo;
```

``` ts
// ESM index.ts
import { bar } from './mod'
import foo from './mod' // <-- 必须配置 esModuleInterop: true

console.log(bar)
console.log(foo)
console.log(foo())
```

``` jsonc
// tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2019",
    "esModuleInterop": true
  }
}
```

输出 `index.js` 是这样的：

``` js
"use strict";
var __importDefault = (this && this.__importDefault) || function (mod) {
    return (mod && mod.__esModule) ? mod : { "default": mod };
};
Object.defineProperty(exports, "__esModule", { value: true }); // <-- 标识当前模块是 ES 模块

const mod_1 = require("./mod");
const mod_2 = __importDefault(require("./mod"));
console.log(mod_1.bar);
console.log(mod_2.default);
console.log(mod_2.default());
```

开启 `esModuleInterop` 后，如果被导入的模块没有标识 `__esModule`，则默认导入将直接返回一个只含有 `default` 属性的对象。如果不开启 `esModuleInterop` 编译选项，则不能使用默认导入，必须用 `import * as mod from './mod'` 才能通过编译。

# 总结

`__esModule` 是用来兼容 ES 模块导入 CommonJS 模块默认导出方案。个人推荐向标准看齐，在以后写 CommonJS 模块的时候尽量不要用 module.exports 导出单对象，而是导出具体的属性名 `exports.foo = bar`。在 ES 模块中也尽量不要用 `export default`。
