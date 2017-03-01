---
layout: post
title:  "MacOs快速搭建开发环境-python"
date:   2016-12-22 12:42:00 +0800
category: 安装文档
tagline: "Supporting tagline"
tags: [macbook, mac os, 环境搭建, python]
---

本篇介绍在MacOS上从头开始搭建一个python程序员使用的开发环境，包括IDE，依赖库等。

# 1. 前置

MacOS系统初始化，AppleID账户，网络设置等在此略过。

# 2. 软件安装

### 2.1 Xcode

App Store中安装即可

### 2.2 Mac Office软件

略

### 2.3 brew

官网下载安装

### 2.4 wget

`$ brew install wget`

# 3. IDE及依赖

### 3.1 JDK

首先需要安装一个jdk，去官网下载安装即可。

### 3.2 Eclipse

下载安装即可

##### 3.2.1 安装pydev

from Eclipse Marketplace

##### 3.2.2 安装easyshell

from Eclipse Marketplace


### 3.3 python

如果Macos自带的python，将会在更新某些依赖时遇到权限问题，使用sudo也无法解决。

所以我们不使用默认python

`$ brew install python`


##### 3.3.1 pip

> https://pip.readthedocs.io/en/stable/installing/

`$ wget https://bootstrap.pypa.io/get-pip.py`

`$ sudo python get-pip.py`

##### 3.3.2 virtualenv

* 安装virtualenv

`$ pip install virtualenv`

* 安装virtualenvwrapper

`$ pip install virtualenvwrapper`

* 创建防止virtualenv的目录

`$ mkdir ~/.virtualenvs`

* 设置wrapper在每次打开shell时自动运行

`$ source /usr/local/bin/virtualenvwrapper.sh`

> 重启终端之后即可

* 创建一个virtualenv

`$ mkvirtualenv <name>`

> virtualenv将会被安装到 ~/.virtualenvs目录下

* 切换

`$ workon`可以查看当前创建的virtualenvs

`$ workon <name>`可以切换环境

* 退出

`$ deactivate`

* 删除

`$ rmvirtualenv`

##### 3.3.3 在Eclipse中使用virtualenv

在Preferences -> PyDev -> Interpreters -> Python Interpreter界面 点击 New

可执行文件选择： ~/.virtualenvs/<name>/bin/python

### 3.4 安装nodejs/npm

`$ brew install node --with-full-icu`

### 3.5 安装git

`$ brew install git`

##### 3.5.1 安装Commitizen

Commitizen 是一个撰写合格 commit message 的工具。

```
$ npm install -g commitizen
$ npm i -g cz-conventional-changelog
$ echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```

> 凡是用到 git commit 命令，一律改为使用 git cz。这时，就会出现选项，用来生成符合格式的 commit message

### 3.6 安装SourceTree

### 3.7 安装mysql

主要是为了mysql的CLI

`$ brew install mysql`

启动

`$ mysql.server start`

使用brew启动

`$ brew services start mysql`

mysql默认用户名`root`, 密码为空


### 3.8 安装mongodb

`$ brew install mongo`

### 3.9 安装redis

`$ brew install redis`

### 3.10 安装SequelPro

一款mac上比较好用的mysql可视化工具

### 3.11 安装postgresql

`$ brew install postgresql`

### 3.12 docker

从docker官网上下载docker for mac

> https://docs.docker.com/docker-for-mac/


# 4. Python常用库

```

$ pip install mysql-python

$ pip install psycopg2
```

# 5. Mac设置

### 5.1 Finder设置隐藏文件可见

```bash

defaults write com.apple.finder AppleShowAllFiles Yes && killall Finder

```

### 5.2 ls颜色

vim ~/.bash_profile

```
export CLICOLOR=1
export LSCOLORS=exfxhxhxgxhxhxgxgxbxbx
```

> http://blog.sina.com.cn/s/blog_a046022d0102v6fi.html

### 5.3 vim

##### 5.3.1 vim颜色设置

> http://www.cnblogs.com/puyangsky/p/5442153.html

###### 5.3.2 打开文本时光标自动定位到上次打开位置

待完成
