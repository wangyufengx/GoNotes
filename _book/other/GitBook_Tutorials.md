# Gitbook安装使用教程

## 本地Gitbook安装

### 安装NodeJs

gitbook是一个基于Node.js的命令行工具，所以要先安装Node.js。高版本的NodeJs（17.0.1）安装GitBook出现问题，貌似是版本不兼容。NodeJs地址：https://nodejs.org/en/

NodeJs都会默认安装npm(包管理工具),所以不需要单独安装npm。

以下是我安装成功的nodeJs版本信息：

```bash
node -v
v12.16.3
npm -v
6.14.4
```

### 安装Gitbook

打开命令行执行如下命令安装gitbook:

```
npm install -g gitbook-cli
```

检查是否安装成功：

```bash
gitbook -V
CLI version: 2.3.2
GitBook version: 3.2.3
```

### Gitbook的使用

#### 初始化

创建文件夹（如test）并进入文件夹内，执行以下命令：

```
gitbook init
```

生成如下文件：

```
README.md
SUMMARY.md
```

#### 文件目录定义

笔记的目录结构是在`SUMMARY.md`中定义的。

例如：

```
# Summary

* [Introduction](README.md)

* [part1](part1/README.md)

	* [概述](part1/概述.md)
	
* [part2](part2/README.md)

	* [概述](part2/概述.md)
```

目录结构如下：

```
├── book.json	#引入插件时创建
├── README.md
├── SUMMARY.md	#目录结构
├── part1/
|   ├── README.md
|   └── 概述.md
└── part2/
    ├── README.md
    └── 概述.md
```

#### 构建

使用下面命令构建笔记

```
gitbook build
```

#### 启动服务

使用下面命令启动笔记，本地访问http://localhost:4000查看书籍。

```
gitbook serve
```

## 创建Github仓库

在github上创建一个仓库，将文件夹下的内容上传到github仓库。

## 云端Gitbook

点击`new space`创建一个书籍。

在最右上角的选择启用github同步并进行相关配置。需要身份验证并且会将gitbook安装到您的github上（可选择一个仓库还是所有仓库都安装gitbook）。

官方说明如下：

https://docs.gitbook.com/integrations/git-sync/enabling-github-sync

此时你的电子书已搭建完成。可点击`Publish`可公开笔记并且可在`get the link`输入框下看到笔记的网址。

### 参考文档

https://docs.gitbook.com/integrations/git-sync/enabling-github-sync
