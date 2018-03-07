---
title: 在Windows平台下搭建Electron-Vue开发环境
date: 2018-03-07 19:35:31
tags: [JS, 学习笔记]
---

有没有想过用JavaScript撸桌面应用？

<!-- more -->

# Electron是个啥

官网说：
> Build cross platform desktop apps with JavaScript, HTML, and CSS.
> 使用 JavaScript, HTML 和 CSS 构建跨平台的桌面应用。

我说：  
Node.js + Chrome ≈ [Electron](https://electronjs.org/)  

简单来说，Electron就是个__壳__，它内嵌了Chrome浏览器的内核，自带Node.js环境，Node.js的API在Electron里同样能用，但是有轻微的差别。它加载一个HTML，这个HTML页面就是应用的界面，页面加载的JS运行在带有Node.js环境的上下文中。Electron有两个进程，分别是__主进程__和__渲染进程__，主进程是Electron执行入口JS文件的进程，类似Node.js的进程，而渲染进程是Electron内页面所加载的JS的进程，类似浏览器中运行的JS，与浏览器不同的是，渲染进程中的JS仍然可以使用Node.js的API，它的`window`对象就是`global`对象。

下面这些东西的壳就是Electron：
* [Atom](https://atom.io/)
* [Github Desktop](https://desktop.github.com/)
* [Visual Studio Code](https://code.visualstudio.com/)

只要会做网页，用Electron就能做跨平台的桌面应用，更用不着担心浏览器兼容问题，是不是很棒棒？Vue也适合用来做中小型的单页面应用，Electron配合Vue，两个字，舒服。下面就介绍一下Windows平台下的开发环境搭建。  

# 基本操作

## 目录结构
*  \- build/ （构建脚本）
  * dev.js  （启动开发模式）
  * native.js  （处理原生模块路径）
  * pack.js  （打包应用）
  * webpack.dll.config.js  （打包生产依赖库，在/public下生成dll.js）
  * webpack.main.config.js  （打包主进程代码，在/public下生成main.js）
  * webpack.renderer.config.js  （打包渲染进程代码，在/public下生成renderer.js）
* \- public/ （打包代码）
  * \+ lib/  （存放文件扩展名为.node的C++模块）
  * index.html  （界面）
* \- src/  （源代码）
  * \+ cpp/
  * \+ css/
  * \- js/
    * app.js  （被App.vue引用）
    * main.js  （主进程webpack入口）
    * renderer.js  （渲染进程webpack入口）
  * \- res/  （图片等静态资源，被webpack一起打包）
    * \+ icon/
    * \+ img/
  * \- vue/
    * App.vue  （根组件）
* .eslintrc.json  （ESLint配置，我这里是standard编码风格）
* .gitignore
* package.json
* package-lock.json

## package.json

``` javascript
{
  "name": "electron-vue-simple", // 打包时必需
  "version": "1.0.0", // 打包时必需
  "description": "Electron-Vue quick start", // 打包时必需
  "main": "./public/main.js", // Electron的入口文件，必需
  "scripts": {
    "start": "electron . --enable-logging",
    "webpack": "webpack --config ./build/webpack.renderer.config.js&&webpack --config ./build/webpack.main.config.js",
    "dll": "webpack --config ./build/webpack.dll.config.js",
    "dev": "node ./build/dev",
    "prod": "set NODE_ENV=production&&npm run dll&&npm run webpack",
    "pkg32": "set NODE_ENV=production&&npm run dll&&npm run webpack&&node ./build/pack ia32",
    "pkg64": "set NODE_ENV=production&&npm run dll&&npm run webpack&&node ./build/pack x64"
  },
  "author": "toyobayashi", // 打包时必需
  "license": "MIT",
  "devDependencies": {
    "cmd-rainbow": "^1.0.1",
    "css-loader": "^0.28.9",
    "electron": "1.8.2", // 有C++原生模块时建议把版本写死
    "eslint": "^4.16.0",
    "eslint-config-standard": "^11.0.0",
    "eslint-plugin-html": "^4.0.2",
    "eslint-plugin-import": "^2.8.0",
    "eslint-plugin-node": "^5.2.1",
    "eslint-plugin-promise": "^3.6.0",
    "eslint-plugin-standard": "^3.0.1",
    "extract-text-webpack-plugin": "^3.0.2",
    "file-loader": "^1.1.6",
    "rcedit": "^0.9.0",
    "request": "^2.83.0",
    "style-loader": "^0.19.1",
    "uglifyjs-webpack-plugin": "^1.1.6",
    "unzip": "^0.1.11",
    "url-loader": "^0.6.2",
    "vue-loader": "^13.7.0",
    "vue-template-compiler": "^2.5.13",
    "webpack": "^3.10.0",
    "webpack-dev-server": "^2.11.1"
  },
  "dependencies": {
    "vue": "^2.5.13"
  }
}
```

## 插一句嘴

由于众所周知的原因，在安装依赖之前，我们也许需要配置一下npm环境变量。
``` bash
> npm config set registry http://registry.npm.taobao.org/
> npm config set electron_mirror https://npm.taobao.org/mirrors/electron/
```

我用的编辑器是Visual Studio Code，支持ESLint检查代码高亮，不需要更多的配置，更重要的是它支持调试Electron应用。

# 常规操作

## Webpack配置

写这篇文章的时候Webpack已经更新了4版本，无奈坑太多没文档，暂时就先用着3。

### webpack.main.config.js

给Electron主进程代码打包。

``` javascript
const webpack = require('webpack')
const UglifyJSPlugin = require('uglifyjs-webpack-plugin')
const path = require('path')

let main = {
  target: 'electron-main',
  entry: path.join(__dirname, '../src/js/main.js'),
  output: {
    path: path.join(__dirname, '../public'),
    filename: 'main.js'
  },
  node: { // 禁止Webpack处理库中出现的Node.js全局变量
    __dirname: false,
    __filename: false
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': process.env.NODE_ENV === 'production' ? '"production"' : '"development"'
    })
  ]
}

if (process.env.NODE_ENV === 'production') { // 生产环境下压缩代码
  const uglifyjs = new UglifyJSPlugin({ // 压缩ES6版本以上的JS代码，由于Electron使用8版本的Node.js，所以基本不需要Babel了
    uglifyOptions: {
      ecma: 8,
      output: {
        comments: false,
        beautify: false
      },
      warnings: false
    }
  })
  main.plugins.push(uglifyjs)
}

module.exports = main
```

### webpack.dll.config.js

生产要用到的库，使用DLL插件单独搞出一个JS来给HTML引，开发时Webpack就不会重复打包了。

``` javascript
const webpack = require('webpack')
const path = require('path')
const UglifyJSPlugin = require('uglifyjs-webpack-plugin')
const { dependencies } = require('../package.json')

module.exports = {
  target: 'electron-renderer',
  entry: {
    vendor: Object.keys(dependencies)
  },
  node: { /* 与上面相同 */ },
  output: {
    path: path.join(__dirname, '../public'),
    filename: 'dll.js',
    library: 'dll' // 被HTML引入时暴露的全局变量名
  },
  plugins: [
    new UglifyJSPlugin({ /* 与上面相同 */ }),
    new webpack.DllPlugin({
      path: path.join(__dirname, 'manifest.json'), // 在当前文件夹生成依赖模块的清单文件，供DLL使用者读取
      name: 'dll' // 注册在清单文件中，告诉DLL使用者全局变量名，必须与output.library保持一致
    }),
    new webpack.DefinePlugin({ /* 与上面相同 */ })
  ]
}
```

### webpack.renderer.config.js

给Electron内部浏览器使用的JS代码（渲染进程的代码）打包，它就是DLL的使用者。

``` javascript
const webpack = require('webpack')
const ExtractTextPlugin = require('extract-text-webpack-plugin')
const UglifyJSPlugin = require('uglifyjs-webpack-plugin')
const path = require('path')
const native = require('./native.js')

let renderer = {
  target: 'electron-renderer',
  entry: path.join(__dirname, '../src/js/renderer.js'),
  output: {
    path: path.join(__dirname, '../public'),
    filename: 'renderer.js'
  },
  node: {
    __dirname: process.env.NODE_ENV !== 'production',
    __filename: process.env.NODE_ENV !== 'production'
  },
  module: {
    rules: [{
      test: /\.(eot|woff|svg|woff2|ttf|otf)$/,
      exclude: /node_modules/,
      loader: 'file-loader?name=./asset/font/[name].[ext]?[hash]'
    }, {
      test: /\.(png|jpg|gif)$/,
      exclude: /node_modules/,
      loader: 'url-loader?limit=8192&name=./img/[name].[ext]?[hash]'
    }, {
      test: /\.vue$/,
      exclude: /node_modules/,
      loader: 'vue-loader',
      options: {
        loaders: {},
        extractCSS: process.env.NODE_ENV === 'production'
        // other vue-loader options go here
      }
    }]
  },
  externals: native(['hello']),
  plugins: [
    new webpack.DllReferencePlugin({
      manifest: require('./manifest.json') // 读取DLL里面的模块清单信息，require或import的时候就不会再打进包里了
    })
  ]
}

if (process.env.NODE_ENV === 'production') {
  const uglifyjs = new UglifyJSPlugin({ /* 与上面相同 */ })
  renderer.plugins = renderer.plugins.concat([
    uglifyjs,
    new ExtractTextPlugin('./renderer.css'),
    new webpack.LoaderOptionsPlugin({
      minimize: true
    })
  ])
  renderer.module.rules.push({
    test: /\.css$/,
    exclude: /node_modules/,
    use: ExtractTextPlugin.extract({ fallback: 'style-loader', use: 'css-loader' })
  })
} else {
  renderer.module.rules.push({
    test: /\.css$/,
    exclude: /node_modules/,
    loader: 'style-loader!css-loader'
  })
  renderer.devServer = {
    contentBase: path.join(__dirname, '../public'),
    compress: true,
    port: 7777
  }
}

module.exports = renderer
```

`native.js`处理开发环境和生产环境下C++原生模块的路径问题。

``` javascript
const path = require('path')
const outputPath = path.join(__dirname, '../public')
const nativeDir = './lib'

module.exports = nativeModules => {
  let externals = {}
  for (let i = 0; i < nativeModules.length; i++) {
    externals[nativeModules[i]] = process.env.NODE_ENV === 'production'
      ? `process.arch === "ia32" ? require("${nativeDir}/${nativeModules[i]}-ia32.node") : require("${nativeDir}/${nativeModules[i]}-x64.node")`
      : `process.arch === "ia32" ? require("${path.join(outputPath, nativeDir, nativeModules[i] + '-ia32.node').replace(/\\/g, '/')}") : require("${path.join(outputPath, nativeDir, nativeModules[i] + '-x64.node').replace(/\\/g, '/')}")`
  }
  return externals
}
```

### 监视文件
同时衍生两个子进程，分别监视主进程和渲染进程的变化。以下是`dev.js`的内容。

``` javascript
const { exec } = require('child_process')
const path = require('path')

let serverProcess = exec(path.join('../node_modules/.bin/webpack-dev-server') + ' --config webpack.renderer.config.js --hot', { cwd: __dirname })
let mainProcess = exec(path.join('../node_modules/.bin/webpack') + ' --config webpack.main.config.js -w', { cwd: __dirname })

const callback = data => console.log(data.toString())
serverProcess.stdout.on('data', callback)
serverProcess.stderr.on('data', callback)
mainProcess.stdout.on('data', callback)
mainProcess.stderr.on('data', callback)
```

### 怎么玩

看看前面`package.json`的`script`。

开发：
1. `npm run dll` （运行了生产脚本的话会打包生产环境的DLL，要在开发环境下再打包一次DLL）
2. `npm run dev`

生产：
* `npm run prod`

启动：
* `npm start`

## 开始Electron的表演

### 主进程

Electron和Node.js一样，需要执行一个入口JS文件，这个入口就是主进程生命周期的开始。

下面是入口文件`main.js`的写法

``` javascript
import { app, BrowserWindow } from 'electron'

let mainWindow

function createWindow () { // 创建窗口
  mainWindow = new BrowserWindow({ // 设置窗口属性
    width: 800,
    height: 600
  })

  let winURL = process.env.NODE_ENV === 'development'
    ? 'http://localhost:7777/index.html'
    : `file:///${__dirname.replace(/\\/g, '/')}/index.html`

  mainWindow.loadURL(winURL) // 浏览器窗口加载的URL
  // 如果是开发环境，自动开启Chrome浏览器的DevTools
  if (process.env.NODE_ENV === 'development') mainWindow.webContents.openDevTools()

  mainWindow.on('closed', function () {
    mainWindow = null // 窗口关闭时清除缓存
  })
}

app.on('ready', createWindow)

app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

app.on('activate', function () {
  if (mainWindow === null) {
    createWindow()
  }
})
```

### 渲染进程

浏览器里怎么写，渲染进程JS就怎么写，可以用Node.js的API。

主进程中加载的`index.html`中只要引入DLL和打包出来的JS就行了，`title`会显示为窗口标题。

``` html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>electron-template-test</title>
    <link rel="stylesheet" type="text/css" href="./renderer.css" />
  </head>
  <body>
    <div id="app"></div>
    <script src="./dll.js"></script>
    <script src="./renderer.js"></script>
  </body>
</html>
```

`renderer.js`就可以当作是Web开发时常规的Webpack入口文件了

``` javascript
import '../css/public.css'
import Vue from 'vue'
import App from '../vue/App.vue'
import electron from 'electron'

Vue.use({
  install (Vue) {
    Vue.prototype.electron = electron // 注册全局变量，所有组件的实例都能访问到
  }
})

new Vue({
  el: '#app',
  render: h => h(App)
})
```

`App.vue`。

``` html
<template>
<div>
  <div class="text-center"><img src="../res/img/256x256.png"></div>
  <h1 class="vue text-center">electron-vue-simple</h1>
  <p class="text-center" v-text="`Electron: ${electronVer}`"></p>
  <p class="text-center" v-text="`Vue: ${vueVer}`"></p>
  <p class="text-center" v-text="`Dir: '${dirname}'`"></p>
  <p class="text-center" v-text="`File: '${filename}'`"></p>
</div>
</template>

<script src="../js/app.js"></script>

<style src="../css/app.css"></style>
```

`app.js`。
``` javascript
import Vue from 'vue'

export default {
  data () {
    return {
      electronVer: process.versions.electron,
      vueVer: Vue.version,
      dirname: __dirname,
      filename: __filename
    }
  }
}
```

# 骚操作

## 使用C++原生模块

由于Electron自带的Node.js是打了补丁的，和原本的Node.js版本也不一样，所以在Node.js能用的C++扩展在Electron里用不了，要重新编译成对应Electron内Node.js版本的二进制文件才能用，这就是为什么使用C++原生模块的情况下要把Electron依赖版本写死的原因。

Windows就是喜欢折腾嘛。先全局安装`node-gyp`，接下来有两条路：
1. 不喜欢折腾的，`npm install --global --production windows-build-tools`走一个，这个也得装不少时间。
2. 喜欢折腾的，一条龙安装.Net 4.5.1 （如果是WIN7） / Python2.7（对不起3就是不行你能拿我怎么样咧） / Visual Studio 2015（需要VC++ v140工具集）。

走第二条路的就注意了，最好是装VS2015，装VS2017某些用到了C++扩展的库（比如`sqlite3`）不能`rebuild`。

编译自己写的原生模块：`target`是Electron版本，`arch`是原生模块用在什么架构的应用上，`dist-url`是编译Electron原生模块需要的头文件下载地址。
``` bash
> node-gyp rebuild --target=1.8.2 --arch=ia32 --dist-url=https://atom.io/download/electron
```

我在这算做个笔记吧，安装`sqlite3`模块要这样安装，自己`rebuild`是不行的
``` bash
> npm install sqlite3 --save-dev --build-from-source --runtime=electron --target=1.8.2 --target_arch=ia32 --dist-url=https://atom.io/download/electron
```

## 打包应用

前面说过，Electron只是个壳，它本身已经是能够运行的桌面应用了。关键是它会跑开发者写的代码，加载HTML呈现界面，所以，我们只要连同自己写的代码和Electron的二进制文件一起打包就可以发布应用了。

至于怎么打包，只要把包含`package.json`的文件夹放在Electron文件夹下的`/resouces/app`目录下就可以了，然后启动Electron时Electron会加载我们写好的HTML，内容就成了我们的应用。

那么问题来了，这样的话可执行文件还是`electron.exe`！这不太OK。所以我们必须修改`electron.exe`的内容。看看下面的`pack.js`。

``` javascript
const unzip = require('unzip') // 用于解压Electron压缩包
const rcedit = require('rcedit') // 用于修改exe文件
const request = require('request') // 用于下载
const { slog, log, ilog, elog, wlog } = require('cmd-rainbow') // 用于打印彩色日志
const fs = require('fs')
const path = require('path')
const packageJson = require('../package.json')

pack({
  platform: 'win32', // 打包平台，我这里只支持win32
  arch: process.argv[2], // Electron架构
  electronVersion: packageJson.devDependencies.electron, // Electron版本
  packDir: path.join(__dirname, '..'), // 要打包的项目目录
  distDir: path.join(__dirname, '../dist'), // 输出目录
  ignore: new RegExp(`node_modules|build|dist|src|.gitignore|README|.eslintrc.json|package-lock.json|.git|.vscode`), // 过滤文件
  versionString: { // exe文件配置项，具体参见微软官方
    icon: path.join(__dirname, '../src/res/icon/app.ico'),
    'file-version': packageJson.version,
    'product-version': packageJson.version,
    'version-string': {
      // 'Block Header': '080404b0',
      FileDescription: packageJson.description,
      InternalName: packageJson.name,
      OriginalFilename: packageJson.name + '.exe',
      ProductName: packageJson.name,
      CompanyName: packageJson.author,
      LegalCopyright: `Copyright (C) ${new Date().getFullYear()} ${packageJson.author}`
    }
  }
})

function pack (option) { /* ... */}

/* ... */
```
然后`npm run pkg32`或`npm run pkg64`走一个，应用就打包出来在`/dist`目录下面了。

# 总结

咋废话那么多呢，走一个试试呗。

* [toyobayashi/electron-vue-simple](https://github.com/toyobayashi/electron-vue-simple)

``` bash
> npm install vue-cli -g
> vue init toyobayashi/electron-vue-simple [project-name]
> cd <project-name>
> npm install
> npm run prod
> npm start
```