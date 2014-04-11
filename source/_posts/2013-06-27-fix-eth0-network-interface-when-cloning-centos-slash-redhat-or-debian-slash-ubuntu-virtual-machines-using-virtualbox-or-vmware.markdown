---
layout: post
title: "复制 Virtualbox/vmware linux虚拟机网络无法访问错误的解决方法"
date: 2013-06-27 16:36
comments: true
categories: Linux VirtualBox
---

使用 VirtualBox/VMWare 复制 centos/redhat 或 debian/ubuntu 虚拟机时，如果你选择了初始化网卡MAC地址，复制之后往往会发现网络无法使用。

如执行以下命令则提示错误

{% codeblock  %}
# ifup eth0
Device eth0 does not seem to be present, delaying initialisation
{% endcodeblock %}

原因是VirtualBox/VMWare 在复制时改变了网卡的MAC地址，和原虚拟机网卡eth0地址不符。

解决办法：

1、删除网络规则文件
{% codeblock  %}
# rm -f /etc/udev/rules.d/70-persistent-net.rules
{% endcodeblock %}

2、修改 eth0 的配置文件 (debian/ubuntu 跳过)
{% codeblock  %}
# vim /etc/sysconfig/network-scripts/ifcfg-eth0
删除 HWADDR 和 UUID 行即可
{% endcodeblock %}

3、重启
{% codeblock  %}
# reboot 
{% endcodeblock %}

完成！


