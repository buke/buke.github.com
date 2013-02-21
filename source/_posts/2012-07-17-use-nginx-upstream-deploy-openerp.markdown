---
layout: post
title: "使用Nginx Upstream 部署 OpenERP"
date: 2012-07-17 15:51
comments: true
categories: OpenERP
---

Openerp 6.1 使用werkzeug 作为web服务的框架，性能比之前的cherrypy 有了很大的改善。但无论是 werkzeug 还是cherrypy ，都不是专门的web服务器。通常的做法是在openerp 之前加一个 Nginx、Apache或其他服务器。下面介绍使用Nginx Upstream 部署openerp 的方法。

# 一 前提

此处假设您已经安装好 openerp ，并运行在 127.0.0.1:8069 

# 二 安装Nginx

debian/ubuntu:

{% codeblock %}
# apt-get install nginx
{% endcodeblock %}

redhat/centos:

{% codeblock %}
# yum install nginx
{% endcodeblock %}

# 三 配置Nginx

## 1. 修改/etc/nginx/nginx.conf ，开启gzip 压缩

{% codeblock %}
# vi /etc/nginx/nginx.conf

--------------nginx.conf 需修改内容节选--------------------------
        gzip on;
        gzip_disable "msie6";

        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        #添加一个类型 application/javascript
        gzip_types text/plain text/css application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
{% endcodeblock %}

吐槽一下，是否开启gzip，差别真不小。oe 首页加载的http://127.0.0.1/web/webclient/js 开启前文件大小是 1.4M , 开启后大小是350.6 KB （通过firebug 查看）。

## 2. 建立 openerp 配置文件

{% codeblock %}
# touch /etc/nginx/sites-enabled/openerp
# vi /etc/nginx/sites-enabled/openerp

--------------------openerp 文件内容---------------------------

proxy_temp_path /tmp/nginx_proxy_temp;
proxy_cache_path  /tmp/nginx_proxy_cache levels=1:2  keys_zone=oecache:100m inactive=3d max_size=1000m;

proxy_buffer_size     32k;              #设置代理服务器（nginx）保存用户头信息的缓冲区大小
proxy_buffers         4 32k;            #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
proxy_busy_buffers_size  64k;           #高负荷下缓冲大小（proxy_buffers*2）
proxy_temp_file_write_size  64k;       #设定缓存文件夹大小，大于这个值，将从upstream服务器传

proxy_connect_timeout      60;
proxy_send_timeout         60;
proxy_read_timeout         3000;

upstream oeserver{
        server 127.0.0.1:8069;
}

server {

        server_name  www.example.com;

        root /var/www/openerp-6.1-1/openerp/addons;

        location /{

                proxy_cache              oecache;
                #proxy_cache_key "$host$request_uri$request_body";
                proxy_cache_key $host$request_uri$request_body;
                proxy_cache_valid  200 304 1d;
                proxy_cache_valid  any   1d;

                proxy_next_upstream http_502 http_504 error timeout invalid_header;
                proxy_pass_header Set-Cookie;
                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
                proxy_redirect  off;

                proxy_pass http://oeserver;

                proxy_buffering on;
                proxy_cache_valid       1d;
                expires 1d;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
                proxy_buffering on;
                proxy_cache_valid       1d;
                expires 1d;
        }

}
{% endcodeblock %}

# 完成 ！

Nginx 此处仅仅是作为 openerp 的前端WEB服务器，Nginx 还有更大的作用是可以实现Openerp 的负载平衡。此处按下不表，呵呵 


