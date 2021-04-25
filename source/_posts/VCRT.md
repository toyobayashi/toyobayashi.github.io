---
title: VC 运行时
date: 2021-03-11 17:02:16
tags: ['学习笔记']
---

动态链接 VC 运行时 (/MD /MDd) 需要的 DLL

<!-- more -->

# Visual C++ Redistributable

VS 自带：

```
<VSINSTALLDIR>\VC\Redist\MSVC\<VCToolsVersion>\<Platform>\

VSINSTALLDIR = C:\Program Files (x86)\Microsoft Visual Studio\2019\Community
VCToolsVersion = 14.28.29910
Platform = x64

C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Redist\MSVC\14.28.29910\x64\
```

VC 各发行版安装地址：[最新支持的 Visual C++ 下载](https://support.microsoft.com/zh-cn/topic/2647da03-1eea-4433-9aff-95f26a218cc0)

参考：[确定要重新分发的 DLL](https://docs.microsoft.com/zh-cn/cpp/windows/determining-which-dlls-to-redistribute?view=msvc-160)

# VS 2015 以上的 UCRT

Win 10 自带：

```
<UniversalCRTSdkDir>\Redist\<UCRTVersion>\ucrt\DLLs\<Platform>\

UniversalCRTSdkDir = C:\Program Files (x86)\Windows Kits\10
UCRTVersion = 10.0.19041.0
Platform = x64

C:\Program Files (x86)\Windows Kits\10\Redist\10.0.19041.0\ucrt\DLLs\x64\
```

VC++ DLL 依赖通用 CRT，在非 Win 10 需要真实的 DLL 存在。

参考：[通用 CRT 部署](https://docs.microsoft.com/zh-cn/cpp/windows/universal-crt-deployment?view=msvc-160)

# 官方文档

* [部署本机桌面应用程序 (Visual C++)](https://docs.microsoft.com/zh-cn/cpp/windows/deploying-native-desktop-applications-visual-cpp?view=msvc-160)
