---
title: 日志集中化收集（二）：logrotate 配置
permalink_url_parent: articles
permalink_url: center-log-with-logrotate
comments: true
tags:
  - Linux
  - DevOps
  - Log
categories: 运维
date: 2017-01-03 21:55:03
updated_time: 2017-01-03 21:55:03
keywords:
  - 日志集中化
  - log collect
  - 日志收集
  - logrotate config
  - logrotated
  - logrotate 配置
  - 日志轮询
  - crontab
  - 定时任务
  - anacrontab
---

![logrtate log](/images/center-log/paille-truck.jpg)

本文接上篇  [日志集中化收集（一）：rsyslog 配置](/articles/center-log-with-rsyslog/)。
用 rsyslog 收集了上百G的日志后，得用另一个Linux自带的脚本：
 `/usr/sbin/logrotate` 自动的压缩，分割，归档好历史日志。

<!-- more -->

# logrotate 简介

logrotate （[GitHub 地址](https://github.com/logrotate/logrotate)） 诞生于 [1996/11/19](https://github.com/logrotate/logrotate/releases) ，当前（2017/01/03）最新版本 [3.11.0](https://github.com/logrotate/logrotate/releases/tag/3.11.0)。

> logrotate - rotates, compresses, and mails system logs

测试机器 CentOS 6.8 Final， 系统自带的版本为 3.7.8： 

``` bash
$ logrotate -v
logrotate 3.7.8 - Copyright (C) 1995-2001 Red Hat, Inc.   

Usage: logrotate [-dfv?] [-d|--debug] [-f|--force] [-m|--mail command] [-s|--state statefile] [-v|--verbose] [-?|--help] [--usage] [OPTION...] <configfile>
```
最新版本，需要自行下载源码编译安装。

## 配置文件
执行文件： `/usr/sbin/logrotate`
主配置文件: `/etc/logrotate.conf`
自定义配置文件: `/etc/logrotate.d/*.conf`

修改配置文件后，并不需要重启服务。
由于 logrotate 实际上只是一个可执行文件，不是以daemon运行。

`/etc/logrotate.conf` - 顶层主配置文件，通过 `include` 指令，会引入 `/etc/logrotate.d` 下的配置文件  
``` bash
# see "man logrotate" for details
weekly
rotate 4
# create new (empty) log files after rotating old ones
create
# use date as a suffix of the rotated file
dateext
# uncomment this if you want your log files compressed
#compress
# RPM packages drop log rotation information into this directory
include /etc/logrotate.d
# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp
        minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
# system-specific logs may be also be configured here.

```

`/etc/logrotate.d/` - 通常一些第三方软件包，会把自己私有的配置文件，也放到这个目录下。 如 yum， zabbix-agent，syslog 等。 

``` bash
$ cat /etc/logrotate.d/yum
/var/log/yum.log {
    missingok
    notifempty
    size 30k
    yearly
    create 0600 root root
}
```
## 运行 logrotate

**crontab**定时：
通常惯用的做法是配合 `crontab` 来定时调用。
``` bash
$ crontab -e

*/30 * * * * /usr/sbin/logrotate /etc/logrotate.d/rsyslog > /dev/null 2>&1 &
```

在调试自定义配置的时候，我们需要手动运行，来确保是按照我们所需运行的。

**手动运行**：
debug 模式： 指定 `[-d|--debug]`
``` bash
logrotate -d <configfile>
```

并不会真正进行 rotate 或者 compress 操作，但是会打印出整个执行的流程，和调用的脚本等详细信息。

verbose 模式： 指定 `[-v|--verbose]`
``` bash
logrotate -v <configfile>
```
会真正执行操作，打印出详细信息（debug模式，默认是开启verbose）

**系统自带** cron task： `/etc/cron.daily/logrotate`，每天运行一次。

``` bash
$ cat /etc/cron.daily/logrotate
#!/bin/sh

/usr/sbin/logrotate /etc/logrotate.conf >/dev/null 2>&1
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0

```


# logrotate 参数

详细介绍请自行 `man logrotate`， 或者[在线 man page](https://linux.die.net/man/8/logrotate)。

主要介绍下完成常用需求会用到的一些参数。

一个典型的配置文件如下：
``` bash
/Data/logs/production/*/*/*.log
/Data/logs/erp/*/*/*.log
/Data/logs/erp/mq_order/*/*/*.log
{
   prerotate
		# ....
   endscript

   #daily
   rotate 10
   size 5M
   create 0644 karltest karltest
   dateformat  -%Y%m%d-%s
   compress
   missingok

   postrotate
     /bin/kill -HUP $(/bin/cat /var/run/syslogd.pid 2>/dev/null) &>/dev/null
   endscript
}
```

第一部分是匹配的文件pattern，可以是通配符，**注意：如果对应的log不存在会报错，中断处理**，可以自行用 debug 模式测试。（可以添加 `missingok` 缓解）

`{ ... }` 花括号里面的就是具体的指令参数了， logrotate 支持一些hook预处理，可以在rotate执行之前或者之后调用命令或者自己的脚本。

最常见的需求：
- 限制大小： size 1k (如 5M， 2G)
- 压缩： compress， 默认gzip，后缀为gz。  也可以指定其他压缩程序，如bzip2，后缀名也可以修改。
- `create <user> <group>`
- 保留个数： rotate <num>
- dateformat: rotate的文件后缀格式
- postrotate： 这个是最常用的，用来 reopen 被rotate后的文件，详见下文 [Trouble Shooting](#reopen)。
- 其余hook:
	- prerotate/endscript
	- firstaction/endscript
	- lastaction/endscript
	- preremove/endscript
	- sharedscripts

# 日志集中化的配置

介绍完基础知识后，回归到日志集中化收集的任务上来。

回顾下，最终 rsyslog 收集组成的文件夹结构：

``` bash
# tree -I "*gz|*log" /Data/logs/
/Data/logs/
├── gateway
│   ├── 172.31.70.18
│   │   └── archived
│   ├── 172.31.70.19
│   │   └── archived
│   ├── 172.31.70.195
│   │   └── archived
│   ├── 172.31.70.197
│   │   └── archived
│   ├── 172.31.70.198
│   │   └── archived
│   └── 172.31.70.20
│       └── archived
├── product
│   ├── 172.31.70.118
│   │   └── archived
│   ├── 172.31.70.119
│   │   └── archived
│   ├── 172.31.70.23
│   │   └── archived
│   └── 172.31.70.24
│       └── archived
...

# du -sh /Data/logs
271G    /Data/logs
```

需求：
1. 大小超过1G压缩
2. 每天归档 *.gz 的压缩包到 `archived`目录下，也就是说当前目录只保留当天的日志
3. 保留一个月的备份

这些需求，单独的logrotate并不能完成，所以我写了个 shell 脚本，来完成额外的功能。

## logrotate 配置

监控多个目录，压缩文件，移至目录 `olddir archived`。

```bash
$ cat /etc/logrotate.d/karltest-custom-conf/logrotate-karltest-log-quick-run.conf
/Data/logs/karltest/*/*/*.log
/Data/logs/erp/*/*/*.log
/Data/logs/erp/mq_order/*/*/*.log
{
   rotate 32
   size 1024M					# 对于 *-daily-run.conf，设置了较小阈值 5M
   create 0644 karltest karltest
   #dateformat  -%Y%m%d-%s			# 时间戳，已经由rsyslog产生了，这里不需要
   compress
   missingok
   olddir archived				# 归档目录
   postrotate					# 重启 rsyslog，让其 reopen 新的同名文件
     /bin/kill -HUP $(/bin/cat /var/run/syslogd.pid 2>/dev/null) &>/dev/null
   endscript
}
```
实际上，我准备了两个 conf 文件： `*-daily-run.conf` 和 `*-quick-run.conf`，唯一的区别就是 **size 阈值**。
希望达成这样的效果：
- 较大阈值的配置文件，运行的更频繁些，防止磁盘爆掉。
- 较小阈值的配置文件，更多的是整理的作用，每天00：01分运行一次，来移动前一天的所有日志（即使很小），归档到archived目录

## crontab 设置

*这里我是用 Ansible 来配置的logcenter 机器上的的定时任务。*

``` bash
$ crontab -l
#Ansible: For server: Daily run - logrotate *.log under /Data/logs
1 0 * * * /bin/sh /Data/logs/.run_logrotate.sh create /etc/logrotate.d/karltest-custom-conf/logrotate-karltest-log-daily.conf -v >> /Data/logs/karltest_log_rotate_history.log 2>&1 &
#Ansible: For server: Quick run - logrotate *.log under /Data/logs
23 6,12,18 * * * /bin/sh /Data/logs/.run_logrotate.sh create /etc/logrotate.d/karltest-custom-conf/logrotate-karltest-log-quick-run.conf -v >> /Data/logs/karltest_log_rotate_history.log 2>&1 &
#Ansible: For server: older than file under /Data/logs
28 4 * * *  /usr/bin/find /Data/logs -type f -mtime +30 -delete >> /Data/logs/karltest_log_rotate_history.log 2>&1 &
```
说明：
- 每天 00:01 运行较小阈值的配置文件
- 每天 6/12/18:23 运行较大阈值的配置文件
- 每天 4:28 执行一次清理工作：删除最近30天（`-mtime +30`）没有修改的文件

## 自定义的脚本

完整文件在这里 [run_logrotate.sh](/files/center-log/run_logrotate.sh.txt)。

主要功能：

- `create_folder` - 创建archived folder， 因为 logrotate 的 `olddir` 指向的目录，必须存在
- `moveold` - 移动最近一天没有修改的文件，到 `olddir` 指向的目录，主要和`daily.conf` 互补， 因为 小于5M的并不会被压缩，对于某些 error.log经常只有10k。
- `clean_dummy_file` - 删除大小为0的文件，因为 在每天的0点， rsyslog会向新的日期文件里写日志，此时，由于logrotate的作用，会多出一个大小为0的空旧日期的文件。

**其实这个脚本才是我花工作量最大的地方**。上面的功能点，也是我调试的时候遇到的一些坑。

# Trouble Shooting

## 自定义的log压缩，每天多运行一次

**现象**：在每天03:47时候，多运行了一次，但是 `crontab -l` 里，并没有配置。
**原因**：系统自带的 `/etc/cron.daily/logrotate`， 每天会自己运行一次。
**解决**：把自定义的配置文件，放到别的目录，或者 移到下层目录： `/etc/logrotat.d/karltest_conf/*.conf` 

**多问一句**： 为啥时间是03：47呢，有时候随机是03：xx别的时间，而不是每天的00:00呢？
详情参考：[When does `cron.daily` run?](http://serverfault.com/questions/135906/when-does-cron-daily-run) 和 `man anacrontab`。
``` bash 
$ cat /etc/anacrontab
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
1       5       cron.daily              nice run-parts /etc/cron.daily
7       25      cron.weekly             nice run-parts /etc/cron.weekly
@monthly 45     cron.monthly            nice run-parts /etc/cron.monthly
```
简单来说：大多数Linux发行版本，除了有 `crontab`，还有 `anacron`，它的使用场景，适用于服务器不能7x24运行，但是有些定时任务又需要运行。 如果在任务还没到执行时间的时候，服务器关机了，那么配置 `anacron`，下次开机特定时间后（如上面的5分钟运行 cron.daily, 25分钟运行cron.weekly）。
配合 RANDOM_DELAY 和 START_HOURS_RANGE 来控制随机的时间。

这个功能就类似Windows上的计划任务有个设置：*如果错过执行时间，下次开机延迟多久后立刻启动*。

**解释03:xx运行时刻**：
由于我们的服务器是7x24小时，所以不会错过执行时间。默认的设置 `START_HOURS_RANGE=3-22`，所以是在3点，
配合第二列的延迟5分钟，加上随机的45分钟：
所以**最终的 `cron.daily` 的执行时间为： `3:05 ~ 3:50`**， 符合几天的观测结果。

## 日志rotate之后，如何reopen，继续在新文件里的写log？<a name='reopen'></a>

主要用到的就是 `postrotate` 这个hook了，由于logrotate之后，即使已经移走了，但是rsyslog还是持有这个文件操作句柄，会继续往 *log.1.gz 里写，所以需要 restart rsyslog 来 reopen 下 logrotate新创建的同名文件。

``` bash
   postrotate
     /bin/kill -HUP $(/bin/cat /var/run/syslogd.pid 2>/dev/null) &>/dev/null
   endscript
```
由于rsyslog的发送端，有本地队列缓存，所以新产生的日志并不会丢失，接收端的rsyslog可以放心的重启。

那么对于那些不能中断的服务的日志，怎么解决呢，然后重新打开日志文件？
logrotate 提供了 `copytruncate`, 但是会有一定的时间差，丢失部分的日志数据。
对于哪些写日志比较频繁的，如debug级别的，更有可能丢失。
对于error级别的，应该问题不大。
**copytruncate 原理**： 
默认的指令 `create` 做法，是 移动旧文件，创建新文件，然后用脚本reopen新文件。
而 copytruncate 是采用的先拷贝再清空， 先复制一份旧的日志，然后请客原文件，整个过程原来的文件句柄，并没有变化，所以不需要reopen，服务可以不中断。

``` bash
copytruncate
	Truncate the original log file in place after creating a copy, instead of moving the 
	old log file and optionally creating a new one. It can be used when some program 
	cannot be told to close its logfile and thus might continue writing (appending) 
	to the previous log file forever. **Note that there is a very small time slice between 
	copying the file and truncating it, so some logging data might be lost**. When this
	option is used, the create option will have no effect, as the old log file stays in place.
```

另一个解决思路， 引用自 [被遗忘的Logrotate](http://huoding.com/2013/04/21/246)：

> 熟悉Apache的朋友可能会记得cronolog，不过Nginx并不支持它，有人通过 **mkfifo** 命令曲线救国，先给日志文件创建管道，再搭配cronolog轮转，虽然理论上没有问题，但效率上有折扣。另外，Debian/Ubuntu下有一个简化版工具savelog，有兴趣可以看看

参考 [nginx startup script with cronolog](https://gist.github.com/ntakaaki/2891895)

## 通配符 *，missingok 和 olddir 不能同时使用

这个感觉还是logrotate的bug，参考，最新版本仍然存在。参考 [Logrotate wildcard fails on missing files with “missingok” AND “olddir”](http://superuser.com/questions/1059033/logrotate-wildcard-fails-on-missing-files-with-missingok-and-olddir)。
``` bash
# If there's no any log under that directory, it will complain:

error verifying log file path /Data/logs/*/*: No such file or directory
```
- 使用missingok 和 通配符 * 就没问题， 这也是 missingok 字面上的作用。
- 使用olddir 和 通配符* 也没有问题，本来也是有问题，[最新版刚Fix掉了](https://github.com/logrotate/logrotate/commit/73493ec38c5e806fa66d8c3f13259775da6282d9)。

> Fix 'olddir' usage with wildcard in the middle of path in the pattern…
… definition when the pattern did not match any log file.


但是三个合在一起就出问题了， `missingok + olddir + *` ==> Boom... `:(`

**复现**：在监控的目录pattern（如 `/var/logs/*/*.log`）下的一个文件夹( `/var/log/10.1.2.3.4/`)里，保证为空文件夹，不存在 `*.log` 日志文件
此时用 `logrotate -d xxx.conf` 就会报错。

**目前解决不了**。
好在我们的机器够多，日志文件每时每刻都产生，不存在空文件夹，只有刚刚配置logcenter收集服务器的时候，刚启动一段时间有这个可能出现错误，稳定运行后，就不会出现了。

# 参考链接
1. [logrotate GitHub 项目地址](https://github.com/logrotate/logrotate)
2. [logrotate man page](https://linux.die.net/man/8/logrotate)
3. [HowTo: The Ultimate Logrotate Command Tutorial with 10 Examples](http://www.thegeekstuff.com/2010/07/logrotate-examples/)
4. [被遗忘的Logrotate](http://huoding.com/2013/04/21/246)
5. [When does `cron.daily` run?](http://serverfault.com/questions/135906/when-does-cron-daily-run)
6. [Cron Vs Anacron: How to Setup Anacron on Linux](http://www.thegeekstuff.com/2011/05/anacron-examples/)
7. [Why does my CentOS logrotate run at random times?](http://serverfault.com/questions/454118/why-does-my-centos-logrotate-run-at-random-times)
8. [Logrotate wildcard fails on missing files with “missingok” AND “olddir”](http://superuser.com/questions/1059033/logrotate-wildcard-fails-on-missing-files-with-missingok-and-olddir)
9. [nginx startup script with cronolog](https://gist.github.com/ntakaaki/2891895)
