---
title: node-gyp 与 Visual Studio 2019 的坑
date: 2019-04-04 21:56:32
tags: ['学习笔记']
---

Windows下 `node-gyp` 使用 VS 2019 的 VC 工具集编译踩坑记录。

<!-- more -->

# 现状

写这篇笔记时 `node-gyp` 版本还是八个月前最新发布的 `3.8.0`，官方不支持 VS 2019。

# 问题

1. 找不到 `MSbuild`

    原因是VS2019中微软把 `MSbuild` 放在了别的地方导致 `node-gyp` 找不到它：

    > The MSBuild toolset version has been changed from 15.0 to Current. MSBuild.exe is now in %VSINSTALLDIR%\MSBuild\Current\Bin\MSBuild.exe.
    > MSBuild (and Visual Studio) now targets .NET Framework 4.7.2. If you wish to use new MSBuild API features, your assembly must also upgrade, but existing code will continue to work.

2. 找不到 VC++ v141 工具集

    `node-gyp` 3.8.0 默认找 VS 2017 的编译工具。

# 强行解决

`node-gyp` 一般会装在三个地方：

* npm 内置
* 全局 `node_modules` 内
* 项目 `node_modules` 内

根据需要，修改其中某个或某几个 `node-gyp` 中的代码。

`node-gyp/lib/configure.js` 152 行左右：

``` js
if (vsSetup) {
  gyp.opts.msvs_version = '2015'
  process.env['GYP_MSVS_VERSION'] = 2015
  process.env['GYP_MSVS_OVERRIDE_PATH'] = vsSetup.path
  // defaults['msbuild_toolset'] = 'v141'
  defaults['msbuild_toolset'] = 'v142'
  defaults['msvs_windows_target_platform_version'] = vsSetup.sdk
  /* variables['msbuild_path'] = path.join(vsSetup.path, 'MSBuild', '15.0',
                                           'Bin', 'MSBuild.exe') */
  variables['msbuild_path'] = path.join(vsSetup.path, 'MSBuild', 'Current',
                                           'Bin', 'MSBuild.exe')
}
```

把默认的 v141 工具集改成 v142 的，还有把 `MSbuild` 的寻找路径改一下。

# 未解决

这么一改自己写的原生模块和别的一些原生模块都能编译过了，唯独 `sqlite3` 编译不过，报错找不到 VS 2015 的 v140 工具集，没有找到原因在哪里。

`sqlite3` 依赖 `nan` 和 `node-pre-gyp`，这三个地方我都没有找到哪里默认了要找 VS 2015 的编译工具，只能再装一份 v140 工具集，编译成功。
