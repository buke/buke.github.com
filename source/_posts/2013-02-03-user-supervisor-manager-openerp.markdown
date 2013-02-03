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
                
# 1、安装
debian/ubuntu
{% codeblock %}
apt-get install supervisor
{% endcodeblock %}

# 2、
