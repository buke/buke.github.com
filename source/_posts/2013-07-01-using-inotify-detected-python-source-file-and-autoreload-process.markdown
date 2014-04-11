---
layout: post
title: "使用 inotify 监视 python 源码文件自动重新加载进程"
date: 2013-07-01 17:40
comments: true
categories: Linux OpenERP
---

在上一篇文章 [http://buke.github.io/blog/2013/06/30/reload-the-process-when-source-chage-is-detected/](修改py文件自动重启加载进程方法) 中，实现了监视 py 文件并自动重启进程。

但核心实现是定时轮询获取文件的最新修改时间，效率低下而且很不 pythonic。

{% codeblock lang:python %}
    while True:
        try:
            print_stdout(process)
            max_mtime = max(file_times(PATH,extension))
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
{% endcodeblock %}

对于程序员来说，定时轮询基本是不能接受滴 ~ 

所幸的是，linxu 2.6 + 开始在内核提供 inotify，为监控文件变化提供了良好的事件支持。来自 wiki 的介绍：

inotify是Linux核心子系统之一，做为文件系统的附加功能，它可监控文件系统并将异动通知应用程序。本系统的出现取代了旧有Linux核心里，拥有类似功能之dnotify模块。

因此，https://github.com/buke/autoreload for linux 使用 pyinotify 模块加以改进。 完整代码如下：

{% codeblock lang:python %}
#!/usr/bin/env python
# -*- coding: utf-8 -*-
##############################################################################
# AutoReload Process Using inofify
# wangbuke <wangbuke@gmail.com>
# taken from https://github.com/stevekrenzel/autoreload
#
# To use autoreload:
# 1. Make sure the script is executable by running chmod +x autoreload
# 2. Run ./autoreload <path> <ext1,ext2,extn> <cmd>
# e.g: $ ./autoreload '.' '.py,.xml,.conf' './openerp-server -c openerp-server.conf'
#
##############################################################################

import os
import signal
import subprocess
import sys
import time
import pyinotify
from pyinotify import log

class ReloadNotifier(pyinotify.Notifier):
    def loop(self, callback=None, daemonize=False, **args):
        if daemonize:
            self.__daemonize(**args)

        # Read and process events forever
        while 1:
            try:
                self._default_proc_fun._print_stdout()
                self.process_events()
                if (callback is not None) and (callback(self) is True):
                    break
                ref_time = time.time()
                # check_events is blocking
                if self.check_events():
                    self._sleep(ref_time)
                    self.read_events()
            except KeyboardInterrupt:
                # Stop monitoring if sigint is caught (Control-C).
                log.debug('Pyinotify stops monitoring.')
                self._default_proc_fun._stop_process()
                break
        # Close internals
        self.stop()

class OnChangeHandler(pyinotify.ProcessEvent):
    def my_init(self, cwd, extension, cmd):
        self.cwd = cwd
        self.extensions = extension.split(',')
        self.cmd = cmd
        self.process = None
        self._start_process()

    def _start_process(self):
        self.process = subprocess.Popen(self.cmd, shell=True, preexec_fn=os.setsid)

    def _stop_process(self):
        os.killpg(self.process.pid, signal.SIGTERM)
        self.process.wait()

    def _restart_process(self):
        print '==> Modification detected, restart process. <=='
        self._stop_process()
        self._start_process()

    def _print_stdout(self):
        stdout = self.process.stdout
        if stdout != None:
            print stdout

    def process_IN_CREATE(self, event):
        if any(event.pathname.endswith(ext) for ext in self.extensions) or "IN_ISDIR" in event.maskname:
            self._restart_process()

    def process_IN_DELETE(self, event):
        if any(event.pathname.endswith(ext) for ext in self.extensions) or "IN_ISDIR" in event.maskname:
            self._restart_process()

    def process_IN_CLOSE_WRITE(self, event):
        if any(event.pathname.endswith(ext) for ext in self.extensions) or "IN_ISDIR" in event.maskname:
            self._restart_process()

    def process_IN_MOVED_FROM(self, event):
        if any(event.pathname.endswith(ext) for ext in self.extensions) or "IN_ISDIR" in event.maskname:
            self._restart_process()

    def process_IN_MOVED_TO(self, event):
        if any(event.pathname.endswith(ext) for ext in self.extensions) or "IN_ISDIR" in event.maskname:
            self._restart_process()

def autoreload(path, extension, cmd):
    wm = pyinotify.WatchManager()
    handler = OnChangeHandler(cwd=path, extension=extension, cmd=cmd)
    notifier = ReloadNotifier(wm, default_proc_fun=handler)

    mask = pyinotify.IN_CLOSE_WRITE | pyinotify.IN_MOVED_FROM | pyinotify.IN_MOVED_TO | pyinotify.IN_CREATE | pyinotify.IN_DELETE | pyinotify.IN_DELETE_SELF | pyinotify.IN_MOVE_SELF
    wm.add_watch(path, mask, rec=True, auto_add=True)

    print '==> Start monitoring %s (type c^c to exit) <==' % path
    notifier.loop()

if __name__ == '__main__':
    path = sys.argv[1]
    extension = sys.argv[2]
    cmd = sys.argv[3]
    autoreload(path, extension, cmd)

{% endcodeblock %}

用法：
1. $ chmod +x autoreload (修改为可执行文件)

2. Run ./autoreload <path> <ext1,ext2,extn> <cmd>
   e.g: $ ./autoreload '.' '.py,.xml,.conf' './openerp-server -c openerp-server.conf'

以上代码已开源并发布在[https://github.com/buke/autoreload](https://github.com/buke/autoreload) ， 欢迎反馈意见和建议！



