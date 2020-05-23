---
title: 常用应用及资源列表
date: 2020-05-23 11:34:21
tags: []
---

一个普通前端码农的 Windows 常用应用和资源列表。

<!-- more -->

Windows 是主力，WSL 和虚拟机 mac 打辅助，以下内容默认针对 Windows。 

# 浏览器

* [Google Chrome](https://www.google.cn/intl/zh-CN/chrome/)

    可以说是一家独大了吧。我的插件：

    * Google 翻译

    * Axure RP Extension for Chrome

    * Adobe Acrobat

* [新 Microsoft Edge](https://www.microsoft.com/zh-cn/edge)

    换了 Chromium 内核真香，Chrome 有的它都有，Chrome 没有的它也有。

# 下载器

* [迅雷](https://www.xunlei.com/)

    虽然是流氓。

* [Free Download Manager](https://www.freedownloadmanager.org/zh/)

    免费没广告。

* [aria2](https://aria2.github.io/)

    开源跨平台。

# 压缩包处理

* [Bandizip 6](http://www.bandisoft.com/bandizip/old/6/)

    6.x 没有广告。

* [7zip](https://www.7-zip.org/)

    开源。

# PDF 阅读器

* [Acrobat Reader DC](https://acrobat.adobe.com/cn/zh-Hans/acrobat/pdf-reader.html)

    Adobe 的 PDF 阅读器。

# 录屏

* [OBS Studio](https://obsproject.com/)

    直播 / 录屏专用，开源免费。

# 系统相关

* [MSDN, 我告诉你](https://msdn.itellyou.cn/)

    原版纯净镜像下载站，不提供[白嫖服务](http://www.yishimei.cn/network/319.html)。

* [VMware WorkStation Pro](https://www.vmware.com/go/downloadworkstation-cn)

    不想买电脑就用虚拟机。

    * [虚拟机 macOS 15 安装教程](https://blog.csdn.net/qq_41855420/article/details/102756313)

# 开发相关

## 版本管理

* [Git](https://git-scm.com/)

    撸码的不知道这个就离谱。除了源码也可以管理其他的文档比如说文章笔记之类的。

* [TortoiseGit](https://tortoisegit.org/download/)

    Git 小乌龟，提供图形界面操作 Git，必须先安装 Git。

## 编辑器

* [Visual Studio Code](https://code.visualstudio.com/)

    全能撸码编辑器，用来写 JS / TS / C / C++ / Markdown。配色主题使用 Monokai。

## 集成开发环境

* [Visual Studio](https://visualstudio.microsoft.com/zh-hans/)

    提供 VC++ 环境。工作负载：

    * 使用 C++ 的桌面开发

* [IntelliJ IDEA](https://www.jetbrains.com/zh-cn/idea/)

    Java 集成开发环境。

* [Android Studio](https://developer.android.google.cn/studio/)

    安卓集成开发环境。

* [微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/devtools.html)

    不得不说开发体验极差，码农何必要为难码农，但是微信开发不用这玩意儿还不行。

## 解释型语言运行时

* [Node.js](https://nodejs.org/zh-cn/)

    基于 V8 引擎的 JavaScript 运行环境。

* [JDK 8](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html)

    Java 开发套件，包含 JRE （Java 运行环境）。

* [Python 3](https://www.python.org/)

    Python 3 运行环境。

* [Deno](https://deno.land/)

    基于 V8 引擎的 JavaScript 安全运行环境，自带 TypeScript 编译器。

* [Ruby](https://www.ruby-lang.org/zh_cn/)

    Ruby 运行环境。

## C/C++ 工具链

* Windows 上的 VC++ 工具链

    cl 编译器 / link 链接器等等，安装 Visual Studio 「使用 C++ 的桌面开发」工作负载就有了。

* [GCC](https://gcc.gnu.org/)

    GNU 编译器。

* [Clang](https://clang.llvm.org/)

    苹果弄的编译器。

* [CMake](https://cmake.org/download/)

    自动生成跨平台的 C/C++ 工程文件。

* [GNU Make](https://www.gnu.org/software/make/)

    控制从源码到目标文件构建过程的工具，描述依赖关系和执行的一系列的脚本。

## 终端

* [Windows Terminal](https://docs.microsoft.com/zh-cn/windows/terminal/)

    微软新出的终端。

## 字体

* [Consolas](https://docs.microsoft.com/zh-cn/typography/font-list/consolas)

    适合代码和终端显示的等宽字体。

* [Cascadia Code](https://docs.microsoft.com/zh-cn/windows/terminal/cascadia-code)

    微软随 Windows Terminal 一起发布的开源等宽字体，支持 Powerline 字形。

## 包管理器

* [Chocolatey](https://www.chocolatey.org/)

    Windows 第三方包管理器。

## 网络相关

* [Postman](https://www.postman.com/)

    发请求调试接口，跨平台桌面应用。

* [Fiddler](https://www.telerik.com/fiddler)

    抓包工具。

* [Proxifier](https://www.proxifier.com)

    配置全局代理。

* [curl](https://curl.haxx.se/)

    发请求、数据传输。

## 数据库相关

* [DB Browser for SQLite](http://www.sqlitebrowser.org/)

    Qt 写的 SQLite 数据库浏览器。

## 远程

* [TeamViewer](https://www.teamviewer.cn/cn/)

    远程控制设备。

## PSD 浏览

* [PxCook](https://www.fancynode.com.cn/pxcook)

    看 PSD 设计稿。

# 其它命令行工具

* [you-get](https://github.com/soimort/you-get)

    基于 Python 3 的网络视频爬取器，可用来下载 B 站视频。

* nvm

    Node.js 版本管理器，自由切换 Node.js 版本。

    * POSIX bash 版本：[nvm](https://github.com/nvm-sh/nvm)

    * Go 语言写的 Windows 版本：[nvm-windows](https://github.com/coreybutler/nvm-windows)

    * 我写的跨平台版本：[nodev](https://github.com/toyobayashi/nodev)

* [cross-zip](https://github.com/toyobayashi/cross-zip)

    基于 Node.js 的 zip 命令行工具。

# Visual Studio Code 相关

## 扩展列表

| 扩展名                     | 简要描述                                        |
|:--------------------------| ----------------------------------------------|
| Chinese (Simplified) Language Pack for Visual Studio Code | 简体中文汉化包。 |
| ESLint                    | 检查 JS / TS 代码风格。                          |
| Debugger for Chrome       | 在 VSCode 中打断点，直接调用 Chrome 调试 JS 代码。  |
| C/C++                     | 微软官方 C/C++ 扩展。                            |
| CMake                     | CMake 脚本语法高亮。                             |
| CMake Tools               | CMake 扩展。                                   |
| Bracket Pair Colorizer    | 成对括号相同颜色高亮扩展。                         |
| file-size                 | 状态栏显示当前打开的文件大小。                      |
| Git History               | 查看 Git 仓库历史。                              |
| GitLens                   | Git 集成。                                     |
| hexdump for VSCode        | 查看文件二进制数据。                              |
| Inno Setup                | Inno Setup 脚本语法高亮。                        |
| language-stylus           | Stylus 语法高亮。                               |
| Live Server               | 静态页面开发神器，保存自动刷新。                    |
| Manta's Stylus Supremacy  | Stylus 代码风格格式化。                          |
| Remote - SSH              | 服务器远程开发。                                 |
| Remote - WSL              | WSL 远程开发。                                  |
| Vetur                     | Vue 扩展。                                     |
| vscode-icons              | VSCode 图标主题。                               |

## 用户配置文件

配置文件路径：`C:\Users\USERNAME\AppData\Roaming\Code\User\settings.json`

``` json
{
  "terminal.integrated.shell.windows": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
  "workbench.iconTheme": "vscode-icons",
  "editor.tabSize": 2,
  "editor.fontFamily": "'Fira Code', Consolas, 'Courier New', 'Microsoft YaHei', monospace",
  "window.zoomLevel": 0,
  "git.autofetch": true,
  "diffEditor.ignoreTrimWhitespace": false,
  "cmake.configureOnOpen": false,
  "files.eol": "\r\n",
  "C_Cpp.clang_format_fallbackStyle": "Chromium",
  "workbench.colorTheme": "Monokai",
  "git.ignoreLegacyWarning": true,
  "color-highlight.markerType": "underline",
  "color-highlight.languages": [
    "*"
  ],
  "editor.fontLigatures": true,
  "terminal.integrated.fontFamily": "'Cascadia Mono PL', 'Microsoft YaHei'",
  "terminal.integrated.fontSize": 16,
  "[jsonc]": {
    "editor.defaultFormatter": "vscode.json-language-features"
  }
}
```
