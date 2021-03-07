---
title: 科学的网络环境
date: 2019-12-12 10:02:37
tags: ['学习笔记']
---

上网要科学。

<!-- more -->

[v2ray 官网](https://www.v2fly.org)（需要科学的网络环境）

# 懒得折腾

[搬瓦工官方机场](https://justmysocks3.net)芜湖起飞，洛杉矶折扣后 5.5 刀一个月，速度还可以，不用担心封 IP，不过外服游戏会断线。日本的买不起。

# 自己动手

有 3.5 刀一个月的日本 VPS，但是有封 IP 的可能，封了 IP 要找商家更换。

1. CentOS 7 服务器

2. SSH 连接

    ```bash
    $ ssh <user>@<ip> [-p <port>]
    $ 输密码
    ```

3. [官方安装脚本](https://github.com/v2fly/fhs-install-v2ray)，敲命令就完事了

    ``` bash
    $ bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
    $ systemctl enable v2ray
    ```

4. 服务端配置文件示例

    放在 `/usr/local/etc/v2ray/config.json`

    ```json
    {
      "inbounds": [
        {
          "port": 23580,
          "protocol": "vmess",
          "settings": {
            "clients": [
              {
                "id": "{id}",
                "level": 1,
                "alterId": 64
              }
            ]
          }
        }
      ],
      "outbounds": [
        {
          "protocol": "freedom",
          "settings": {}
        },
        {
          "protocol": "blackhole",
          "settings": {},
          "tag": "blocked"
        }
      ],
      "routing": {
        "rules": [
          {
            "type": "field",
            "ip": ["geoip:private"],
            "outboundTag": "blocked"
          }
        ]
      }
    }
    ```

    [更多配置示例](https://github.com/v2fly/v2ray-examples)

    [文档](https://www.v2fly.org/config/overview.html)（需要科学的网络环境）

5. 管理服务

    ```bash
    # 启动
    $ systemctl start v2ray
    # 停止
    $ systemctl stop v2ray
    # 重启
    $ systemctl restart v2ray
    # 禁用
    $ systemctl disable [--now] v2ray
    # 查看状态
    $ systemctl status v2ray
    ```

# 客户端

[v2rayN](https://github.com/2dust/v2rayN) 3.x 有 PAC 模式。4.x 没有，要自己配置路由规则更麻烦一点。
