---
layout: post
title: "使用 Eclipse Pydev 调试 GreenOpenERP For Windows"
date: 2013-06-27 13:32
comments: true
categories: OpenERP GreenOpenERP Windows Eclipse Pydev
---

##关于 GreenOpenERP

OpenERP 绿色版 For Windows，源码运行，解压即用，集成 python/postgresql/openerp

相关介绍和下载，请看 [http://buke.github.io/blog/2013/03/10/greenopenerp-portable-openerp-for-windows/](http://buke.github.io/blog/2013/03/10/greenopenerp-portable-openerp-for-windows/)


##关于 Eclipse 和 Pydev

Eclipse, 请到[http://www.eclipse.org/](http://www.eclipse.org/) 下载安装。

Pydev, 请到[http://pydev.org/](http://pydev.org/) 下载安装。(推荐使用eclipse 在线安装 pydev，打开eclipse 后，点击菜单 “help” -> "Install New Software", 输入 “http://pydev.org/updates” 刷新后选择 pydev 安装即可。)

##源码调试 GreenOpenERP

此处假设您已经安装好 Eclipse 和 Pydev，并可以正确运行。

### 1、下载 GreenOpenERP
从[http://sourceforge.net/projects/greenopenerp/files/](http://sourceforge.net/projects/greenopenerp/files/) 下载并解压 GreenOpenERP-7.0-xxxxxxxx-xxxxxx 

###2、配置 pydev 的 python interpreters

打开 eclipse ， 选择菜单 “Window” -> "Preferences"

{% img /images/my/pydev-0.png %}

点击 pydev，选择 Interpreter - Python , 如有旧的Interpreter 请先删除。点击 New ，在弹出的窗口中 Browse 到下载解压的 GreenOpenERP-7.0-xxxxxxxx-xxxxxx\python\python.exe 

{% img /images/my/pydev-1.png %}

选择 Select All， 然后点击 OK

{% img /images/my/pydev-2.png %}

回到刚才窗口， python interpreters 的相关参数应已配好。点击 Apply 后，点 Ok

{% img /images/my/pydev-3.png %}

###3、选择 pydev 开发界面。

Eclipse 默认是 java 开发界面，点击右上角的小图表，在弹出的窗口选择 pydev

{% img /images/my/pydev-4.png %}

###4、导入 GreenOpenERP Project

在 Package Explorer 界面内点击右键，在弹出的菜单里选择 Import

{% img /images/my/pydev-5.png %}

选择 Existing Project into Workspace 后，点 Next

{% img /images/my/pydev-6.png %}

点击 Browse 后，选择解压后的 GreenOpenERP 文件夹, 随后点击 Finish

{% img /images/my/pydev-7.png %}

###5、启动 Postgresql 

运行 GreenOpenERP 文件夹的 start-pg.bat 启动 pg 数据库。（关闭pg 数据库则运行 stop-pg.bat）


###6、启动调试

在相应文件设置好断点后（也可不设置断点），在 eclipse 的 Package Explorer 界面右键点击 openerp-server, 在弹出菜单选择 Debug As -> Python Run

{% img /images/my/pydev-8.png %}


###7、完成

送断点调试界面图一张

{% img /images/my/pydev-9.png %}


###8、注意事项

* 调试前需运行 start-pg.bat 启动数据库。
* 如需关闭数据库，请运行 stop-pg.bat 关闭。切勿直接关闭cmd窗口。
* 正常运行(非调试)和关闭 GreenOpenERP，请运行 start.bat 和 stop.bat .

