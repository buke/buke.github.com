---
layout: post
title: "OpenERP 邮件发送问题修复 -- 解决 qq/163 等邮箱发送失败问题"
date: 2014-04-11 11:24
comments: true
categories: OpenERP
---

openerp-mail-server-smtp-user

Fixed email from header.修复OpenERP 邮件头From 格式错误。

默认情况下，OpenERP 打包邮件时, 直接使用 email from 作为邮件头的 From 值。如qq/163等邮箱会严格校验email from 和 发件人帐号是否一致，如不一致则返回发送失败。如以下情况：

{% codeblock %}
OE 默认情况
发件人为： admin@ex.com
发送账户为：11111@qq.com

OE 打包邮件内容为 email['From'] == 'admin@ex.com' (qq/163 校验失败)

安装本模块后：
发件人为： admin@ex.com
发送账户为：11111@qq.com

OE 打包邮件内容为 email['From'] == 'admin@ex.com <11111@qq.com> ' (qq/163 校验成功)
{% endcodeblock %}


下载地址： [https://github.com/buke/openerp-mail-server-smtp-user](https://github.com/buke/openerp-mail-server-smtp-user)


