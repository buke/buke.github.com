---
layout: post
title: "Taobao OpenERP Connector 简要说明"
date: 2012-07-18 16:29
comments: true
categories: OpenERP 
---

Taobao OpenERP Connector

项目托管地址：[https://github.com/buke/openerp-taobao](https://github.com/buke/openerp-taobao)

作者： wangbuke@gmail.com


# 功能：

1. 接受淘宝主动通知，自动添加、确认订单、发货等。
2. 同步淘宝订单
3. 导入淘宝产品, 同步库存
4. 导入淘宝用户
5. 自动评价，中差评预警
6. 跟踪淘宝订单物流信息, 签收提醒
7. .... 等等等 (懒的写了，自己发现吧)

# 系统要求：

* OpenERP 6.1
* beanstalkd
* pycurl

# 安装说明：

## 1. 安装beanstalkd

### 1.1 linux 系统

debian/ubuntu: # apt-get install beanstalkd

redhat/centos: # yum install beanstalkd

安装完成之后，开启beanstalkd的持久化选项：

{% codeblock %}
# vi /etc/default/beanstalkd

## Defaults for the beanstalkd init script, /etc/init.d/beanstalkd on
## Debian systems. Append ``-b /var/lib/beanstalkd'' for persistent
## storage.
BEANSTALKD_LISTEN_ADDR=0.0.0.0
BEANSTALKD_LISTEN_PORT=11300
#DAEMON_OPTS="-l $BEANSTALKD_LISTEN_ADDR -p $BEANSTALKD_LISTEN_PORT"
DAEMON_OPTS="-l $BEANSTALKD_LISTEN_ADDR -p $BEANSTALKD_LISTEN_PORT -b /var/lib/beanstalkd"

## Uncomment to enable startup during boot.
START=yes
{% endcodeblock %}

### 1.2 windows 系统

beanstalkd 原生不能在windows 下运行，当然也有大牛用cgywin 编译了一个。请参考 http://software1987.de/2011/03/beanstalkd-unter-windows-mit-cygwin/  。编译后的 beanstalkd 下载地址是 http://software1987.de/wp-content/uploads/2011/03/beanstalkd-1.4.6-cygwin.zip

下载解压后，打开 cmd.exe 运行

{% codeblock %}
C:\beanstalkd\bin>beanstalkd.exe -l 127.0.0.1 -p 11300 -b C:\beanstalkd
{% endcodeblock %}

注意上面的目录路径，根据您的实际情况修改。 -b 后面是目录，用于存放beanstalkd 持久化的文件。 上面是直接运行，当然您也可以创建快捷方式，或者用runasservice 工具封装成windows 的服务。

## 2. 安装pycurl

### 2.1 linux 系统

debian/ubuntu: # apt-get install python-pycurl

redhat/centos: # yum install python-pycurl

### 2.2 windows 系统

#### 2.2.1 OpenERP - 源码安装

##### 2.2.1.1 安装 python （如已经安装则跳过）

到 http://python.org/ 下载安装，不解释

##### 2.2.1.2 安装 pycurl（如已经安装则跳过）

到 http://www.lfd.uci.edu/~gohlke/pythonlibs/#pycurl 下载对应版本的 pycurl 安装

#### 2.2.2 OpenERP - all in one  

all in one 的版本 在安装完以上步骤之外，还需要把 C:\Python26\Lib\site-packages 目录下的 curl 目录 和 pycurl.pyd 文件 复制到 C:\Program Files\OpenERP 6.1-20120717-233333\Server\server 目录中。（注意路径！，根据实际情况修改）不然下面的安装会提示找不到pycurl。

注意：我现在的all in one （OpenERP 6.1-20120717-233333\） python版本 2.6，所以使用all in one 版本的同学注意了，上面2步都要下载安装for python 2.6 版本的。 （通过看C:\Program Files\OpenERP 6.1-20120717-233333\Server\server\python26.dll这个文件的后缀可以知道python 版本）

## 3. 安装 Taobao OpenERP Connector 模块

这里和OE安装模块方法一样。首先到https://github.com/buke/openerp-taobao 下载，然后有2种方法：一种是把taobao 文件夹放到OpenERP 的 addon 目录下，第二种是把taobao 文件夹压缩为zip 文件，通过OE后台上传模块。

## 4. OpenERP conf 文件配置参数

Taobao OpenERP Connector 模块有几个默认配置参数如下：

{% codeblock %}
beanstalkd_interface = localhost
beanstalkd_port = 11300
taobao_stream_service = True
taobao_stream_thread_limit = 1
taobao_worker_thread_limit = 4 
{% endcodeblock %}

 上面是默认值，如果您不需要修改则不用放入OpenERP 启动的 conf中。反之，如果你需要修改 ，则将上面几个参数写在conf 文件中。

## 5. 关于淘宝 api 的几个问题

首先登陆 open.taobao.com 创建一个 C/S 架构 自用型应用，然后开通主动通知业务。

App Key : 自己找，不解释

App Secret: 自己找，不解释

App SessionKey: 获取方法

1. 先访问 http://my.open.taobao.com/auth/authorize.htm?appkey={appkey}获得授权码
2. 再访问 http://container.open.taobao.com/container?authcode={授权码},会得到类似如下的字符串top_appkey=1142&top_parameters=xxx&top_session=xxx&top_sign=xxx,字符串里面的top_session值即为SessionKey

根据淘宝文档说明，C/S应用的 SessionKey 有效期为一年，大家到时记得更新。

PS:

配置淘宝商店的时候出现报错的，请检查你们的淘宝应用权限 。必须是C/S架构的商家后台系统。淘宝规定请看 http://dev.open.taobao.com/bbs/read.php?tid=24315  自2012年7月12日起，“商家后台系统标签”的申请只允许商城店铺和集市三皇冠以上商家申请。 

欢迎大家参与此项目，或者到https://github.com/buke/openerp-taobao 提需求、BUG等，也可以直接给我来信。谢谢~ 





