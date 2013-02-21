---
layout: post
title: "OpenERP PyChart 中文报表模块（支持CJK语言）"
date: 2012-08-19 15:44
comments: true
categories: OpenERP
---

OpenERP PyChart Unicode Report (Support CJK Font)

作者：wangbuke@gmail.com

源码托管地址：https://github.com/buke/openerp-pychart-unicode-report

OpenERP 官方APP下载地址： http://apps.openerp.com/addon/8009

支持pychart中文报表，如“库存预测”、“工作中心负载” 等报表。

# 模块原理

让pychart 生成svg 文件，然后用cairosvg 模块生成PDF报表。

# 依赖模块

python-cairo python-cairosvg 

Debian/Ubuntu安装方法： $ su apt-get install python-cairo python-cairosvg

# 安装与设置

## 1、安装字体

复制您所用的字体文件，如simsun.ttc 到系统目录下。

debian/ubuntu: $ sudo cp simsun.ttc /usr/share/fonts

windows :  C:\> copy simsun.ttc  c:/windows/fonts

### 2、配置pychart 报表字体 默认使用宋体

修改openerp 配置文件 openerp-server.conf , 添加以下参数：

pychart_ttfont_name = Simsun

注：默认是宋体，如使用默认值则无需修改 conf 文件

祝你好运 ~


{% img /images/my/oe-pychart-report.jpg %}

