---
layout: post
title: "linux 下使用 Supervisor 管理源码启动的 OpenERP"
date: 2012-07-16 22:50
comments: true
categories: OpenERP
---

从源码启动openerp，简单的做法是添加启动脚本到/etc/init.d/rc.local等，让openerp 随系统启动而运行。此类方法只在系统启动时运行，但万一程序在运行中崩溃，您可能要等到用户发现不能使用了，才去重启服务器。下面请出今天的主角： supervisor   ( http://supervisord.org/)

Supervisor 是什么？
        
Supervisor is a client/server system that allows its users to monitor and control a number of processes on UNIX-like operating systems.
Supervisor 是一个客户端/服务器系统，允许用户监控和控制类 Unix 操作系统上的进程数。
                
## 1、安装

debian/ubuntu

{% codeblock %}
apt-get install supervisor
{% endcodeblock %}

redhat/centos
{% codeblock %}
yum install supervisor
{% endcodeblock %}

## 2、建立openerp 的配置文件

{% codeblock %}
# touch /etc/supervisor/conf.d/openerp.conf
# vi /etc/supervisor/conf.d/openerp.conf
{% endcodeblock %}

openerp.conf 内容
{% codeblock %}
[program:openerp]
; openerp 启动脚本
command=python /var/www/openerp-6.1-1/openerp-server -c /var/www/openerp-6.1-1/openerp-server.conf
; openerp 目录
directory=/var/www/openerp-6.1-1/
; 是否随系统启动
autostart=true
; 自动重启
autorestart=true
; 启动时间，如果超过这个时间oe还没有挂，则视为已经启动
startsecs=3
; 启动用户
user=www-data
redirect_stderr=true
; log 文件
stdout_logfile=/var/www/openerp-6.1-1/openerp-server.log
stdout_logfile_maxbytes=500MB
stdout_logfile_backups=50
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
loglevel=warn
{% endcodeblock %}

## 3、 完成！
重启系统试试看openerp 是否已经启动。也可以想办法把openerp 搞崩溃，试试supervisor 能不能及时将openerp 重启 

## 4、 常用命令
{% codeblock %}
# supervisorctl 
openerp                          RUNNING    pid 9454, uptime 4:43:34
supervisor> start openerp  #启动
supervisor> stop openerp   #停止
supervisor> restart openerp #重启
supervisor> status openerp #查看状态
{% endcodeblock %}




