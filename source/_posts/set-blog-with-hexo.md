title: 用Hexo搭建博客（一）:如何搭建？
categories: hexo
tags:
- hexo
- TOI
date: 2014-10-27 14:57:30
updated: 2014-10-28 22:02:00
---

# 前言


[Hexo](http://hexo.io/)是来自台湾的[@tommy351](https://github.com/hexojs/hexo)写的一个轻量的静态博客框架，基于Node.js。
引用官网的介绍：
> 　A fast, simple & powerful blog framework, powered by Node.js.

我们可以利用该框架快速的生成静态博客站点，并用[markdown](http://daringfireball.net/projects/markdown/)语法书写文章。

类似的静态博客框架还有
- [Jekyll]()
- [Octopress]()

<!-- more -->

# Hexo 优点
 [这里](http://www.zhihu.com/question/21981094)有一篇在知乎的讨论：比较了很多博客框架的特点。
作者折腾过Jekyll：可直接以markdown在github上发布，github负责解析生成html，设置步骤较多。
Octpress本质上也是Jeklly的简化版。
Hexo上传的直接是生成好的html页面了，相比Jekyll会快一些。借用官网的介绍：
> - Blazing Fast：生成静态页面快速
> - Markdown Support：不仅支持原生语法，还支持Github的[GFM](https://help.github.com/articles/github-flavored-markdown/)语法
> - One-Command Deployment
> - Various [Plugins](https://github.com/tommy351/hexo/wiki/Plugins)


# How to install Hexo?
这里主要以Windows平台为例介绍。
## 安装Git
移步官网下载[Git](http://git-scm.com/downloads)，如已安装请跳过。
## 安装Node.js
[Node.js](http://nodejs.org/)基于Google V8 Javascript引擎，最新版本是v0.10.32。
下载安装文件，安装对应平台的[版本](http://nodejs.org/download/)。

## 安装Hexo
利用npm即可安装，在命令行里运行：
```bash
npm install -g hexo // -g 参数表示作为全局的 npm module 安装
```
当前最新hexo版本是2.8.3。

## 初始化blog框架
在打算书写博客的文件夹如`e:\HexoTest\`下，在命令行运行如下命令：
``` bash
hexo init
```
在生成blog框架后，**一定**要运行如下命令，会自动根据初始框架里面的配置，安装一些必备的依赖npm module：
``` bash
npm install
```
## 本地测试
框架中自带了一个`hello-world.md`的页面，在blog目录执行以下命令，并在浏览器中用`http://localhost:4000/`查看blog页面：
``` bash
hexo generate   // 生成静态页面，如 html, js, css 等
hexo server	 // 默认运行在 4000 port
```


## 发表文章
1.新建文章
``` bash
hexo n first-post  // 在source\_posts\下生成first-post.md文件
```
2.用markdown语法修改编辑后，用如下命令生成静态文件
``` bash
hexo g  // 等价于 hexo generate
```
3.发表文章
``` bash
hexo s  // 等价于 hexo server
```

# 部署到Github

## 设定域名
自定义的域名需要保证repository目录下有`CNAME`文件，其中内容填上域名如`think-diff.me`。
由于`.deploy\`文件夹每次都是重新生成，需要把CNAME文件放到hexo博客根目录下如`..\source\CNAME`。
参考[Github设置页面](https://help.github.com/articles/about-custom-domains-for-github-pages-sites/)。

顶级域名和subdomain
添加CNAME生效时间很长大概在48小时内。

# Tips
- 空格在markdown语法中默认过滤，可以用中文全角的空格代替，刚好两个汉字空格，参考[这里](http://www.zhihu.com/question/21420126)。
- 默认布局模板位于`..\scaffolds\`，默认使用post模板，可以加入常用的关键字如

``` bash
title: {{ title }}
date: {{ date }}
tags:
categories:
description：
```

- `hexo clean` 可用来清理已经生成的public静态文件，如有些效果修改后没生效，可以先清理再启动server。

``` bash
hexo clean && hexo g && hexo s -p 8888 --debug
```


# 参考
1. [使用 Hexo 搭建静态博客](http://inching.org/2014/03/22/hexo/)
2. [hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)
3. [hexo系列](http://zipperary.com/categories/hexo/)

