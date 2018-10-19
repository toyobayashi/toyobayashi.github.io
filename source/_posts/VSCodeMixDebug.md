---
title: 使用VSCode混合调试C++与Node.js
date: 2018-10-19 19:20:44
tags: ['学习笔记']
---

总结一下VSCode的混合调试。

<!-- more -->

# 效果图

![混合调试](demo.gif)

# VSCode  

必须先吹一把VSCode，巨硬少有的良心作，用来写前端速度快，体位好，手感棒，谁用谁知道，尤其是对JS和TS的支持非常好。

其本身也是由自家TS写的Electron跨平台编辑器，没有Visual Studio的笨重，虽然它不是IDE，但是耍起来堪比IDE好使。

下面进入正题。

# Node.js C++ 原生模块  

Node.js允许我们用C++来开发原生模块。理论上来说C++能做的事JS大部分都能做，为什么还要用C++来写Node.js，我认为有以下几点：

1. 可以使用C++中JS所没有的特性，比如类的私有成员，大数运算等等
2. 可以提高性能，编码解码、加密解密等运算量大的工作C++的效率要比JS快得多
3. 可以使用现有的C++轮子
4. 纯属没事做写着玩

而我就是出于第4种原因。

其实让Node.js和C++搞起基来是非常容易的，接着往下看。

# 开发环境

具体可以参考[node-gyp文档](https://www.npmjs.com/package/node-gyp)

* Visual Studio Code 1.22+（本文主角）
* Node.js 8+ （这里我使用8版本以上才有的NAPI来写）
* node-gyp （官方的原生模块构建工具，通过npm安装）
* Python 2.7 （node-gyp需要，Python 3不行）
* .NET 4.5.1 (非Win7忽略这一项)
* Visual Studio 2015/2017，或单独安装MSBuild和VC++工具集140/141（非Windows系统忽略这一项）
* gcc （非Linux系统忽略这一项）
* Xcode （非Mac系统忽略这一项）

# 搞起
