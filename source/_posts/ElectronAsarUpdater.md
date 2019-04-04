---
title: Electron应用自动更新实现思路
date: 2018-11-07 19:20:44
tags: ['学习笔记']
---

自动更新踩坑记录。

<!-- more -->

# 为什么不用官方自带的 autoUpdater  

Windows下强制给你自动安装在C盘，光凭这一点就足够不想用了。

# Electron 应用中的 Node.js 项目  

在 Windows 和 Linux 中是 `electron/resources/app` 。  
在 macOS 中是 `electron/Electron.app/Contents/Resources/app` 。

只要这个目录中有 `package.json`，Electron 就会跑这个目录中的 JS 代码，`package.json` 中的 `main` 就是 Electron 的主进程入口。

另外 Electron 官方提供了一种 `.asar` 格式的文件，可以用来把整个 Node.js 项目目录打包成一个 asar 文件，Electron 能够访问到 asar 包中的文件。所以只要存在  

* `electron/resources/app.asar`  
* `electron/Electron.app/Contents/Resources/app.asar`  

Electron 也能按照同样的原理正常启动。

# 更新目标

很简单，就是替换整个 Node.js 项目目录的代码。这里分几种情况：

* 使用 asar 包
    ①存在 C++ 原生模块，即存在 `app.asar.unpacked` 目录，该目录下存在 `.node` 文件
    ②不存在 C++ 原生模块，即只存在 `app.asar` 文件
* 不使用 asar 包
    ③存在 C++ 原生模块，即 `app/node_modules` 中存在 `.node` 文件
    ④不存在 C++ 原生模块，即 `app/node_modules` 中不存在 `.node` 文件

使用 asar 包情况，在 Windows 系统里 Electron 应用启动后 node 文件和 app.asar 都会被占用，不能直接修改和删除，必须结束 Electron 应用的进程后才可以替换这些文件。

不使用 asar 包的情况，如果没有 C++ 原生模块可以直接替换 app 目录里的文件，如果存在同理不能直接替换。

# 解决方案

解决方案有两种：

1. 衍生独立存在的子进程，然后结束 Electron 进程，由衍生的子进程做更新替换文件的动作，完了再启动 Electron 应用
2. 利用 Electron 自身执行 Node.js 代码进行更新操作，这需要一点小技巧

具体说使用 asar 包的情况下用第 2 种解决方案。

查 Electron 的源码可以看到

``` js electron/electron/lib/browser/init.js 102行开始
// Now we try to load app's package.json.
let packagePath = null
let packageJson = null
const searchPaths = ['app', 'app.asar', 'default_app.asar']
for (packagePath of searchPaths) {
  try {
    packagePath = path.join(process.resourcesPath, packagePath)
    packageJson = require(path.join(packagePath, 'package.json'))
    break
  } catch (error) {
    continue
  }
}

if (packageJson == null) {
  process.nextTick(function () {
    return process.exit(1)
  })
  throw new Error('Unable to find a valid app')
}
```

选择 Node.js 项目目录的优先级顺序是 `app` ＞ `app.asar` ＞ `default_app.asar` 。也就是说 resources 目录中同时存在 `app` 和 `app.asar` 的情况下 Electron 会选择跑 `app` 目录下的代码而不是 asar 包中的代码，利用这一点，就可以进行自动更新了。

具体步骤：

1. 准备好 `resouces/updater/index.js` 和 `resouces/updater/package.json`
1. 从某个地方下载包含新版 asar 包和 C++ 原生模块的 zip 包
2. 解压到临时目录下比如 `resouces/.patch/app.asar` 和 `resouces/.patch/app.asar.unpacked`
3. 重命名 `resouces/updater` 为 `resouces/app`
4. 重启 Electron，执行 `resouces/app` 代码，替换文件后，删除临时目录，重命名 `resouces/app` 为 `resouces/updater`，重启 Electron。

# npm

* [electron-github-asar-updater](https://www.npmjs.com/package/electron-github-asar-updater)
