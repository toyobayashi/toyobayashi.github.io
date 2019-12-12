---
title: 在 CentOS 7 部署 SS 
date: 2019-12-12 10:02:37
tags: ['学习笔记']
---

自己动手丰衣足食。

<!-- more -->

1. 买服务器
2. Xshell SSH 连接
3. 有大佬已经写好了脚本，敲命令就完事了

    ``` bash
    $ wget --no-check-certificate -O shadowsocks.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
    $ chmod +x shadowsocks.sh
    $ ./shadowsocks.sh 2>&1 | tee shadowsocks.log
    ```
