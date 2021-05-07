---
title: 国服 Guardian Tales BGM 提取
date: 2021-05-07 10:02:33
tags: ['游戏']
---

国服《坎特伯雷公主与骑士唤醒冠军之剑的奇幻冒险》BGM 提取步骤。

<!-- more -->

# 准备

* 已完整安装游戏客户端的安卓手机以及数据线 / 已完整安装游戏客户端的安卓模拟器
* U3D Asset bundle 拆包工具，以下可选
    * [AssetStudio](https://github.com/Perfare/AssetStudio/releases) （推荐）
    * [UABE](https://github.com/DerPopo/UABE/releases)（本次使用 2.2 stable d）+ [PowerToys](https://github.com/microsoft/PowerToys/releases) 启用 PowerRename 功能用于正则重命名

# 开始

下面提到的所有路径不要包含中文字符，以免出现奇奇怪怪的问题。

1. 把 `/storage/emulated/0/Android/data/com.bilibili.snake/files/AssetBundles/Android/audio/bgms` 这个目录拷出来，比如拷到 `C:\GTBGM`

2. 文件资源管理器进入 `C:\GTBGM`，按类型排序后，把全部以 `.sha1` 结尾的文件删掉，留下不带后缀的文件

3. 运行 `AssetStudioGUI.exe`，File → Load folder → 选择 `C:\GTBGM` → Export → All assets → 选择导出目录 → 完成

以下是使用 UABE + PowerToys 的步骤，前两步和上面一样。

3. 在 `C:\GTBGM` **外**的地方，新建文件夹，比如新建 `C:\GTBGMU3D`

4. 在 `C:\GTBGMU3D` 新建文本文档 `batch.txt`，内容如下，以 UTF-8 编码保存

    ```txt
    +DIR C:\GTBGM
    ```

5. 在 `C:\GTBGMU3D` 新建文本文档 `uabe.txt`，内容如下，把其中的 `<path>` 替换成你本地 UABE 的安装目录，保存后重命名为 `uabe.bat` 文件。

    ```bat
    @echo off
    <path>\AssetBundleExtractor.exe -fd -keepnames batchexport batch.txt
    ```

6. 双击运行 `uabe.bat`，`C:\GTBGMU3D` 里面会出来很多 CAB 开头的文件

7. 运行 `<path>\AssetBundleExtractor.exe`，File → Open → 全选第 6 步输出的文件 → 按 Type 排序 → 全选 Type 为 AudioClip 的 Name → 右侧 Plugin 按钮 → Export sound → OK → 选择导出目录（比如 `C:\GTBGMWAV`） → 完成导出

8. 右键 `C:\GTBGMWAV` 文件夹 → PowerRename → 勾选 `使用正则表达式` → 搜索 `-\w+-\w+-\w+\.wav` 替换为 `.wav` → 点击重命名
