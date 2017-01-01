---
title: 日志集中化收集（一）：rsyslog 配置
permalink_url_parent: articles
permalink_url: center-log-with-rsyslog
comments: true
tags:
  - Linux
  - DevOps
  - Log
categories: 运维
date: 2016-12-18 16:55:09
updated_time: 2016-12-18 16:55:09
keywords:
  - 日志集中化
  - log collect
  - 日志收集
  - rsyslog config
  - rsyslog配置
---


![collect log](/images/center-log/collect-trash.jpg)


最近遇到一个需求，需要把线上环境的debug日志及集中化收集起来，一方面是方便开发调试；一方面是避免直接到线上环境查看，存在安全隐患。

常用可选方案：

1. rsyslog发送端 + rsyslog接收端： 直接存在接收端的本地硬盘
2. rsyslog发送端 + logstash接收端 + <后续第三方处理>： 接受到log更新行后，通过logstash简单处理后，可以继续往第三方处理，如放到ElasticSearch，或者放到消息队列Kfaka等
3. rsyslog发送端 + Splunk： Splunk是商业软件，也是业内用的比较多的方式，价格不菲 

基本原理和处理流程都是类似的： 监控本地log文件内容的变化，然后把变化的文件内容发送到远端收集服务上。
例如常说的[ELKstack](https://www.gitbook.com/book/chenryn/elk-stack-guide-cn/details)（ElasticSearch+Logstash+Kibana）的第一步都是配置rsyslog发送端。

不管哪种方案都得监控本地文件的变化，rsyslog属于必选。我们需求比较简单，暂时选用了落地到本地盘，默认存储15天debug日志。

本文主要介绍rsyslog发送端、接收端的配置，以及遇到的一些坑。

*友情提醒，文章很长，请自备小板凳。*

<!-- more -->

# rsyslog简介

[rsyslog](http://www.rsyslog.com/) 在Linux上自带，兼容syslog语法，在syslog基础上增加了更多协议的支持，配合额外module插件可以完成很多场景的使用。借用下官网的图片：

![rsyslog-features-imagemap](/images/center-log/rsyslog-features-imagemap.png)

注： Windows 平台需要 [nxlog](https://nxlog.co/) （nxlog 是用C 语言写的一个跨平台日志收集处理软件）。

## 安装

在CentOS 6.8 Final 上自带的版本为 5.8.10。如需最新版本，可[参考官网](http://www.rsyslog.com/doc/v8-stable/installation/)。

``` bash
$ rsyslogd -version
rsyslogd 5.8.10, compiled with:
        FEATURE_REGEXP:                         Yes
        FEATURE_LARGEFILE:                      No
        GSSAPI Kerberos 5 support:              Yes
        FEATURE_DEBUG (debug build, slow code): No
        32bit Atomic operations supported:      Yes
        64bit Atomic operations supported:      Yes
        Runtime Instrumentation (slow code):    No

See http://www.rsyslog.com for more information.
```

V5版本开发于2010年，属于比较旧的版本，最新版本是V8，支持了更多的字符串处理函数和更多module，当然性能也更好。
缺点是：新旧配置语法不兼容，而采用内置版本的另一个偷懒的好处是云端的镜像也不需要再额外升级，能支持更多老机器。

后面介绍以V5版本为例，如有不同的，会单独指出。

## 配置文件介绍

主配置文件: `/etc/rsyslog.conf`
自定义配置文件: `/etc/rsyslog.d/*.conf`
修改配置文件后，重启服务： `sudo /etc/init.d/rsyslog restart`

一份配置文件主要包括以下几个部分：

1. MODULES
2. RULES
3. 全局指令，模板，模块参数等


自带的配置文件如下，参考后面的注释：

```
# rsyslog v5 configuration file

# For more information see /usr/share/doc/rsyslog-*/rsyslog_conf.html
# If you experience problems, see http://www.rsyslog.com/doc/troubleshoot.html

#### MODULES ####	               # 模块放在开头加载

$ModLoad imuxsock # provides support for local system logging (e.g. via logger command) # 可以用来调试，稍后有例子
$ModLoad imklog   # provides kernel logging support (previously done by rklogd)
#$ModLoad immark  # provides --MARK-- message capability

# Provides UDP syslog reception
#$ModLoad imudp
#$UDPServerRun 514

# Provides TCP syslog reception	   # TCP server，接收端需要加载这个模块，发送端不需要
#$ModLoad imtcp
#$InputTCPServerRun 514


#### GLOBAL DIRECTIVES ####

# Use default timestamp format
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat  # 消息格式

# File syncing capability is disabled by default. This feature is usually not required,
# not useful and an extreme performance hit
#$ActionFileEnableSync on

#### RULES ####				       # 用来指定哪种类型的，哪种级别的log，发送给谁处理

# Log all kernel messages to the console.
# Logging much else clutters up the screen.
# kern.*                                                 /dev/console

# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;authpriv.none;cron.none                /var/log/messages

# The authpriv file has restricted access.
authpriv.*                                              /var/log/secure

# Log all the mail messages in one place.
mail.*                                                  -/var/log/maillog    # '-' 表示异步


# Log cron stuff
cron.*                                                  /var/log/cron

# Everybody gets emergency messages
*.emerg                                                 *

# Save news errors of level crit and higher in a special file.
uucp,news.crit                                          /var/log/spooler

# Save boot messages also to boot.log
local7.*                                                /var/log/boot.log	# local开头的是自定义的日志类型


# ### begin forwarding rule ###
# The statement between the begin ... end define a SINGLE forwarding
# rule. They belong together, do NOT split them. If you create multiple
# forwarding rules, duplicate the whole block!
# Remote Logging (we use TCP for reliable delivery)
#
# An on-disk queue is created for this action. If the remote host is
# down, messages are spooled to disk and sent when it is up again.
#$WorkDirectory /var/lib/rsyslog # where to place spool files
#$ActionQueueFileName fwdRule1 # unique name prefix for spool files
#$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
#$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
#$ActionQueueType LinkedList   # run asynchronously
#$ActionResumeRetryCount -1    # infinite retries if host is down
# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
#*.* @@remote-host:514
# ### end of the forwarding rule ###

# Finally include all config files in /etc/rsyslog.d. This allows overrides
# of the default configuration above.
$IncludeConfig /etc/rsyslog.d/*.conf			# 这里会自动加载自定义的*.conf配置文件，可以覆盖默认参数	
```


# 模块 imfile

为了完成我们的任务，除了上面默认的模块，还需要加载 `imfile`，,来指定监控哪些文件，[参考文档](http://www.rsyslog.com/doc/v5-stable/configuration/modules/imfile.html)。

``` bash
$ModLoad imfile  # Load the imfile input module

```
该模块把标准的文本文件转换成syslog的message格式， 所谓标准文本是指：保护可打印的字符，每行以 `LF` 作为分隔符号。 支持文件正在在logrotate的时候，仍能正确处理。
它会把监控文件的读取到哪一个位置（类似游标cursor），存储在state文件里（由 `$WorkDirectory` 指定）。

该模块支持如下指令，一组如下设置，可以称为一个 **listener**：

``` bash
$InputFileName /path/to/file   # 待监控的文件路径

$InputFileTag tag	# 文件唯一标识tag，最好保持唯一，用于接收端区分原始log文件，可以包含特殊字符，如":"、","等

$InputFileStateFile /path/to/state/file
# 【重要】需要保证发送端唯一，记录读取到哪儿，状态文件保存在$WorkDirectory，默认为 /var/lib/rsyslog
#  如果某个要监控的文件名变化了，一定要重新设置该值

$InputFileFacility facility  # log类型，默认local0， local开头的表示自定义类型

$InputFileSeverity severity	 # log级别：info，warning，默认notice

$InputRunFileMonitor		 # 启动监控当前的文件，如果忘记这行，则啥事也不会发生

$InputFilePollInterval seconds	# 全局设置，默认轮询是10s

$InputFilePersistStateInterval lines  # 每多少行更新state文件状态

$InputFileReadMode mode   # 官网竟然没这个解释，不过也没用到。。。

$InputFileMaxLinesAtOnce number	
# 默认10240，如果在发送端，需要同时监控多个文件，会处理完当前文件特定行后，切换到下一个文件，避免一个文件一直占用处理，导致收集别的文件不及时。

$InputFileBindRuleset ruleset  # 属于较高级的设置，可以把这个listener绑定到特点的规则(http://www.rsyslog.com/doc/v5-stable/concepts/multi_ruleset.html)
```

接收端配置，**注意tag里的逗号 `','`**，稍后在接收端，会它来分隔：
``` bash
# For product, total 19 files.

$InputFileName  /Data/logs/product/cache-Statistic.log
$InputFileTag   product,cache-Statistic	
$InputFileSeverity info
$InputFileStateFile state_product_cache-Statistic
$InputFilePersistStateInterval 25000
$InputFileFacility local5
$InputRunFileMonitor

```

# Rule设置

一条rule的语法格式如： `<Facility>.<Severity>  <Target>`
例如：
```
# Log cron stuff
cron.*         /var/log/cron

# 记录info到本地messages文件，.none 结尾表示排除掉这些文件类型。
*.info;mail.none;authpriv.none;cron.none    /var/log/messages   

# 所有为local5的任意级别日志发送到远端514
local5.* @@192.168.56.10:514	
```

## Facility

日志设备(可以理解为日志类型):
``` bash
auth         #pam产生的日志，认证日志
authpriv     #ssh,ftp等登录信息的验证信息，认证授权认证
cron         #时间任务相关
kern         #内核
lpr          #打印
mail         #邮件
mark(syslog) #rsyslog服务内部的信息,时间标识
news         #新闻组
user         #用户程序产生的相关信息
uucp         #unix to unix copy, unix主机之间相关的通讯
local 1~7    #自定义的日志设备
```

## Severity

``` bash
debug           #有调式信息的，日志信息最多
info            #一般信息的日志，最常用
notice          #最具有重要性的普通条件的信息
warning, warn   #警告级别
err, error      #错误级别，阻止某个功能或者模块不能正常工作的信息
crit            #严重级别，阻止整个系统或者整个软件不能正常工作的信息
alert           #需要立刻修改的信息
emerg, panic    #内核崩溃等严重信息
```

## Target


- 文件：       /var/log/messages
- 用户：       root，*（表示所有用户）， 会发到/var/spool/mail/<user>收件箱里
- 日志服务器：  @192.168.56.10  或者 @@192.168.56.10
- 管道：       | COMMAND

一个 `@` 表示 `UDP`, 两个 `@@` 表示 `TCP` 协议。

# 常用指令

## 模板$template
模板 `$template`， 最主要的一个指令，在 **接收端** 可用来定义消息格式、文件名。主要是在接收端使用。
可参考 [官方文档Templates](http://www.rsyslog.com/doc/v5-stable/configuration/templates.html)

语法：
``` bash
$template <name>,<内容>,<可选项>
$template MyTemplateName,"\7Text %property% some more text\n",<options>
```

默认的消息格式  `RSYSLOG_TraditionalFileFormat`
``` bash
<接收内容的时间>  <发送者的hostname>   <$InputFileTag> <原始消息%msg%>
Dec 18 20:39:27 jumper-172-31-56-18 karltestdemoTag blala... dummy msg
```
如果只需要显示原始消息，可设置
``` bash
$template CleanMsgFormat,"%msg%\n"
```

## 内置属性 Properties
模板里支持一些 [内置的变量](http://www.rsyslog.com/doc/v5-stable/configuration/properties.html)：

``` bash
%msg%
%syslogfacility%
%HOSTNAME%
%syslogpriority%
%timereported:::date-mysql%
%timegenerated:::date-mysql%
%iut%
'%syslogtag%'
```

## 属性处理 Property Replacer

[Property Replacer](http://www.rsyslog.com/doc/v5-stable/configuration/property_replacer.html) 用来处理变量，支持一些简单的字符串处理，如大小写，substring等，类似jinja2的filter概念。
版本越新支持的处理函数越强大。
语法： `%property:fromChar:toChar:options%`
``` bash
# For product
$template productFileFormat,"/Data/logs/product/%fromhost-ip%/%syslogtag:F,44:2%-%$YEAR%%$MONTH%%$DAY%.log"
if $syslogtag startswith 'product' then ?productFileFormat;CleanMsgFormat
& ~
```

上面的例子，在接收端会保存为 `/Data/logs/product/198.168.56.123/karltest_demo-20161218.log`

解释下这个指令`%syslogtag:F,44:2%`

`F,` - 代表自定义一个分隔符
`44` - 是逗号 `,` 的 ASCII 码值，如需要别的分隔符，需要查对应 ASCII 值
`2`  - 取分隔后的第二个字段

所以就是： 假设发送端自定义的tag为 `$InputFileTag   product,karltest_demo`，
如果tag以product开始，则取出逗号分隔的第二个字段作为保存的文件名，这也是为啥上面tag里要设置一个逗号的缘故。

另外，还支持一定的 regex 语法，可以进行更高级的控制。官方提供了一个[在线的 regex 语法测试](http://www.rsyslog.com/regex/)。 
*友情提醒：真的很难用。。。*

## 停止指令
上面的接收端，在一条规则后，加上了如下的指令（也叫停止指令），代表如果log被当前的rule已经处理过了，则完成本次执行，跳过后续rule的处理。** 类似 C++里 switch/case，如果忘记加break的穿透副作用**。 
```
& ~
```
如果没有这个指令，则一条新来的消息可以被多个rule处理。 这里我们并不需要，只要命中就保存到接收端同名的文件里。


# 发送端配置
1. 加载 imfile 模块
2. 指定要监控的 log 文件路径，设置合适的tag
3. 指定远端的接收端的地址

完整配置： [/etc/rsyslog.conf](/files/center-log/send-rsyslog-conf.txt)  和 [/etc/rsyslog.d/product.conf](/files/center-log/send-rsyslogd-product-sample.txt)

# 接收端配置

1. 加载 imtcp 模块
2. 设置 message 格式
3. 设置 文件存储路径，文件名格式

完整配置： [/etc/rsyslog.conf](/files/center-log/recv-rsyslog-conf.txt)


# 遇到的坑

## UDP or TCP ?

一般来说选择TCP都是OK的，除非忍受部分丢失，在意影响性能，可以改用UDP。
但是注意：**如果你的消息每行大小超过了4k，只能用TCP**。这是因为UDP栈大小限制的。

引用官方有关 [MaxMessageSize](http://www.rsyslog.com/doc/v5-stable/configuration/global/index.html) 的描述： 
>  Note: testing showed that 4k seems to be the typical maximum for UDP based syslog. This is an IP stack restriction. Not always ... but very often. If you go beyond that value, be sure to test that rsyslogd actually does what you think it should do ;) It is highly suggested to use a TCP based transport instead of UDP (plain TCP syslog, RELP). This resolves the UDP stack size restrictions. 

## 如何测试： vim vs echo ?

配置好接收端、发送端rsyslog后，需要验证下是否能正确传送新的log行。

1. echo追加： `echo "dummy message" >> /Data/logs/product/karltest.log`
2. vim编辑：  ` vim /Data/logs/product/karltest.log `

用vim编辑后，保存会刷新整个文件，导致rsyslog在比较state file的时候，认为全部是新增的行，会在接收端出现重复的log行。
所以正确测试方法是用 echo 追加的方式。

> Tips: 
> 发送端：可以配合 watch 来测试： ` watch -n 1 " echo $(date) dummy message >> /Data/logs/product/karltest.log " `
>  接收端： ` tailf  /Data/logs/karltest/karltest.log `


## 接收端：log行太长，被截断了
默认大小是2k，大概可以保存1000个中文字符，参考官方说明 [$MaxMessageSize](http://www.rsyslog.com/doc/v5-stable/configuration/global/index.html)， 最小也是2k。

在加载imtcp/imudp之前设置， **此配置包括发送和接收，所以rsyslog客户端、服务端都要设置**：
``` bash
$MaxMessageSize 32k

# Provides TCP syslog reception
#$ModLoad imtcp
#$InputTCPServerRun 514
```

## 发送端：/var/log/messages 文件变大
log除了发送到了接收端，还在本地 `/var/log/messages` 里重复出现了，导致messages上G
罪魁祸首是默认的配置文件如下这行：

``` bash
# 修改前
*.info;mail.none;authpriv.none;cron.none                            /var/log/messages      

# Fix后
*.info;mail.none;authpriv.none;cron.none;local5.none;local6.none    /var/log/messages      
```
因为前面有通配符 `*.info` 导致我们自定义的 `local5.info` 也会写到本地messages文件。
**为了保险，请把接收端、发送端的配置文件都修改掉，忽略掉local5。**

## 接收端：消息被多个rule处理
在命中某条rule后，直接break停止

``` bash
if $syslogtag startswith 'product' then ?productFileFormat;CleanMsgFormat
& ~

# 新版本v6之后，变为：
# rsyslogd: warning: ~ action is deprecated, consider using the 'stop' statement instead [try http://www.rsyslog.com/e/2307 ]
if $syslogtag startswith 'product' then ?productFileFormat;CleanMsgFormat
stop

```

## 接收端： 保存的文件路径不对
要注意自定义tag的前缀匹配，**如果两个tag有共同的前缀，需要把长的放在前面**，调整好顺序。

Fix 前：
``` bash
# For erp_wms
$template erp_wms_FileFormat,"/Data/logs/erp/wms/%fromhost-ip%/%syslogtag:F,44:2%-%$YEAR%%$MONTH%%$DAY%.log"
if $syslogtag startswith 'erp_wms' then ?erp_wms_FileFormat;CleanMsgFormat
& ~

# For erp_wms3
$template erp_wms3_FileFormat,"/Data/logs/erp/wms3/%fromhost-ip%/%syslogtag:F,44:2%-%$YEAR%%$MONTH%%$DAY%.log"
if $syslogtag startswith 'erp_wms3' then ?erp_wms3_FileFormat;CleanMsgFormat
& ~
```

Fix 后：
``` bash
# 这里注意下面的tag的顺序， 一定要让长的tag（erp_wms3）保持在上面，因为他们有共同的前缀(erp_wms)
# For erp_wms3
$template erp_wms3_FileFormat,"/Data/logs/erp/wms3/%fromhost-ip%/%syslogtag:F,44:2%-%$YEAR%%$MONTH%%$DAY%.log"
if $syslogtag startswith 'erp_wms3' then ?erp_wms3_FileFormat;CleanMsgFormat
& ~

# For erp_wms
$template erp_wms_FileFormat,"/Data/logs/erp/wms/%fromhost-ip%/%syslogtag:F,44:2%-%$YEAR%%$MONTH%%$DAY%.log"
if $syslogtag startswith 'erp_wms' then ?erp_wms_FileFormat;CleanMsgFormat
& ~
```

## 接收端： rsyslog 文件名太长后被截断

比如发送端原始文件名tag: `product，cache_status_im_request.log`
但是到了接收端就截断了： `cache_status_im_re-20161218.log`
因为本来名字就长，加上了时间后更长了，

个人理解，Linux中关于文件名（255），文件路径（4096）的限制如下，而在接收端，都没有超过这个长度。


	$ cat /usr/include/linux/limits.h
	
	#ifndef _LINUX_LIMITS_H
	#define _LINUX_LIMITS_H
	
	#define NR_OPEN         1024
	
	#define NGROUPS_MAX    65536    /* supplemental group IDs are available */
	#define ARG_MAX       131072    /* # bytes of args + environ for exec() */
	#define LINK_MAX         127    /* # links a file may have */
	#define MAX_CANON        255    /* size of the canonical input queue */
	#define MAX_INPUT        255    /* size of the type-ahead buffer */
	#define NAME_MAX         255    /* # chars in a file name */
	#define PATH_MAX        4096    /* # chars in a path name including nul */
	#define PIPE_BUF        4096    /* # bytes in atomic write to a pipe */
	#define XATTR_NAME_MAX   255    /* # chars in an extended attribute name */
	#define XATTR_SIZE_MAX 65536    /* size of an extended attribute value (64k) */
	#define XATTR_LIST_MAX 65536    /* size of extended attribute namelist (64k) */
	
	#define RTSIG_MAX         32
	
	#endif


进过一番艰难的测试复现，猜测是 `$InputFileTag` 这个 `%syslogtag%`的原因。

官方解释[Sending messages with tags larger than 32 characters](http://www.rsyslog.com/sende-messages-with-tags-larger-than-32-characters/)。

发送端默认的模板为：
``` bash
# 忽略旧的5.8.6的语法
template (name="ForwardFormat" type="string" string="<%PRI%>%TIMESTAMP:::date-rfc3339% %HOSTNAME%
%syslogtag:1:32%%msg:::sp-if-no-1st-sp%%msg%")
```

可以看到 `%syslogtag:1:32%`，被截断到32字符，这个也与我测试的结果一致。

所以如果需要处理更长的tag，需要修改 发送端的template模板，去掉 `:1:32` 限制。
然后绑定这个模板到对应的target上。
注意：接收端可能也要相应处理，才能handle更长的tag名（未测试）。

> 免责声明：由于折腾这个rsyslog太累，最后一条暂时没有Fix，如有需要，请自行测试后再用。

# 接下来......

通过一番设置，我们已经能成功收集若干台线上机器的日志了：

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

那么问题来了：

> 上百台机器的上几百G的日志，怎么才能避免接收端硬盘爆掉？

得用另一个Linux自带的脚本 `/usr/sbin/logrotate`, 来配合 rsyslog。

请参考： [日志集中化收集（二）：logrotate 配置]()

# 参考链接

1. [rsyslog官方v5版本文档](http://www.rsyslog.com/doc/v5-stable/)
2. [rsyslog imfile 模块](http://www.rsyslog.com/doc/v5-stable/configuration/modules/imfile.html)
3. [rsyslog Template](http://www.rsyslog.com/doc/v5-stable/configuration/templates.html)
4. [rsyslog 内置属性](http://www.rsyslog.com/doc/v5-stable/configuration/properties.html)
5. [rsyslog 属性处理](http://www.rsyslog.com/doc/v5-stable/configuration/property_replacer.html)
6. [Storing Messages from a Remote System into a specific File](http://www.rsyslog.com/storing-messages-from-a-remote-system-into-a-specific-file/)
7. [Writing specific messages to a file and discarding them](http://www.rsyslog.com/writing-specific-messages-to-a-file-and-discarding-them/)
8. [Sending messages with tags larger than 32 characters](http://www.rsyslog.com/sende-messages-with-tags-larger-than-32-characters/)

