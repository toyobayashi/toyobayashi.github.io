---
title: 在 WSL Ubuntu 下搭建 Git 远程仓库
date: 2019-12-23 20:43:12
tags: ['学习笔记']
---

可在局域网内玩耍。

<!-- more -->

# 安装 Windows Subsystem for Linux

略。

# 服务器配置

## 重新配置 openssh-server

``` bash
$ sudo dpkg-reconfigure openssh-server
```

## 编辑 sshd 配置文件

``` bash
$ sudo vi /etc/ssh/sshd_config
```

放开注释，端口可改可不改，默认 22。

```
#Port 22
PubkeyAuthentication yes
AuthorizedKeysFile     .ssh/authorized_keys
```

`:wq` 保存并退出。

重启 ssh 服务

``` bash
$ sudo service ssh restart
```

## 创建 git 用户

``` bash
$ sudo adduser git
$ su git
$ cd
$ mkdir .ssh && chmod 700 .ssh
$ touch .ssh/authorized_keys && chmod 600 .ssh/authorized_keys
```

## 客户端生成 ssh key

``` bash
$ mkdir ~/.ssh
$ cd ~/.ssh
$ ssh-keygen -t rsa -C "email@example.com" [-f <keyname>]
```

如果有多个密钥，加入密钥链再写一下 `~/.ssh/config` 配置文件

``` bash
$ ssh-agent bash
$ ssh-add ~/.ssh/<keyname>
$ touch ~/.ssh/config
```

```
Host <ip>
    Port 22
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/<keyname>
```

## 服务端添加 ssh 公钥

``` bash
$ cat /mnt/c/Users/<username>/.ssh/<keyname>.pub >> ~/.ssh/authorized_keys
```

# 搭建远程仓库

## 服务端创建裸仓库

``` bash
$ cd
$ mkdir -p ./repo/test.git
$ git init --bare ./repo/test.git
```

## 客户端关联仓库

可以直接拉代码。

``` bash
$ git clone ssh://git@<ip>[:<port>]/home/git/repo/test.git
```

也可以在现有仓库中添加远程仓库地址。

``` bash
$ git remote add myremote ssh://git@<ip>[:<port>]/home/git/repo/test.git
```

## Git for windows 汉化

```sh
curl -o zh_CN.po https://raw.githubusercontent.com/git-for-windows/git/v2.31.0.windows.1/po/zh_CN.po
msgfmt --check -o git.mo ./zh_CN.po
mkdirp <git_install_dir>/mingw64/share/locale/zh_CN/LC_MESSAGES
mv ./git.mo <git_install_dir>/mingw64/share/locale/zh_CN/LC_MESSAGES/
```
