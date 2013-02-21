---
layout: post
title: "OpenERP 负载平衡"
date: 2013-01-21 15:11
comments: true
categories: OpenERP
---


 OpenERP 7.0 带来了许多新特性，架构上也有许多改进。其中可配置 worker 参数，可使 OpenERP 运行在多进程模式，突破GIL的限制，有效利用了现代多核CPU的性能。但默认情况下，OpenERP 只能运行于一台服务器，对于提供SAAS服务或并发很大的情况下，单台服务器的性能是有限的。本文介绍实现 OpenERP 负载平衡的方法和原理。

一、架构

    ┌──────────────────────────────────────────────┐
    │                   Nginx                      │
    └──────────────────────────────────────────────┘
            /              |                \
    ┌────────────┐   ┌────────────┐    ┌────────────┐
    │ OE Server  │   │ OE Server  │    │ OE Server  │
    └────────────┘   └────────────┘    └────────────┘
             \             |                /
    ┌───────────────────────────────────────────────┐
    │                Redis Server                   │
    └───────────────────────────────────────────────┘

注：实现负载平衡的关键点在于 cache 和 session 共享。

二、Web 服务器配置

WEB 服务器选择 Nginx + upstream 配置，可参考 “使用Nginx Upstream 部署 OpenERP ” http://my.oschina.net/wangbuke/blog/67450 。

默认情况下，nginx 采用轮询的方式，将请求分发到多个 OE Server 里。建议改为 ip_hash 方式，如：

    upstream bakend {
         ip_hash;
         server 192.168.0.11:8069;
         server 192.168.0.12:8069;
    }

三、OpenERP 的 Session 和 Cache 处理

3.1 OpenERP Web Session 处理

OpenERP 中的Session 处理默认用FilesystemSessionStore，使用文件系统存储用户 session  。如 openerp/addons/web/http.py

{% codeblock lang:python %}
class Root(object):
   
    def __init__(self):
  
        self.session_store = werkzeug.contrib.sessions.FilesystemSessionStore(path)
        self.session_lock = threading.Lock()

{% endcodeblock %}

那么我们只要将OpenERP 中的SessionStore，改为 RedisSessionStore，RedisSessionStore 可参考https://gist.github.com/1451947 。

修改方法，可以直接修改 http.py 文件。
或是和我一样，重写一个库，重载 session_context 方法，这样可以不修改OpenERP的源文件，方便以后升级。

3.2 OpenERP LRU Cache 处理

openerp/tools/cache.py 中 ormcache 和 ormcache_multi 是 OpenERP 中非常重要的缓存类。OpenERP ORM 大部分的方法调用都会经过 @tools.ormcache 或 @ormcache_multi 修饰。经过修饰后，结果会被缓存，这个缓存是存放于内存中。 这个就是OE在加载一次数据后，第二次会明显快很多的原因。还有，通过web 界面翻译OE术语不能实时生效，也是因为缓存没有更新。

可以修改ormcache 和 ormcache_multi 类，以使用 redis 缓存。关键代码如下：

{% codeblock lang:python %}
    def lookup(self, self2, cr, *args):
        key = args[self.skiparg-2:]
        key = '%s:%s' % (self.method.__name__, str(key))
        #key = md5(key).hexdigest()
        hash_name = self.db_key_template % cr.dbname
        value = self.redis.hget(hash_name, key)
        if value:
            self.stat_hit += 1
            return loads(value)
        else:
            self.stat_miss += 1
            value = self.method(self2, cr, *args)
            self.redis.hset(hash_name, key, dumps(value, HIGHEST_PROTOCOL))
            self.redis.expire(hash_name, self.timeout)
            return value

{% endcodeblock %}

缓存的值使用 cPickle 序列化后，将每个键值对存放于 redis 的 哈希表中。

3.3 auth_openid 模块

auth_openid模块也使用文件系统存储用户登录凭证。如：

{% codeblock lang:python %}
class OpenIDController(openerp.addons.web.http.Controller):

    _store = filestore.FileOpenIDStore(_storedir)

{% endcodeblock %}
如果您启用了这个模块，那么这里也需要修改为存储在redis中。如果没有启用此模块，则无需理会。

相关实现可参考，https://github.com/bbangert/openid-redis/blob/master/openidredis/__init__.py

四、OpenERP Cron 处理

默认情况下，每个OpenERP Server 实例都会运行一个 cron 进程任务。这里建议只允许一个实例运行CRON。把OpenERP 7.0 的配置参数 max_cron_threads 设置为0 ，即可禁止cron。相关代码如下：

{% codeblock lang:python %}
    def process_spawn(self):
        while len(self.workers_http) < self.population:
            self.worker_spawn(WorkerHTTP, self.workers_http)
        while len(self.workers_cron) < config['max_cron_threads']:
            self.worker_spawn(WorkerCron, self.workers_cron)

{% endcodeblock %}

五、OpenERP Module RegistryManager 处理

OpenERP Module Registry 主要负责管理OE的对象。一般是安装或更新的模块时候，会根据定义来更新数据库。 在OE多进程模式下，OE会自动管理 Module Registry ，相关的更新信息会存放在数据库里。RegistryManager  会检测是否有更新，如有更新将会自动清除缓存并重新载入。相关代码如下：

{% codeblock lang:python %}
    @classmethod
    def setup_multi_process_signaling(cls, cr):
        if not openerp.multi_process:
            return

    @classmethod
    def check_registry_signaling(cls, db_name):
        if openerp.multi_process and db_name in cls.registries:

{% endcodeblock %}

这里，实际上无需做改动,上面只是说明情况。只需让OE运行在多进程模式即可（也就是配置 worker 参数）。

六、完成！

经过以上几个步骤，可以让OpenERP 运行于多台服务器，通过Redis 分布式缓存处理相关的 Cache 和 Session，从而实现 OpenERP 负载平衡。

注：
1、本文仅讨论 OpenERP 负载平衡部署方式，并不涉及 Postgresql 和 Redis 的负载平衡，相应的方法请自行搜索。
2、鉴于OpenERP SA 官方已不再维护 GTK 客户端，并没有对GTK客户端的情况进行完整测试。





