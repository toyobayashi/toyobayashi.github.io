---
title: CGSS桌面应用mishiro背后的故事
date: 2018-04-01 14:11:22
tags: CGSS
---

一个菜鸟前端的自娱自乐。

<!--more-->

# 关于mishiro  

## mishiro是什么

mishiro是一个针对《偶像大师灰姑娘女孩星光舞台》这款音乐手游开发的Windows PC桌面端应用程序（或许以后可以跨平台）。它最大的用处是提取游戏里的各种资源，此外还提供游戏卡片数据的搜索、模拟游戏内的抽卡、估算玩家打活动的效率、模拟游戏LIVE的过程等功能。为什么取mishiro这个名字，是因为它在日语里写作汉字是「美城」，一说到灰姑娘，自然会联想到安徒生童话里的「美丽的城堡」。

## 为什么开发mishiro

去年大三实习，我正好在玩这个游戏，比较喜欢它里面的音乐，无奈在网上没有找到提取音乐资源的工具，就自己做一个得了，同时也正好可以拿来当作实习的成果。不过去年做的是网页版的CinderellaHelper（仓库已删除）和一个命令行工具CGSSAssetsDownloader。今年大四实习要搞毕设，正好可以把这两个东西结合起来改一下，做成更完善的一个东西。所以mishiro其实可以算是一年前就已经开始开发了。

## 用什么开发的

由于我比较菜，不会C++和C#，只会写JS，所以没有什么选择。  

壳应用框架Electron和NW.js二选一，我选Electron，看Electron更顺眼，Electron比NW.js更灵活，用VSCode也方便调试。  

前端框架Angular、Vue、React三选一，我选Vue，另外两个写得不多。

## 核心

1. 我是一个假客户端，我要和游戏服务器通信，没法通信什么都别谈。
2. 我要拿到我想要的数据（资源表、卡片、卡池、活动等等）。
3. 我要拿到我想要的资源（图片、音频、谱面等等）。

# 原理

## 与服务器通信

核心里的第一条就是个难点，毕竟不是我的服务器，不是谁敲门他都会开门，不过幸好有反编译游戏APK包看到了C#源码的老前辈们为后人铺好了路。

* 服务器：`game.starlight-stage.jp`

### 准备

当我们启动游戏在入口按下屏幕的那一瞬间，游戏客户端就往这里发了个POST请求，具体发了什么可以抓包看。这里插一句，由于我的手机是Android 7.0的系统，7.0以上不能轻易抓到HTTPS的包，安装了根证书也不行，是因为谷歌加了什么东西来着，还需要在应用里改，太麻烦了所以我用模拟器配合Fiddler抓。过程就不写了，直接说结论。

__发送请求需要三个必不可少的东西__

* UDID
* Viewer ID
* User ID

其中UDID(Unique Device Identifier)是唯一设备标识。形如  
`/^[0-9]{9}:[0-9]{9}:[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/`
和GUID的格式一样。Viewer ID就是我们在游戏里可以看到的9位数的ID。User ID也是9位数的ID，但是和Viewer ID不一样。用户注册时Viewer ID和User ID由服务器生成。这三个东西就相当于是一个帐号。至于上哪儿找——抓包。

UDID和User ID被加密放在请求头`UDID`和`USER_ID`中。加密的算法是这样的，前四位数是16进制，用来表示字符串的长度。例如长度为36的UDID，加密后的前四位就是`0024`。随后，分别在字符串中的每一个字符前加两位随机数，在后面加一位随机数，并把该字符变成ascii码中后十位的字符。最后加上32位的随机数。用JS实现加解密如下：
``` javascript
function encode (s) {
  let arr = []
  for (let i = 0; i < s.length; i++) {
    let c = s[i]
    arr.push(createRandomNumberString(2) + chr(ord(c) + 10) + createRandomNumberString(1))
  }
  return $04x(s.length) + arr.join('') + createRandomNumberString(32)
}

function decode (s) {
  let l = parseInt(s.substr(0, 4), 16)
  let e = ''
  for (let i = 6; i < s.length; i += 4) {
    e += s[i]
  }
  e = e.substr(0, l)
  let arr = []
  for (let i = 0; i < e.length; i++) {
    arr.push(chr(ord(e[i]) - 10))
  }
  return arr.join('')
}

function chr (code) {
  return String.fromCharCode(code)
}

function ord (str) {
  return str.charCodeAt()
}

function createRandomNumberString (l) {
  let s = ''
  for (let i = 0; i < l; i++) {
    s += Math.floor(10 * Math.random())
  }
  return s
}

function $04x (n) {
  let s = n.toString(16)
  let d = 4 - s.length
  return Array.from({ length: d }, () => '0').join('') + s
}
```
有了前面说的这三个东西，并且他们是正确的，剩下的就是拼接请求头和请求体了。注意每个接口需要的请求参数都不一样，但是请求参数中必须包含`timezone`和`viewer_id`字段。

### 请求头

需要下面这些请求头：

|头名     |说明    |
|:------:|:-------|
|HOST|game.starlight-stage.jp|
|User-Agent| |
|Content-Type|application/x-www-form-urlencoded|
|Content-Length| |
|PARAM|被sha1加密的UDID、Viewer ID、请求路径、请求参数|
|KEYCHAIN|KEYCHAIN ID|
|CARRIER|运营商|
|APP_VER|游戏应用版本|
|RES_VER|资源版本号|
|IP_ADDRESS|IP地址|
|DEVICE_NAME|设备名|
|X-Unity-Version|Unity版本|
|GRAPHICS_DEVICE_NAME|GPU名|
|DEVICE_ID|设备固有ID|
|PLATFORM_OS_VERSION|系统版本信息|
|DEVICE|引继次数|
|USER_ID|被上述方法加密的User ID|
|UDID|被上述方法加密的UDID|
|SID|被md5加密的SESSION ID与SESSION ID KEY拼接的字符串，SESSION ID第一次为Viewer ID和UDID的拼接串，后使用由服务器返回的session串，SESSION ID KEY为`r!I@nt8e5i=`|

### 请求体

前面说了不同的接口需要不同的请求参数，请求参数中必须存在`timezone`和`viewer_id`字段。`timezone`没什么好说的，就是日本时间与UTC时间的时间差`09:00:00`，`viewer_id`按以下步骤加密Viewer ID：
1. 生成一个32位的随机列（形如`/^[0-9]{32}$/`）
2. 使用Rijndael-256-cbc算法加密Viewer ID，向量为第1步生成的32位随机数，key为`s%5VNQ(H$&Bqb6#3+78h29!Ft4wSg)ex`
3. 使用base64编码加密后的数据
4. 把刚才的向量拼接在base64字符串的最前面，就得到了`viewer_id`字段里要填的数据。

接下来按以下步骤生成请求体：
1. 使用MessagePack把请求参数转换成二进制数据
2. 使用base64编码第1步得到的二进制数据
3. 32次重复生成并自拼接4位以下的16进制随机列
4. 使用base64编码这个随机列，并取编码后的前32位
5. 使用Rijndael-256-cbc算法加密第2步得到的数据，向量为去掉所有`-`的UDID，key为第4步得到的数据
6. 把加密的key拼接在加密后数据的最后
7. 使用base64编码第6步得到的数据，就得到了请求体

### 响应体

按以下步骤解密：
1. base64解码
2. 使用Rijndael-256-cbc算法解密解码后的第1位至倒数第33位，向量为去掉所有`-`的UDID，key为解码后的倒数第32位至倒数第1位
3. 使用base64解码第2步得到的数据
4. 使用MessagePack解码第3步得到的数据，就得到了JSON明文

### JavaScript简单伪代码实现

``` javascript
let sid = ''

function post (path, args) {
  let viewerIV = createRandomNumberString(32)
  args.timezone = '09:00:00'
  args.viewer_id = viewerIV + b64encode(encryptRJ256(viewerID, viewerIV, VIEWER_ID_KEY))
  let plain = b64encode(msgpack.encode(args))
  let key = b64encode($xFFFF32()).substring(0, 32)
  let bodyIV = UDID.replace(/-/g, '')
  let body = b64encode(encryptRJ256(plain, bodyIV, key) + key)
  sid = sid ? sid : viewerID + UDID
  let headers = {
    'PARAM': sha1(UDID + viewerID + path + plain),
    'KEYCHAIN': '',
    'USER_ID': encode(userID),
    'CARRIER': 'google',
    'UDID': encode(UDID),
    'APP_VER': '9.9.9',
    'RES_VER': resVer,
    'IP_ADDRESS': '127.0.0.1',
    'DEVICE_NAME': 'Nexus 42',
    'X-Unity-Version': '5.1.2f1',
    'SID': md5(sid + SID_KEY),
    'GRAPHICS_DEVICE_NAME': '3dfx Voodoo2 (TM)',
    'DEVICE_ID': md5('Totally a real Android'),
    'PLATFORM_OS_VERSION': 'Android OS 13.3.7 / API-42 (XYZZ1Y/74726f6c6c)',
    'DEVICE': '2',
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Dalvik/2.1.0 (Linux; U; Android 13.3.7; Nexus 42 Build/XYZZ1Y)'
  }
  return new Promise((resolve, reject) => {
    request({
      url: `https://game.starlight-stage.jp${path}`,
      timeout: 10000,
      headers: headers,
      body: body
    }, (err, res, body) => {
      if (err) reject(err)
      else {
        let bin = Buffer.from(body, 'base64')
        let data = bin.slice(0, bin.length - 32)
        let key = bin.slice(bin.length - 32).toString('ascii')
        let plain = decryptRJ256(data, bodyIV, key)

        let msg = msgpack.decode(Buffer.from(plain, 'base64'))
        sid = typeof msg === 'object' ? (msg.data_headers ? msg.data_headers.sid : void 0) : void 0
        resolve(msg)
      }
    })
  })
}

function $xFFFF32 () {
  let s = ''
  for (let i = 0; i < 32; i++) {
    s += Math.floor(65536 * Math.random()).toString(16)
  }
  return s
}
```

这样一来，通过`/load/check`就很方便自动更新资源版本了，以前不懂原理的时候只有用暴力的方法，用一个一个版本号去请求服务器看是不是404。

## 提取资源

* 资源存储服务器：`storage.game.starlight-stage.jp`
* 必需的请求头：`X-Unity-Version`

### 资源版本号

每次卡池更新、活动更新、增加新曲等等数据变化的时候，资源版本号就会改变，进入游戏时会弹窗提示「有新的数据需要下载」。

资源版本号是8位的数字（如10037050），99%是10的倍数（我曾遇到过个位是5的版本号）。每次更新两个版本号的差99%不会超过200（以前最多是190），大多数时候是更新10或更新百位数。

### 资源清单Manifest

资源清单是一个SQLite3数据库，里面包含了所有（？）游戏要使用的资源。所有的资源清单列表地址为：
`/dl/资源版本号/manifests/all_dbmanifest`
获取地址为：
`/dl/资源版本号/manifests/Android_AHigh_SHigh`

通过以上URL下载下来的文件是经过LZ4压缩的文件，LZ4解压后就是数据库了。另外，也可以在手机的游戏目录下找到这个文件。

使用SQLite浏览工具打开数据库，可以看到有一个`manifests`表，这个表里面写的就是游戏需要使用的资源。主要是unity3d文件，acb文件和bdb(Blob Database)文件和mdb(Master Database)文件。以下三个字段为关键字段：
* name: 文件名
* hash: LZ4解压前的文件MD5
* attr: 是否使用LZ4压缩，1为压缩，0为未压缩

获取地址：
* unity3d文件：`/dl/resources/High/AssetBundles/Android/${hash}`（部分404）
* acb文件：`/dl/resources/High/Sound/Common/${name.split('/')[0]}/${hash}`
* bdb文件和mdb文件：`/dl/resources/Generic/${hash}`

### 主数据库Master

即资源清单中的`master.mdb`。主数据库内包含游戏内的各种数据，包括角色、卡片、活动等等。它同样也是SQLite3数据库，下载下来需要先LZ4解压。地址是：
`/dl/resources/Generic/${hash}`

### 图片

可以使用Unity Assets Bundle Extractor来批量提取unity3d文件内的texture2d。我不懂Unity文件格式，所以没办法只有用工具提取UI贴图了，在mishiro中是直接从`starlight.kirara.ca`获取卡面和头像的图片。

### ACB文件的格式

CGSS的音频用的是CRI Middleware公司提供的格式。简单概括一下就是acb文件里有acb、awb文件，awb文件里有hca文件，acb文件里有utf表，根据表就能从awb文件中提取出hca文件。acb并没有压缩hca文件，只是把hca和其它的utf表和一些索引混在了一起。

ACB文件的二进制数值采用的是__大端序__。

#### 文件头

```
0x0000 | 40 55 54 46 | 固定（'@UTF'）
0x0004 | 00 39 02 78 | 除了文件头的文件大小（也就是总大小减8）
```

#### 表头

8位以后开始就是表头，下面所有的偏移值都从这里开始计算

```
0x0000 | 00 01       | 不清楚
0x0002 | 01 58       | 表数据的偏移值
0x0004 | 00 00 02 8D | 字符串数据的偏移值
0x0008 | 00 00 05 B8 | 二进制数据的偏移值
0x000c | 00 00 00 00 | 字符串数据中表名字符串的偏移值
0x0010 | 00 40       | 列数
0x0012 | 01 35       | 各行总字节数（除了常数）
0x0014 | 00 00 00 01 | 行数
```

#### 列字段数据

指定各列的类型和列名。列类型是 0x10 或 0x50 的情况就是5字节，0x30 或 0x70 的情况下在5字节后面有常数。

```
0x0018 | 54          | 类型 ID
0x0019 | 00 00 00 07 | 字符串数据中列名字符串的偏移值

0x001d | 54          | 类型 ID
0x001e | 00 00 00 16 | 字符串数据中列名字符串的偏移值

...

0x0153 | 5B          | 类型 ID
0x0154 | 00 00 02 D2 | 字符串数据中列名字符串的偏移值
```

类型 ID 是列类型和该列数据类型的逻辑或（|）。

| 列类型ID | 内容     |
|--------|----------|
| 0x10   | 固定0    |
| 0x30   | 常数     |
| 0x50   | 常规     |
| 0x70   | 常数     |

| 数据类型ID | 名称    | 表数据上所占长度                          |
|------|---------|-----------------------------------------------------|
| 0x00 | UInt8   | 1                                                   |
| 0x01 | Int8    | 1                                                   |
| 0x02 | UInt16  | 2                                                   |
| 0x03 | Int16   | 2                                                   |
| 0x04 | UInt32  | 4                                                   |
| 0x05 | Int32   | 4                                                   |
| 0x06 | UInt64  | 8                                                   |
| 0x07 | Int64   | 8                                                   |
| 0x08 | Float   | 4                                                   |
| 0x09 | Double  | 8                                                   |
| 0x0A | String  | 4（字符串数据内的偏移值）                             |
| 0x0B | Binary  | 8（前四位是二进制数据内的偏移值，后四位是长度）         |

#### 表数据

列字段数据接下来就是表数据，从表头里的「表数据的偏移值」的位置开始。

```
0x0158 | 00 00 00 00             | 第一列的数据（Int32）
0x015c | 00 00 00 00             | 第二列的数据（Int32）
...
0x0285 | 00 00 00 00 00 00 00 00 | 最后（第64列）一列的数据（Binary）
```

#### 字符串数据

接下来是字符串数据，每个单词以 00 （NULL）结尾。从表头里的「字符串数据的偏移值」的位置开始。第一个单词是表名。

```
0x028d | 48 65 61 64 65 72 00                         | Header（表名）
0x0294 | 46 69 6C 65 49 64 65 6E 74 69 66 69 65 72 00 | FileIdentifier（第一列的列名）
...
0x0596 | 73 6F 6E 67 5F 33 30 32 39 00                | song_3029（第35列「Name」下的字符串）
```

#### 二进制数据

接下来是二进制数据，从表头里的「二进制数据的偏移值」的位置开始。

```
0x05b8 | 0E F7 50 41 ... | 第6列「AcfMd5Hash」下的二进制
...
```

### AWB文件的格式

AWB文件的二进制数值采用的是__小端序__。

#### 文件头

```
0x0000 | 41 46 53 32 | 固定（'AFS2'）
0x0004 | 01          | 不清楚
0x0005 | 04          | 数据长度
0x0006 | 02 00       | 不清楚
0x0008 | 24 00 00 00 | 包含的文件数
0x000c | 20 00 00 00 | 校准值
```

#### 文件索引

```
0x0010 | 00          | ID
0x0011 | 00          | 不清楚

0x0012 | 01          | ID
0x0013 | 00          | 不清楚

...

0x0056 | 23          | ID
0x0057 | 00          | 不清楚
```

#### 文件终点位置

头里面的「数据长度」是多少字节这里就是多少字节，这个文件是4字节。总长度为包含的文件数＋1。

```
0x0058 | EC 00 00 00 | 头终点（236）
0x005c | 60 52 01 00 | 第一个文件的终点（86624）
0x0060 | C0 12 02 00 | 第二个文件的终点（135872）
...
```

文件的起始点一定是校准值的倍数。所以，前一个文件的终点位置向上取到第一个校准值的倍数就是当前文件的起始点。比如上面的例子里，第一个文件是从 0x0100 开始 而不是 从 0x00EC 开始。

### 处理HCA

HCA完全搞不懂，mishiro直接用别人写的C++解码库，通过编译成Node.js的C++扩展模块，供JavaScript调用。HCA解码后就得到了WAV文件，再使用FFMPEG转成MP3，喂给HTML5 Audio。

## 附表

### `data_headers.result_code`一览
| result_code | 错误名                                     | 说明                             |
|------------:|----------------------------------------------|----------------------------------|
| 1           |                                              | 成功                             |
| 101         | RC_MAINTENANCE                               |                                  |
| 102         |                                              | 服务器错误                        |
| 201         | RC_SESSION_ERROR                             |                                  |
| 202         |                                              | 用户不存在                        |
| 203         |                                              | 帐号被封禁                        |
| 204         | RC_VERSION_ERROR                             | 有新的应用版本                    |
| 207         | RC_NG_WORD_ERROR                             | 含有被禁止的词语                   |
| 209         |                                              | 请求体解析失败                     |
| 213         |                                              | 数据处理失败                      |
| 214         | RC_RESOURCE_VERSION_ERROR                    | 有新的资源版本                    |
| 300         | RC_PAYMENT_HISTORY_ERROR                     |                                  |
| 301         | RC_PAYMENT_ALREADY_ERROR                     |                                  |
| 302         | RC_PRODUCT_DATA_CSV_ERROR                    |                                  |
| 303         | RC_PAYMENT_RECEIPT_ERROR                     |                                  |
| 304         | RC_PAYMENT_LIMIT_ERROR                       |                                  |
| 305         | RC_PAYMENT_TIMEOUT_ERROR                     |                                  |
| 306         | RC_PAYMENT_RESPONSE_ERROR                    |                                  |
| 307         | RC_PAYMENT_AGE_GROUP_ERROR                   |                                  |
| 308         | RC_PAYMENT_VALIDATION_ERROR                  |                                  |
| 516         | RC_TRANSITION_OLD_ACCESS_ERROR               |                                  |
| 900         | RC_PLATFORM_ERROR                            |                                  |
| 1202        | RC_LIVE_PLAYING_STATE_MISMATCH_ERROR         |                                  |
| 1203        | RC_LIVE_PERIOD_ERROR                         |                                  |
| 1219        | RC_PINYA_REQUEST_DISABLE                     |                                  |
| 1220        | RC_PINYA_REQUEST_NOT_FOUND_ERROR             |                                  |
| 1251        | RC_STORY_ITEM_OUT_PERIOD_ERROR               |                                  |
| 1252        | RC_STORY_ITEM_NOT_ENOUGH_ERROR               |                                  |
| 1253        | RC_STORY_ITEM_NUM_INVALID_ERROR              |                                  |
| 1254        | RC_STORY_ALREADY_OPEN_ERROR                  |                                  |
| 1255        | RC_STORY_OPEN_OUT_PERIOD_ERROR               |                                  |
| 1302        | RC_ROOM_ITEM_BUY_ERROR                       |                                  |
| 1303        | RC_ROOM_DATA_UNSERIALIZE_ERROR               |                                  |
| 1304        | RC_ROOM_ITEM_SELL_ERROR                      |                                  |
| 1305        | RC_ROOM_ITEM_LEVELUP_ERROR                   |                                  |
| 1306        | RC_ROOM_ITEM_STOP_LEVELUP_ERROR              |                                  |
| 1307        | RC_ROOM_ALREADY_LIKE_ERROR                   |                                  |
| 1308        | RC_ROOM_ITEM_LEVELUP_SHORT                   |                                  |
| 1309        | RC_ROOM_ITEM_END_LEVELUP_ERROR               |                                  |
| 1310        | RC_ROOM_MUSIC_BUY_ERROR                      |                                  |
| 1311        | RC_ROOM_NOT_SOUND_BOOTH_ERROR                |                                  |
| 1312        | RC_ROOM_ITEM_LEVELUP_ERROR_BY_CHARGE         |                                  |
| 1313        | RC_ROOM_USE_JEWEL_ERROR                      |                                  |
| 1314        | RC_ROOM_ITEM_PRESENT_ERROR                   |                                  |
| 1315        | RC_ROOM_NOW_LEVELUP_ERROR                    |                                  |
| 1317        | RC_ROOM_TYPE_ERROR                           |                                  |
| 1318        | RC_ROOM_ITEM_NOT_LEVELUP_START_ERROR         |                                  |
| 1319        | RC_ROOM_ITEM_MAX_LEVEL_ERROR                 |                                  |
| 1320        | RC_ROOM_EPISODE_PT_GET_PIRIODO_ERROR         |                                  |
| 1321        | RC_ROOM_RECEIVE_CHARGE_ERROR                 |                                  |
| 1322        | RC_ROOM_NOT_CHARGE_ITEM_ERROR                |                                  |
| 1327        | RC_ROOM_BULK_STORAGE_DISABLED                |                                  |
| 1328        | RC_ROOM_BULK_BUY_DISABLED                    |                                  |
| 1329        | RC_ROOM_BULK_PRESENT_DISABLED                |                                  |
| 1330        | ROOM_MYSET_DISABLED                          |                                  |
| 1351        | RC_GACHA_INVALID_PERIOD_ERROR                |                                  |
| 1353        | RC_GACHA_DAILY_FREE_STARTED_ERROR            |                                  |
| 1354        | RC_GACHA_DAILY_FREE_FINISHED_ERROR           |                                  |
| 1355        | RC_GACHA_DAILY_FREE_NOT_AVAILABLE_ERROR      |                                  |
| 1356        | RC_GACHA_DAILY_FREE_AVAILABLE_ERROR          |                                  |
| 1451        | RC_FRINED_COUNT_OVER_ERROR                   |                                  |
| 1452        | RC_PARTNER_FRIEND_COUNT_OVER_ERROR           |                                  |
| 1453        | RC_IS_FRIEND_ERROR                           |                                  |
| 1454        | RC_REQUEST_MAX_ERROR                         |                                  |
| 1455        | RC_IS_FRIEND_APPLY_ERROR                     |                                  |
| 1456        | RC_GET_UNREAD_FRIEND_REQUEST                 |                                  |
| 1457        | RC_SEARCH_USER_NOT_FOUND                     |                                  |
| 1458        | RC_FRIEND_UNREAD_REQUEST_MAX_ERROR           |                                  |
| 1459        | RC_FRIEND_APPLY_NOT_FOUND_ERROR              |                                  |
| 1460        | RC_FRIEND_DENY_NOT_FOUND_ERROR               |                                  |
| 1461        | RC_FRIEND_CANSEL_NOT_FOUND_ERROR             |                                  |
| 1462        | RC_FRIEND_ACCEPT_NOT_FOUND_ERROR             |                                  |
| 1551        | RC_MESSAGE_COMMENT_LENGTH_ERROR              |                                  |
| 1901        | RC_EVOLUTION_ERROR_DUPLICATE_CARD_ID         |                                  |
| 1903        | RC_IDOL_POTENTIAL_DISABLE                    |                                  |
| 2001        | RC_PRESENT_RECEIVE_ERROR_NO_PRESRNT          |                                  |
| 2051        | RC_NOT_ENOUGH_ERROR                          |                                  |
| 2053        | RC_ITEM_PERIOD_PASSED                        |                                  |
| 2102        | RC_STAMINA_MAX_ERROR                         |                                  |
| 2103        | RC_USER_STATUS_ERROR                         |                                  |
| 2201        | RC_MAINTENANCE_TASK_LOAD                     |                                  |
| 2202        | RC_MAINTENANCE_TASK_LIVE                     |                                  |
| 2203        | RC_MAINTENANCE_TASK_ROOM                     |                                  |
| 2204        | RC_MAINTENANCE_TASK_STORY                    |                                  |
| 2205        | RC_MAINTENANCE_TASK_PRESENT                  |                                  |
| 2206        | RC_MAINTENANCE_TASK_SIGNUP                   |                                  |
| 2207        | RC_MAINTENANCE_TASK_PAYMENT_IOS              |                                  |
| 2208        | RC_MAINTENANCE_TASK_PAYMENT_ANDROID          |                                  |
| 2209        | RC_MAINTENANCE_TASK_SHOP                     |                                  |
| 2210        | RC_MAINTENANCE_TASK_GACHA_RARE               |                                  |
| 2211        | RC_MAINTENANCE_TASK_GACHA_FRIEND             |                                  |
| 2212        | RC_MAINTENANCE_TASK_GACHA_TICKET             |                                  |
| 2213        | RC_MAINTENANCE_TASK_CAMPAIGN                 |                                  |
| 2214        | RC_MAINTENANCE_TASK_CAMPAIGN_SERIAL          |                                  |
| 2215        | RC_MAINTENANCE_TASK_CAMPAIGN_INVITE          |                                  |
| 2216        | RC_MAINTENANCE_TASK_FAVORITE                 |                                  |
| 2217        | RC_MAINTENANCE_TASK_MEMBER                   |                                  |
| 2218        | RC_MAINTENANCE_TASK_TRAINING                 |                                  |
| 2219        | RC_MAINTENANCE_TASK_UNIT                     |                                  |
| 2220        | RC_MAINTENANCE_TASK_FRIEND                   |                                  |
| 2221        | RC_MAINTENANCE_TASK_MESSAGE                  |                                  |
| 2222        | RC_MAINTENANCE_TASK_ALBUM                    |                                  |
| 2223        | RC_MAINTENANCE_TASK_ITEM                     |                                  |
| 2224        | RC_MAINTENANCE_TASK_PANEL_MISSION            |                                  |
| 2225        | RC_MAINTENANCE_TASK_PROFILE                  |                                  |
| 2226        | RC_MAINTENANCE_TASK_TIPS                     |                                  |
| 2227        | RC_MAINTENANCE_TASK_TUTORIAL                 |                                  |
| 2228        | RC_MAINTENANCE_TASK_MISSION                  |                                  |
| 2229        | RC_MAINTENANCE_TASK_SHOP_JEWEL_SHOP          |                                  |
| 2230        | RC_MAINTENANCE_TASK_PARTY                    |                                  |
| 2231        | RC_MAINTENANCE_TASK_MONEY_SHOP               |                                  |
| 3001        | RC_JEWEL_SHOP_ERROR_TYPE_NO_PERIOD_CSV       |                                  |
| 3002        | RC_JEWEL_SHOP_ERROR_TYPE_NO_PERIOD_USER      |                                  |
| 3003        | RC_JEWEL_SHOP_ERROR_OVER_BUY_MAX             |                                  |
| 3157        | RC_MISSION_ERROR                             |                                  |
| 3204        | RC_GIFT_CL_BIRTHDAY                          |                                  |
| 3251        | RC_NAME_CARD_UNSERIALIZE_ERROR               |                                  |
| 3252        | RC_NAME_CARD_APPLY_NOT_FOUND_ERROR           |                                  |
| 3253        | RC_NAME_CARD_APPLY_ERROR                     |                                  |
| 3254        | RC_NAME_CARD_HAS_BEEN_ALREADY_ACCEPTED_ERROR |                                  |
| 3255        | RC_NAME_CARD_CANSEL_NOT_FOUND_ERROR          |                                  |
| 3256        | RC_NAME_CARD_ACCEPT_NOT_FOUND_ERROR          |                                  |
| 3257        | RC_NAME_CARD_DENY_NOT_FOUND_ERROR            |                                  |
| 3258        | RC_NAME_CARD_NOT_FOUND_ERROR                 |                                  |
| 3259        | RC_NAME_CARD_LIST_NOT_FOUND_ERROR            |                                  |
| 3260        | RC_NAME_CARD_DISTRIBUTION_INVALID_ERROR      |                                  |
| 3261        | RC_NAME_CARD_MAX_POSSESSION_OVER_ERROR       |                                  |
| 3262        | RC_NAME_CARD_DOUBLE_REQUEST_ERROR            |                                  |
| 3263        | RC_NAME_CARD_EXCHANGE_FINISHED_ERROR         |                                  |
| 3264        | RC_NAME_CARD_CHARA_ID_NOT_FOUND_ERROR        |                                  |
| 3265        | RC_NAME_CARD_BG_ID_NOT_FOUND_ERROR           |                                  |
| 3266        | RC_NAME_CARD_NO_RELEASE_ERROR                |                                  |
| 3268        | RC_NAME_CARD_OWN_EXCHANGE_REQUEST_ERROR      |                                  |
| 3269        | RC_NAME_CARD_DISABLE                         |                                  |
| 3351        | RC_EXCHANGE_SHOP_CLOSE                       |                                  |
| 3352        | RC_EXCHANGE_SHOP_MAX_OVER_ERROR              |                                  |
| 3353        | RC_EXCHANGE_SHOP_DISCOUNT_ERROR              |                                  |
| 3354        | RC_EXCHANGE_SHOP_COST_ERROR                  |                                  |
| 3355        | RC_EXCHANGE_SHOP_ITEM_NOT_ENOUGH_ERROR       |                                  |
| 3356        | RC_EXCHANGE_SHOP_ITEM_MAX_OVER_ERROR         |                                  |
| 3357        | RC_EXCHANGE_SHOP_NO_RELEASE_ERROR            |                                  |
| 10001       | RC_EVENT_CLOSE_PLAY_EVENT_LIVE               |                                  |
| 11001       | RC_EVENT_ATAPON_CLOSE                        |                                  |
| 11004       | RC_EVENT_ATAPON_LAST_TIME_OVER               |                                  |
| 12001       | RC_EVENT_CARAVAN_CLOSE                       |                                  |
| 12005       | RC_EVENT_CARAVAN_EXCHANGE_CLOSE              |                                  |
| 12006       | RC_EVENT_CARAVAN_EXCHANGE_COST_ERROR         |                                  |
| 13001       | RC_EVENT_MEDLEY_CLOSE                        |                                  |
| 13003       | RC_EVENT_START_LIVE_AND_MEDLEY_CLOSE         |                                  |
| 13004       | RC_EVENT_MEDLEY_LAST_TIME_OVER               |                                  |
| 14001       | RC_EVENT_PARTY_CLOSE                         |                                  |
| 14003       | RC_EVENT_PARTY_LAST_TIME_OVER                |                                  |
| 14011       | RC_EVENT_PARTY_EVENT_PLAY_LEVEL_OUT_ERROR    |                                  |
| 14101       | RC_EVENT_PARTY_NON_HOST_ERROR                |                                  |
| 14102       | RC_EVENT_PARTY_NON_SET_EVENT_UNIT_ERROR      |                                  |
| 14103       | RC_EVENT_PARTY_NON_EVENT_START_ERROR         |                                  |
| 14104       | RC_EVENT_PARTY_ROOM_CLOSE                    |                                  |
| 15001       | RC_EVENT_TOUR_CLOSE                          |                                  |
| 15003       | RC_EVENT_TOUR_LAST_TIME_OVER                 |                                  |

### 接口一览

| ID  | ApiType.Type              | URL                                  |
|----:|---------------------------|--------------------------------------|
| 0   | VersionCheck              | load/check                           |
| 1   | SignUp                    | tool/signup                          |
| 2   | PaymentStart              | payment/start                        |
| 3   | PaymentCancel             | payment/cancel                       |
| 4   | PaymentFinish             | payment/finish                       |
| 5   | PaymentItemList           | payment/item_list                    |
| 6   | UpdateBirth               | payment/update_birth                 |
| 7   | Load                      | load/index                           |
| 8   | Tutorial                  | tutorial/update_step                 |
| 9   | MemberUnitEdit            | unit/edit                            |
| 10  | MemberUnitRename          | unit/rename                          |
| 11  | MemberFavoriteEdit        | favorite/edit                        |
| 12  | MemberFavoriteRename      | favorite/rename                      |
| 13  | MemberTraining            | training/reinforce                   |
| 14  | MemberTrainingTrainer     | training/reinforce_item              |
| 15  | MemberEvolve              | training/evolve                      |
| 16  | MemberExceed              | training/exceed                      |
| 17  | MemberSell                | member/sell                          |
| 18  | MemberProtect             | member/protect_card                  |
| 19  | MemberPotentialUp         | training/potential_up                |
| 20  | MemberPotentialInit       | training/potential_initialize        |
| 21  | MemberCardStorageList     | card_storage/storage_list            |
| 22  | MemberCardStorageChange   | card_storage/change_storage          |
| 23  | MemberCardStorageSetting  | card_storage/setting                 |
| 24  | MemberCardStorageCardList | card_storage/card_list               |
| 25  | StoryStart                | story/start                          |
| 26  | StoryReleaseEventStory    | story/open_v2                        |
| 27  | LiveList                  | live/music_list                      |
| 28  | LiveRanking               | live/get_live_detail_ranking         |
| 29  | LiveFriendList            | live/supporter                       |
| 30  | LiveStart                 | live/start                           |
| 31  | LiveEnd                   | live/end                             |
| 32  | LiveContinue              | live/continue_game                   |
| 33  | LivePreContinueProcess    | live/continue_pre_process            |
| 34  | LiveInterrupt             | live/interrupt                       |
| 35  | LiveSendLog               | live/send_log                        |
| 36  | LiveStartView             | live/start_view                      |
| 37  | LiveDiscardStart          | live/discard_start                   |
| 38  | LiveDiscardEnd            | live/discard_end                     |
| 39  | LiveStartRehearsal        | live/start_rehearsal                 |
| 40  | LiveEndRehearsal          | live/end_rehearsal                   |
| 41  | LiveLiveLot               | live/live_lot                        |
| 42  | RoomLoad                  | room/index                           |
| 43  | RoomChangeLoad            | room/change_room                     |
| 44  | RoomBuy                   | room/buy                             |
| 45  | RoomBuyStorage            | room/buy_to_storage                  |
| 46  | RoomBuyMusic              | room/buy_music                       |
| 47  | RoomSell                  | room/sell                            |
| 48  | RoomUpdate                | room/update                          |
| 49  | RoomLevelUpStart          | room/levelup                         |
| 50  | RoomLevelUpComplete       | room/levelup                         |
| 51  | RoomLevelUpStop           | room/levelup                         |
| 52  | RoomLevelUpCutTime        | room/levelup                         |
| 53  | RoomLike                  | room/add_like                        |
| 54  | RoomLikeHistory           | room/get_liked_history_list          |
| 55  | RoomOther                 | room/other                           |
| 56  | RoomDeliveryBoxReceive    | room/receive_parcel                  |
| 57  | RoomDeliveryBoxUpdate     | room/update_delivery_box             |
| 58  | RoomTicketBoardReceive    | room/receive_ticket                  |
| 59  | RoomTicketBoardUpdate     | room/update_ticket_board             |
| 60  | RoomVisitRandom           | room/random_visit                    |
| 61  | RoomExtendBox             | room/extend_box                      |
| 62  | RoomGiftUpdate            | room/update_gift                     |
| 63  | RoomGiftReceive           | room/receive_gift                    |
| 64  | RoomGivePresent           | gift/give_present                    |
| 65  | RoomMoveItem              | room/move_room_item                  |
| 66  | RoomUpdateSetting         | room/update_room_setting             |
| 67  | RoomFavoriteEdit          | room/edit                            |
| 68  | RoomReceiveAll            | room/receive_all_charge_item         |
| 69  | RoomIdolPresent           | room/receive_idol_present            |
| 70  | RoomMySetLoad             | room/get_layout                      |
| 71  | RoomMySetSave             | room/update_layout                   |
| 72  | RoomMySetDelete           | room/clear_layout                    |
| 73  | RoomMySetRename           | room/rename_layout                   |
| 74  | RoomMySetUpdate           | room/update_with_charge_items        |
| 75  | GachaLoad                 | gacha/index                          |
| 76  | GachaExe                  | gacha/exec                           |
| 77  | FriendTop                 | friend/top                           |
| 78  | FriendList                | friend/friend_list                   |
| 79  | FriendApplyingList        | friend/request_list                  |
| 80  | FriendApprovalList        | friend/approval_pending_list         |
| 81  | FriendSearch              | friend/searching_list                |
| 82  | FriendSearchFromId        | friend/search_by_id                  |
| 83  | FriendRequest             | friend/request                       |
| 84  | FriendRequestCancel       | friend/cancel_request                |
| 85  | FriendApproval            | friend/accept                        |
| 86  | FriendApprovalCancel      | friend/deny                          |
| 87  | FriendDelete              | friend/delete                        |
| 88  | FriendCommentList         | message/index                        |
| 89  | FriendCommentSend         | message/send                         |
| 90  | ItemUseRecovery           | item/use_item                        |
| 91  | AlbumLoad                 | album/index                          |
| 92  | ProfileLoad               | profile/get_profile                  |
| 93  | ProfileRename             | profile/rename                       |
| 94  | ProfileComment            | profile/update_comment               |
| 95  | ProfileUpdateSupporters   | profile/update_supporters            |
| 96  | PresentLoad               | present/index2                       |
| 97  | PresentReceive            | present/receive                      |
| 98  | PresentReceiveAll         | present/receive_all                  |
| 99  | PresentHistory            | present/history                      |
| 100 | ShopRecoverStamina        | shop/recover_stamina                 |
| 101 | ShopExpandBox             | shop/extend_box                      |
| 102 | ShopJewel                 | shop/buy_jewel_shop                  |
| 103 | ShopExtendCardStorage     | shop/extend_card_storage             |
| 104 | ExchangeShopList          | exchange_shop/exchange_list          |
| 105 | ExchangeShopItem          | exchange_shop/exchange_item          |
| 106 | DressShopList             | exchange_shop/exchange_list          |
| 107 | DressShopBuy              | exchange_shop/exchange_item          |
| 108 | PanelMissionGetReward     | panel_mission/get_reward             |
| 109 | MissionLoad               | mission_v2/index                     |
| 110 | MissionRewardExe          | mission_v2/get_reward                |
| 111 | EmblemEdit                | emblem/edit                          |
| 112 | ProducerRankTop           | producer_rank/index                  |
| 113 | ProducerRankRanking       | producer_rank/mo_p_ranking_list      |
| 114 | ProducerRankRankingInfo   | producer_rank/mo_p_rank_data         |
| 115 | AtaponTop                 | event/atapon/load                    |
| 116 | AtaponRanking             | event/atapon/ranking_list            |
| 117 | AtaponResult              | event/atapon/result_info             |
| 118 | CaravanTop                | event/caravan/load                   |
| 119 | CaravanExchange           | event/caravan/exchange               |
| 120 | MedleyTop                 | event/medley/load                    |
| 121 | MedleyRanking             | event/medley/ranking_list            |
| 122 | MedleyLiveLot             | event/medley/live_lot                |
| 123 | MedleyStart               | event/medley/start                   |
| 124 | MedleySetUnit             | event/medley/set_unit                |
| 125 | MedleyNext                | event/medley/next                    |
| 126 | MedleyLifeRecover         | event/medley/recover_life            |
| 127 | MedleyLiveBreak           | event/medley/live_break              |
| 128 | MedleyEnd                 | event/medley/end                     |
| 129 | PartyTop                  | event/party/load                     |
| 130 | PartyStart                | event/party/start                    |
| 131 | PartyMatchingExec         | event/party/matching_exec            |
| 132 | PartyResult               | event/party/result                   |
| 133 | PartyEnd                  | event/party/end                      |
| 134 | TourTop                   | event/tour/load                      |
| 135 | TourRanking               | event/tour/ranking_list              |
| 136 | TourStart                 | event/tour/start                     |
| 137 | TourNext                  | event/tour/load                      |
| 138 | TourLiveBreak             | event/tour/load                      |
| 139 | TourEnd                   | event/tour/end                       |
| 140 | TourTitleRename           | event/tour/rename_tour               |
| 141 | EventDeck                 | event/party/unit_edit                |
| 142 | GossipAddUserTips         | tips/add_user_tips                   |
| 143 | MigrationUpdate           | migration/index                      |
| 144 | OptionUpdate              | option/update_guerrilla_notification |
| 145 | ScEx                      | premium_sc/ex_exec                   |
| 146 | NameCardLoad              | name_card/index                      |
| 147 | NameCardUpdate            | name_card/update                     |
| 148 | NameCardOther             | name_card/other                      |
| 149 | NameCardOtherList         | name_card/other_list                 |
| 150 | NameCardApply             | name_card/apply                      |
| 151 | NameCardRequestList       | name_card/request_list               |
| 152 | NameCardAccept            | name_card/accept                     |
| 153 | NameCardDeny              | name_card/deny                       |
| 154 | NameCardFavorite          | name_card/set_favorite               |
| 155 | NameCardDistribute        | name_card/distribute                 |
| 156 | NameCardDelete            | name_card/other_delete               |
| 157 | NameCardSetting           | name_card/setting                    |
| 158 | NameCardFavoriteCancel    | name_card/set_favorite_cancel        |
| 159 | NameCardAllApply          | name_card/all_apply                  |
| 160 | LotteryTopLoad            | lottery/load                         |
| 161 | LotteryGetReward          | lottery/receive                      |
| 162 | MemoryCacheClear          | test/memcache_flush                  |
| 163 | ManageUserInfo            | debug/manage_user_info               |
| 164 | DeleteAllPresent          | debug/delete_all_present             |
| 165 | UpdateItemList            | debug/update_item_list               |
| 166 | AddCardList               | debug/add_card_list                  |
| 167 | DeleteAllAlbum            | debug/delete_all_album               |
| 168 | AddAllAlbum               | debug/add_all_album                  |
| 169 | DebugUserReset            | debug/reset_all_data                 |
| 170 | DebugIdolUpdate           | debug/update_card_list               |
| 171 | DebugAddPresent           | debug/send_present_random            |

# 参考文献和引用

[デレステ解析ノート](https://subdiox.github.io/deresute/)
