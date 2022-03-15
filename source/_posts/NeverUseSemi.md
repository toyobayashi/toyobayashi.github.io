---
title: 永远不要使用分号
date: 2022-03-15 12:38:21
tags: [技术分享]
---

[JavaScript Standard Style][standard] 作者 [Feross](https://feross.org/about/) 的文章[《Never Use Semicolons》](https://feross.org/never-use-semicolons/)中文翻译。

<!-- more -->

# 正文

这篇文章并不是要重新开启古老的 JavaScript 辩论，而只是在人们问为什么 [JavaScript Standard Style][standard] 强制执行「永远不要使用分号」规则时提供参考。

**「总是使用分号」时可以不用担心*自动分号补全（Automatic Semicolon Insertion，简称 ASI）*，这种想法是完全错误的。**

所有 JavaScript 开发者都必须了解 ASI，即使是那些「总是使用分号」的人。举个例子：

```js
function foo () {
  return
    {
      bar: 1,
      baz: 2
    };
}
```

哇哦，虽然你记得在最后添加分号，但这*无关紧要*。ASI 会强行介入并把你的代码修改为：

```js
function foo () {
  return; // <-- ASI 在这里插入了一个分号。现在你的代码出现了一个 bug！
    {
      bar: 1,
      baz: 2
    };
}
```

因此，告诉人们如果他们只是「总是使用分号」，他们的代码就不会出现令人惊讶的 ASI 行为，这是一种误导。

ASI 将永远与我们同在。是时候了解它的[工作原理](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Lexical_grammar#%E8%87%AA%E5%8A%A8%E5%88%86%E5%8F%B7%E8%A1%A5%E5%85%A8)了（译者注：原文链接已无法访问，这里替换为 MDN 的链接）。请放心：ASI 完全符合 ECMAScript 语言标准，所有浏览器都以完全相同的方式实现它。

至少请考虑使用检查意外 ASI 行为的 linter。ESLint 有一个称为 [no-unexpected-multiline](https://eslint.cn/docs/rules/no-unexpected-multiline) 的规则，它可以捕获意外的 ASI 行为。一旦你使用了 linter，你是使用还是省略分号都没有关系，因为 linter 可以保证你的安全。

## 「永远不使用分号」的论据

其实「总是使用分号」并不是那么简单。实际上，在许多极端情况下，你仍然不应该使用分号！例如：

```js
function foo () {
  return 42; // 正确
};           // <-- 要避免！
```

```js
var foo = function () {
}; // 正确
```

再看看这些情况：

```js
class Foo {
  constructor () {
    if (baz) {
      return 42; // 正确
    };           // <-- 要避免！
    return 12;   // 正确
  };             // <-- 要避免！
};               // <-- 要避免！
```

实际上，「总是使用分号」比「永远不使用分号」要记住更多的「边缘情况」。

如果你「永远不使用分号」，那么只需要记住一条规则：**请勿以 `[`、`(` 或 <code>`</code> 开始新的一行**。

在这些情况下，你只需在前面加上 `;`：

```js
;[1, 2, 3].forEach(bar)
```

但是，如果你经常编写这样的代码，可能会显得有些花里胡哨。这样写实际上要简洁明了得多：

```js
const nums = [1, 2, 3]
nums.forEach(bar)
```

如果你使用类似 [`standard`][standard] 的 linter，那么你不需要记忆任何东西，因为意外的 ASI 会导致报错。

*完整列表还包括一些其他字符，在真实世界中，这些字符实际上不会出现在代码的表达式开头：`+`、`*`、`/`、`-`、`,`、`.`。*

[standard]: https://standardjs.com/readme-zhcn.html
