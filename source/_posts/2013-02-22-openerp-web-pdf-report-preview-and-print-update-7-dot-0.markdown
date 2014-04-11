---
layout: post
title: "Openerp Web PDF Report Preview &amp; Print 模块升级至 7.0"
date: 2013-02-22 19:18
comments: true
categories: OpenERP
---

Openerp Web PDF Report Preview & Print

下载地址： [https://github.com/buke/openerp-web-pdf-preview-print](https://github.com/buke/openerp-web-pdf-preview-print)

简介：

将OpenERP 的PDF报表打印下载功能，改为直接在浏览器中预览打印。

* For IE， 需要安装 Adobe Reader。
* For Firefox 19 + , 神马都不用安装。
* For Chrome, 神马都不用安装。

以上在windows 上测试通过。如果浏览器阻止了弹出窗口，请点允许弹出窗口。

系统要求：

* OpenERP 7.0


注：

1.  openerp-7.0-20130222-002152 下测试通过
2.  之前版本可能会出现以下错误：

TypeError: this.get_action_manager(...) is undefined on Firefox

TypeError: Cannot call method 'get_title' of undefined on Chrome / IE

解决办法有2种

F5 刷新页面重新加载 或者  升级OpenERP 版本


{% img /images/my/oe-pdf-7.jpg %}
