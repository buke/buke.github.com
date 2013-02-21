---
layout: post
title: "OpenERP 自动编码去BOM（可用excel编辑）"
date: 2012-09-05 15:00
comments: true
categories: OpenERP
---

openerp-web-import-chardet

作者：wangbuke@gmail.com

源码托管地址： https://github.com/buke/openerp-web-import-chardet

OE apps 下载地址： http://apps.openerp.com/addon/8098

# 功能：

自动检测OpenERP 导入的CSV文件编码 自动移除UTF8文件的BOM。安装完之后，就可以直接用EXCEL WPS等编辑好的CSV文件，导入到OpenERP中。

# 支持编码：

* ASCII, UTF-8
* Big5, GBK, GB2312, HZ-GB, HZ-GB-2312 (简体/繁体中文)
* EUC-JP, SHIFT_JIS, ISO-2022-JP (日文)
* EUC-KR, ISO-2022-KR (韩文)
* KOI8-R, MacCyrillic, IBM855, IBM866, ISO-8859-5, windows-1251 (斯拉夫文)
* ISO-8859-2, windows-1250 (匈牙利文)
* ISO-8859-5, windows-1251 (保加利亚文)
* windows-1252 (英文)
* ISO-8859-7, windows-1253 (希腊文)
* ISO-8859-8, windows-1255 (希伯来文)
* TIS-620 (泰文)

# 依赖模块:

python chardet (本模块已内含chardet-1.1。如果系统安装了新版本，则使用您安装的新版本.)

 
{% img /images/my/oe-chardet.jpg %}

