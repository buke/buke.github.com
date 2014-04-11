---
layout: post
title: "修改py文件自动重启加载进程方法"
date: 2013-06-30 22:49
comments: true
categories: Linux OpenERP
---

一般来说我们的 Python 程序在运行时会自动编译为 .pyc 文件，这样可以加快 Python 程序的启动速度。但在开发调试的时，我们不得不频繁重启进程，让人不胜其烦。

秉承“不会偷懒的程序员不是好程序员”的原则，写了个 autoreload 脚本(兼容 linux/windows)，项目开源地址 [https://github.com/buke/autoreload](https://github.com/buke/autoreload)

完整代码如下：

{% codeblock lang:python %}
#!/usr/bin/env python
# -*- coding: utf-8 -*-
##############################################################################
#    AutoReload Process
#    wangbuke <wangbuke@gmail.com>
#    taken from https://github.com/stevekrenzel/autoreload
#    
#    To use autoreload:    
#    1. Make sure the script is executable by running chmod +x autoreload
#    2. Run ./autoreload <command to run and reload> 
#       e.g: $ ./autoreload openerp-server
#    
##############################################################################

import sys
import subprocess
import time
import os
import signal

PATH = '.'
WAIT = 1

def start_process(cmd):
    if os.name == 'nt':
        process = subprocess.Popen(command, shell=True)
    else:
        process = subprocess.Popen(cmd, shell=True, preexec_fn=os.setsid) 
    return process

def stop_process(process):
    if os.name == 'nt':
        process.kill()
    else:
        os.killpg(process.pid, signal.SIGTERM) # Send the signal to all the process groups
    process.wait()

def file_filter(name):
    return (not name.startswith(".")) and (not name.endswith(".pyc"))  and (not name.endswith(".pyo"))and (not name.endswith(".swp"))

def file_times(path):
    for root, dirs, files in os.walk(path):
        for file in filter(file_filter, files):
            yield os.stat(os.path.join(root, file)).st_mtime

def print_stdout(process):
    stdout = process.stdout
    if stdout != None:
        print stdout

if __name__ == "__main__":

    command = ' '.join(sys.argv[1:])
    process = start_process(command)
    last_mtime = max(file_times(PATH))

    while True:
        try:
            print_stdout(process)
            max_mtime = max(file_times(PATH))
            if max_mtime > last_mtime:
                last_mtime = max_mtime
                print 'Restarting process...'
                stop_process(process)
                process = start_process(command)
            time.sleep(WAIT)
        except (KeyboardInterrupt, SystemExit):
                print "Caught KeyboardInterrupt, terminating process"
                stop_process(process)
                break

# vim:expandtab:smartindent:tabstop=4:softtabstop=4:shiftwidth=4:

{% endcodeblock %}

可配置参数：
* PATH = '.' #默认为 autoreload 文件所在的当前目录。
* WAIT = 1 # 默认扫描间隔，单位秒

用法:
1、修改该文件为可执行脚步 (windows 下执行 python autoreload)
{% codeblock %}
$ chomd +x ./autorelaod
{% endcodeblock %}

2、运行
{% codeblock %}
$ ./autoreload "openerp-server -c openerp-server.conf"
{% endcodeblock %}


项目开源地址 [https://github.com/buke/autoreload](https://github.com/buke/autoreload) ,  如有意见或建议请到上面地址提交反馈。谢谢！

