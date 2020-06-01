---
title: GitHub Actions 参考
date: 2020-06-01 16:02:15
tags: [学习笔记]
---

GitHub Actions 参考

<!-- more -->

# 核心概念

## Workflow

工作流，根据特定的**事件**（Event）触发一整套工作流程，比如一次推送到 master 分支的事件触发包括拉代码，装依赖，构建，测试，等等一系列**任务**（Job）的流程。

## Job

任务，在一个**工作流**（Workflow）中，运行在特定**运行环境**（Runner）中的一个任务，比如在 Windows 上运行构建任务。不同任务可以并行执行，也可以指定依赖关系。

## Step

步骤，一个**任务**（Job）可以拆分成多个**步骤**（Step），一个步骤可以跑一行或多行命令，也可以进行一套**操作**（Action）

## Action

操作，为了完成具体某件事而进行的一套操作，比如创建一个 Release，上传一些文件。操作是可以自定义的，可以是 Node.js 脚本，也可以是 Docker 容器，非常灵活。

## Runner

运行环境，GitHub 官方提供了 Windows / Linux / macOS 三种运行环境，也可以创建自定义的运行环境。

## Event

事件，比如一次推送触发 push 事件，创建 Release 触发 releases create 事件。

# 示例

在仓库根目录创建 `.github/workflows/main.yml` 工作流描述文件，文件名可以随便起，一个 yml 就是一个工作流。

``` yml
# 工作流的名字
name: Build

# 指定什么事件来触发这个工作流
on: 
  push:
    branches:
    - master # 当推送 master 分支时触发

# 设置一些任务，每个任务都运行在不同的运行环境中
jobs:
  build:
    name: Build
    # needs: draft 别的任务 key，也可以是数组
    runs-on: ${{ matrix.os }} # 运行环境
    strategy:
      fail-fast: false # 矩阵中某个任务失败不要停止别的任务
      matrix: # 在三个环境中分别跑这个任务
        os: [windows-latest, ubuntu-latest, macos-latest]

    steps:
    - uses: actions/checkout@v2 # 官方出的，拉仓库代码

    - name: Windows build
      if: ${{ matrix.os == 'windows-latest' }} # 跑这个步骤的条件
      shell: cmd # Windows 还可以是 pwsh
      run: |
        mkdir .\out
        echo win32>.\out\win32.txt

    - name: Linux build
      if: ${{ matrix.os == 'ubuntu-latest' }}
      shell: bash
      run: |
        mkdir -p ./out
        echo "linux" > ./out/linux.txt

    - name: macOS build
      if: ${{ matrix.os == 'macos-latest' }}
      shell: bash
      run: |
        mkdir -p ./out
        echo "darwin" > ./out/darwin.txt
    
    - name: Create release
      # 当推送标签时跑这个步骤
      if: ${{ startsWith(github.event.ref, 'refs/tags') }}
      # 我写的 Action，创建 Release 并上传 Assets
      uses: toyobayashi/upload-release-assets@v3.0.0 
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # 自带，不需要手动配置
      with: # 给当前 action 传参，具体要看这个 action 需要哪些参数，现阶段参数只能是字符串
        tag_name: ${{ github.event.after }}
        release_name: ${{ github.event.after }}
        draft: true
        prerelease: false
        assets: |
          ./out/*.txt
          ./dist/main.js
          ./not/exists
```

# 官方中文文档

[https://help.github.com/cn/actions](https://help.github.com/cn/actions)
