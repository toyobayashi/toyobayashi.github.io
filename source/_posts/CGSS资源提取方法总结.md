---
title: CGSS资源提取方法总结
date: 2017-07-02 14:04:19
tags: CGSS
---

本文介绍一下提取手游《偶像大师灰姑娘女孩星光舞台》游戏资源的过程和方法。

<!--more-->

## 第一章　背景说明

### 什么是CGSS

由日本[Cygames](http://www.cygames.co.jp/)、[BANDAI NAMCO Entertainment Inc.](http://bandainamcoent.co.jp/)共同开发・运营的手机游戏《[アイドルマスターシンデレラガールズスターライトステージ](http://cinderella.idolmaster.jp/sl-stage/)》，简称《デレステ》，英文译名为《THE IDOL M@STER CINDERELLA GIRLS STARLIGHT STAGE》，中文译名为《偶像大师灰姑娘女孩星光舞台》，其首字母缩写即CGSS，是一款以偶像卡牌养成为主题的音乐节奏游戏。[Wiki传送门](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%82%A4%E3%83%89%E3%83%AB%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC_%E3%82%B7%E3%83%B3%E3%83%87%E3%83%AC%E3%83%A9%E3%82%AC%E3%83%BC%E3%83%AB%E3%82%BA)←

### 为什么要提取资源

喜欢，想收藏，网上找不到就只有自己动手咯。

### 声明

进入正篇前请注意，如果你是一位只求得到资源的伸手党，请右上角，本文不会给出完整的资源包，只会提供思路和方法。通过阅读本文成功提取出来的资源版权归BANDAI NAMCO Entertainment Inc.所有，请勿用于商业用途，如有因此引发的任何法律问题，一切与本文作者无关，后果自负。

## 第二章　前置分析

### 准备

* 【必需】耐心
* 【必需】[DB Browser for SQLite · Github](https://github.com/sqlitebrowser/sqlitebrowser/)
* 【非必需】[Fiddler](http://www.telerik.com/fiddler)
* 【非必需】英语基础、编程基础

### manifest资源表

当我想拿走别人的东西时，我首先得知道他的柜子是什么样的，还有我想要的东西被放在哪里。安卓系统下的柜子是

	Android/data/jp.co.bandainamcoent.BNEI0242/files

{% asset_img phone.png 游戏目录 %}

我第一次打开它的时候真的是傻眼了，这只有一个字母的目录命名到底是什么鬼！不仅如此，随便打开其中一个，里面的文件都是以一长串数字和字母的组合命名的，很明显被加密了，根本无法识别是什么东西。

总之先把整个目录复制到电脑上，无奈回到files目录下，从简单的地方开始入手，先不管那些只有一个字母的目录。来看看这个manifest目录，这个目录里始终都只有一个文件，并且我注意到只有游戏更新后这个文件才会发生改变，这就很关键了。一次偶然，我用万能的记事本打开了这个文件，发现了第一行中竟出现了「SQLite format 3」这几个字。

这就意味着，这是个SQLite3的数据库文件！

接下来用DB Browser for SQLite打开它，很庆幸这个数据库没有上锁，发现有一张名为「manifests」的表，里面有30000多条记录，仔细一看，第一列name下大部分都是.unity3d的后缀，还有少部分.acb、.bdb、.mdb的后缀，第二列的列名是hash。直觉告诉我，这就是游戏所依赖的所有文件和它们的md5哈希值，而且这游戏是用unity引擎开发的！

{% asset_img sql_00.png manifest %}

### 摸清目录结构

为了验证我的猜想，我用一段nodejs代码来计算文件的md5哈希值

``` javascript   md5.js
var crypto = require('crypto');
var fs = require('fs');
var args = process.argv.splice(2);

/* ... */

if(args.length === 1) {
    md5Hash(args[0]);
}

/* ... */

function md5Hash(file) {
    const hash = crypto.createHash('md5');
    const input = fs.createReadStream(file);
    input.on('readable', () => {
        const data = input.read();
        if (data){
            hash.update(data);
        }
        else {
            console.log(hash.digest('hex'));
        }
    });
}

/* ... */
```

我在b目录下随便选了一个叫1b4a29bf3b9c655b43cf1e6ea8273137acaefb0e的文件做试验，在命令行中执行下面的命令

``` bash 
$ node md5.js ./files/b/1b4a29bf3b9c655b43cf1e6ea8273137acaefb0e
```

结果如下

{% asset_img cmd_01.png 计算md5 hash值 %}

把f73a51c5022ce269f9684fe8643fc6f7粘贴到工具里筛选hash列

{% asset_img sql_01.png hash筛选结果 %}

果然出来了一行结果。仔细观察name，惊奇地发现前面有一个b/，而这个文件恰好又在b目录下，第一次见acb后缀虽然不知道是什么东西，但我觉得八九不离十应该是跟音频有关的。于是我有了一个大胆的猜测。尝试在name列筛选输入b/

{% asset_img sql_02.png b筛选结果 %}

得出结论，b目录下存放的都是背景音乐（bgm）。同理我也试了一下l/，v/，s/，c/，r/

{% asset_img sql_03.png l筛选结果 %}

{% asset_img sql_04.png v筛选结果 %}

{% asset_img sql_05.png s筛选结果 %}

{% asset_img sql_06.png c筛选结果 %}

{% asset_img sql_07.png r筛选结果 %}

进一步发现，l目录下存放的是乐曲（live），v目录下存放的是卡片语音（voice），s目录下存放的是音效（se），c目录下存放的是故事剧情的语音（communication），r目录下存放的是小屋物品的音效（room）。

大发现！别急，音频找到了，可是图片呢？排除了这几个目录，剩下就是a目录和d目录了，d目录下只有一个localData.db数据库文件，显然不是，那就只有a目录咯。a目录下有几千个文件，再看看资源表，有上万个unity3d文件，肯定跑不掉了，图片都藏在unity3d文件里。再用上面提到的md5.js去测试一下a目录下的文件，发现果然能和资源表里的unity3d文件匹配上。

到此为止，游戏柜子的结构已经基本上搞清楚了，列个表。

<style>
table th:first-of-type {
    width: 100px;
}
</style>

| 主要目录名   | 内容                            |
| :-----------: | -------------                    |
| a             | unity3d资源包 **A**sset         |
| b             | 背景音乐**B**gm                 |
| c             | 故事剧情的语音**C**ommunication |
| l             | 乐曲**L**ive                    |
| r             | 小屋物品音效**R**oom            |
| s             | 音效**S**e                      |
| v             | 卡片语音**V**oice               |
| manifest      | 资源表数据库                    |

### 抓包

如果你有一点编程基础，你完全可以按照上面讲的思路写个程序去算每个文件的md5 hash值，再去读资源表数据库拿hash值查对应的文件。如果你对代码一窍不通，那光知道每个目录下放的是什么东西还是不够的，因为你无法通过文件名来辨别哪个文件是藏着图片的资源包。现在唯一的线索只有资源表里写明的文件名，那我们就换个思路。

试想一下，游戏本身是从什么地方获取到这些文件的呢？

我使用Fiddler对游戏进行抓包（关于使用Fiddler对手机端http抓包的教程自行百度，本文不再多提），再按照上面说的原理经过一番hash值分析之后，得出了以下结论。

.unity3d文件的请求地址：

	http://storage.game.starlight-stage.jp/dl/resources/High/AssetBundles/Android/<哈希值>

.acb文件的请求地址：

	http://storage.game.starlight-stage.jp/dl/resources/High/Sound/Common/<目录名>/<哈希值>

.bdb或.mdb文件的请求地址：

	http://storage.game.starlight-stage.jp/dl/resources/Generic/<哈希值>

资源表数据库文件的请求地址：

	http://storage.game.starlight-stage.jp/dl/<资源版本>/manifests/Android_AHigh_SHigh

一切都清楚了，完事！

## 第三章　开始提取

有两种方法，一种是本地「提取」，一种是在线「偷取」。这里我介绍「在线偷取」的方法。音频和图片的提取步骤分开介绍。

### 准备

* 【必需】[wget · Github](https://github.com/mirror/wget)

#### 提取音频需要用到的工具

* 【推荐】[DereTore · Github](https://github.com/OpenCGSS/DereTore) （使用这个就不需要用下面两个）

或者

* 【可选】[vgmtoolbox](https://sourceforge.net/projects/vgmtoolbox/) （解压acb用）
* 【可选】[HCADecoder v1.15 · 百度网盘](http://pan.baidu.com/s/1i5Hvgrj) （解码hca用，日本人写的，我自己重新编译了，解决了中文环境下控制台输出乱码的问题，附源码）

#### 提取图片需要用到的工具

* 【必需】[UnityLz4 · Github](https://github.com/subdiox/UnityLz4) （建议使用[我重新编译好的](http://pan.baidu.com/s/1nuFAPct)，原始的输出后缀为.dec，我改成了.unity3d）
* 【必需】[Unity Assets Bundle Extractor](https://7daystodie.com/forums/showthread.php?22675-Unity-Assets-Bundle-Extractor) （先安装2.1版本，再打补丁升级到2.1d，不打补丁不能批处理）

#### 补充

由于从2017年6月22日开始游戏服务器设置了访问权限，需要伪造请求头才给正常返回文件，所以必须用wget设置好请求头才能正常下载。

【重要】**在wget目录下创建wget.ini，输入以下三行代码保存。**

``` ini 
header = User-Agent: Dalvik/2.1.0 (Linux; U; Android 7.0; Nexus 42 Build/XYZZ1Y)
header = X-Unity-Version: 5.1.2f1
header = Accept-Encoding: gzip
```

### 提取音频

这里以提取bgm和live为例子，其它的可以举一反三自行去发掘。

#### 获取资源表数据库文件

可以从手机中直接复制到电脑上，也可以使用wget从游戏服务器下载。资源版本可访问 https://starlight.kirara.ca/api/v1/info 查询。在wget目录下运行cmd命令行，执行以下命令开始下载。
``` bash
$ wget -c -O <保存路径和文件名> http://storage.game.starlight-stage.jp/dl/<资源版本>/manifests/Android_AHigh_SHigh 
```
【说明】-c设置断点续传，-O设置路径和文件名。

#### 查找数据库

使用DB Browser for SQLite打开资源表， 切换到Excute SQL选项卡，输入以下sql语句查找bgm和live。
``` sql 查找bgm
SELECT "http://storage.game.starlight-stage.jp/dl/resources/High/Sound/Common/b/"||hash AS url FROM manifests WHERE name LIKE "b/%acb"
```
``` sql 查找live
SELECT "http://storage.game.starlight-stage.jp/dl/resources/High/Sound/Common/l/"||hash AS url FROM manifests WHERE name LIKE "l/%acb"
```

#### 导出下载列表

右下角选择导出CSV，取消勾选列名在首行，字段分隔符为其他（空），保存为txt文件，如图所示。

{% asset_img sql_export_01.png 导出CSV %}

{% asset_img sql_export_02.png 导出选项 %}

{% asset_img sql_export_03.png 保存为txt %}

#### 使用wget批量下载

``` bash
$ wget -c -P <路径> -i <批量下载列表.txt>
```
【说明】-i设置批量下载列表，-P设置保存目录

#### Deretore批处理

如果你使用vgmtoolbox和HCADecoder，请跳过这一步。在Deretore目录下创建一个.bat批处理文件，右键编辑输入以下我写好的批处理脚本保存并执行
``` bat
@echo off
setlocal enabledelayedexpansion

set path=你的下载目录

set /a num=0
set /a complete=0

for %%i in (%path%\*.) do (
	AcbUnzip.exe %%i
	set /a num+=1
)

for /f "delims=" %%f in ('dir /s /b %path%\*.hca') do (
	hca2wav.exe -i %%f
	set /a complete+=1
	echo !complete! / %num%
)
md %path%\wav
for /f "delims=" %%w in ('dir /s /b %path%\*.wav') do move %%w %path%\wav\
for /f "delims=" %%d in ('dir /s /b %path%\_acb*') do rd %%d /s /q

```
【说明】把path替换为你的下载目录，该批处理会在下载目录下创建一个wav文件夹，把从acb文件中解包出来的.wav文件放在wav文件夹内。提取完毕。

#### 解压acb，导出hca

打开vgmtoolbox，进入
VGMToolbox→Misc. Tools→Extraction Tools→Common Archives→CRI ACB/AWB Archive Extractor
把下载好的所有文件拖入右侧。

#### 解码hca，导出wav

解压我编译好的HCADecoder，在该工具的目录下创建一个.bat批处理文件，右键编辑输入以下我写好的批处理脚本保存并执行
``` bat
@echo off

set path=你的下载目录

for /f "delims=" %%f in ('dir /s /b %path%\*.hca') do hca.exe -v 1.0 -m 16 -l 0 -a F27E3B22 -b 00003657 %%f
md %path%\wav
for /f "delims=" %%w in ('dir /s /b %path%\*.wav') do move %%w %path%\wav\
for /f "delims=" %%d in ('dir /s /b %path%\_vgmt*') do rd %%d /s /q

```
【说明】-v设置音量1倍，-m设置16进制模式，-l设置循环次数0为不循环，-a和-b是CGSS的hca解码密钥，固定不可变。把path替换为你的下载目录，该批处理会在下载目录下创建一个wav文件夹，把从acb文件中解包出来的.wav文件放在wav文件夹内，并且把vgmtoolbox生成的文件夹删除掉。提取完毕。

### 提取图片

这里以提取卡片背景为例子，其它的可以举一反三自行去发掘。

#### 查找数据库和批量下载

使用DB Browser for SQLite打开资源表， 切换到Excute SQL选项卡，输入以下sql语句查找卡片背景。
``` sql
SELECT "http://storage.game.starlight-stage.jp/dl/resources/High/AssetBundles/Android/"||hash AS url FROM manifests WHERE name LIKE "card_bg_______.unity3d"
```
下载的方法同提取音频。

#### lz4解压

在UnityLz4工具目录下创建一个.bat批处理文件，右键编辑输入以下我写好的批处理脚本保存并执行。
``` bat
@echo off

set path=你的下载目录

for %%i in (%path%\*.) do UnityLz4.exe %%i
md %path%\unity3d
for %%i in (%path%\*.unity3d) do move %%i %path%\unity3d\
```

【说明】从游戏服务器下载下来的文件也就是a目录下的文件并不是真正的unity3d文件，而是经过lz4压缩的.unity3d.lz4文件，所以必须先解压出真正的unity3d文件来。把path替换为你的下载目录，该批处理会在下载目录下创建一个unity3d文件夹，把从解压出来的unity3d文件放在此文件夹内。

#### UABE批处理

1. 在Unity Assets Bundle Extractor工具目录下新建一个batch.txt
``` txt
+DIR 你的unity3d文件存放目录
```
2. 在Unity Assets Bundle Extractor工具目录下新建一个batch.bat，输入以下内容保存后执行
``` bat
AssetBundleExtractor -removetypetree batchexport batch.txt
```

3. 打开Unity Assets Bundle Extractor，file→open打开所有解包出来的.assets文件，筛选type，选择所有类型为Texture2D的资源，点击Plugins批量导出png，如图所示

{% asset_img UABE_01.png 筛选类型 %}

{% asset_img UABE_02.png 导出png %}

提取完毕。

### 傻瓜模式

不想看或者看不懂长篇大论又或者没有动手能力的，来这里

* [CGSSAssetsDownloader · Github](https://github.com/toyobayashi/CGSSAssetsDownloader)

## 参考文献

* http://weibo.com/ttarticle/p/show?id=2309403943455800842944
* http://www.igiven.com/?p=612
* https://www.reddit.com/r/StarlightStage/comments/4nkk8o/event_bgm_download/
* http://tamae.2ch.net/test/read.cgi/gameurawaza/1457257198/592-n
* http://nozomi.2ch.sc/test/read.cgi/gameurawaza/1492929151/101-200
