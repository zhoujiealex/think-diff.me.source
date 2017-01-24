---
title: CentOS6.x 升级Python2.7
permalink_url_parent: articles
permalink_url: upgrade-python2.7-centos-simplest-way
comments: true
tags:
  - Python
  - Linux
categories: Python
date: 2017-01-24 17:03:05
updated_time: 2017-01-24 17:03:05
keywords:
  - centos 升级 python 2.7
  - centos upgrade python 2.7
  - python 安装 pip
  - python 升级后 修复 yum
---

![logrtate log](/images/upgrade-python/linux-python-logo.jpg)

最近有些小伙伴还是被这个常见的需求坑到了，整理下屡试不爽的宇宙最简单的升级步骤，验证无数次。

# 最简单的安装步骤

CentOS6.x 最简单的升级Python2.7的方法：

``` bash
# Python 2.7.13:
wget https://www.python.org/ftp/python/2.7.13/Python-2.7.13.tar.xz --no-check-certificate
tar xf Python-2.7.13.tar.xz
cd Python-2.7.13
./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
make && make altinstall
```
DONE！ 新的 Python2.7 bin 位于： `/usr/local/bin/python2.7`。

<!-- more -->

## 编译参数说明


- `--enable-unicode=ucs4 --enable-shared` 这两个参数，凡是尝试安装过Oracle模块 `cx_Oracle` 都踩过这个坑，不解释。

- 使用`altinstall`，可以保留系统默认的python环境，不会做任何改动，如有需要请自行设置软连接。 


# 安装pip

方法一：

``` bash
wget https://bootstrap.pypa.io/get-pip.py --no-check-certificate
/usr/local/bin/python2.7 get-pip.py
```

国内网络不行，经常连不上，出现问题，请尝试用如下方法二。

方法二：
``` bash
wget https://bootstrap.pypa.io/ez_setup.py --no-check-certificate
sudo /usr/local/bin/python2.7 ez_setup.py
sudo /usr/local/bin/easy_install-2.7 pip
```

# 设置软连接

如有需要，可以**谨慎覆盖**掉系统默认的python： `/usr/bin/python`

``` bash
rm /usr/bin/python
ln -s /usr/local/bin/python2.7 /usr/bin/python
```

**恢复系统默认python2.6**：
``` bash
rm /usr/bin/python
ln -s /usr/bin/python2.6 /usr/bin/python
```

# 修复yum

如果覆盖了系统的自带python，则需要修复下yum（依赖默认的python2.6）。

编辑 `/usr/bin/yum`，修改第一行的 [Shebang](https://zh.wikipedia.org/wiki/Shebang) 为：
``` bash
//原始为：#!/usr/bin/python

#!/usr/bin/python2.6
```

# 参考链接

1. [Installing python 2.7 on centos 6.3. Follow this sequence exactly for centos machine only](https://github.com/h2oai/h2o-2/wiki/Installing-python-2.7-on-centos-6.3.-Follow-this-sequence-exactly-for-centos-machine-only)
2. [How to install Python 2.7 and Python 3.3 on CentOS 6](http://toomuchdata.com/2014/02/16/how-to-install-python-on-centos/)


