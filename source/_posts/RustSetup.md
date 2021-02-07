---
title: Rust 初始化
date: 2021-02-07 23:27:03
tags: ['学习笔记', 'Rust']
---

学习 Rust。

<!-- more -->

# 加速安装

```bat
set RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
set RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/
rustup-init.exe
```

# Cargo 配置

`~/.cargo/config`

```toml
# 换源
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"

# 静态链接 VC 运行时
# https://github.com/rust-lang/rfcs/blob/master/text/1721-crt-static.md
# https://support.microsoft.com/en-us/topic/the-latest-supported-visual-c-downloads-2647da03-1eea-4433-9aff-95f26a218cc0
[target.x86_64-pc-windows-msvc]
rustflags = ["-Ctarget-feature=+crt-static"]
[target.i686-pc-windows-msvc]
rustflags = ["-Ctarget-feature=+crt-static"]
```

The Rust Programming Language

# 官方文档

[《Rust 程序设计语言》中文译本](https://kaisery.github.io/trpl-zh-cn/) 

打开离线版

```
rustup docs --book
```
