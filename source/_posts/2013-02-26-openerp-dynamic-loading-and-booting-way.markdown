---
layout: post
title: "OpenERP 模块动态加载原理及启动代码分析"
date: 2013-02-26 15:48
comments: true
categories: OpenERP
---

一般来说我们在编程中，对象定义都是预先定义好的。一些 OOP 语言（包括 Python/Java）允许对象是 自省的（也称为 反射）。即，自省对象能够描述自己：实例属于哪个类？类有哪些祖先？对象可以用哪些方法和属性？自省让处理对象的函数或方法根据传递给函数或方法的对象类型来做决定。即允许对象在运行时动态改变方法成员等属性。

得益于OpenERP ORM 模型的精巧设计，实际上 OpenERP 运行时也是动态读取模块信息并动态构建对象的。如在模块开发中，继承了 'res.users', 新增一个方法或新增一个字段。在OpenERP 导入该模块后， OpenERP 会马上重构 'res.users' 对象并将新增的方法或字段添加到该对象。那么，OpenERP 是如何做到这点的呢？ 让我们从OpenERP 的启动部分开始分析：

首先，OpenERP 启动相关的服务, 这时并没有建立数据库链接和载入对象

{% codeblock lang:python %}
    if not config["stop_after_init"]:
        setup_pid_file()
        # Some module register themselves when they are loaded so we need the
        # services to be running before loading any registry.
        if config['workers']:
            openerp.service.start_services_workers()
        else:
            openerp.service.start_services()

{% endcodeblock %}

不过可以在配置文件中指定 'db_name' 参数，可以让 OpenERP 在启动时加载指定数据库的对象并启动 Cron。 实际生产环境中建议启用该参数，否则需要在启动OpenERP后，登录一次OpenERP 在会加载对象并启动CRON

{% codeblock lang:python %}
    if config['db_name']:
        for dbname in config['db_name'].split(','):
            preload_registry(dbname)
{% endcodeblock %}

不指定 'db_name' 参数情况下，OpenERP 会直到用户登录时才会初始化指定数据库。
{% codeblock lang:python %}
def login(db, login, password):
    pool = pooler.get_pool(db)
    user_obj = pool.get('res.users')
    return user_obj.login(db, login, password)
{% endcodeblock %}

打开 get_db 和 get_db_and_pool 定义
{% codeblock lang:python %}
def get_db(db_name):
    """Return a database connection. The corresponding registry is initialized."""
    return get_db_and_pool(db_name)[0]


def get_db_and_pool(db_name, force_demo=False, status=None, update_module=False):
    """Create and return a database connection and a newly initialized registry."""
    registry = RegistryManager.get(db_name, force_demo, status, update_module)
    return registry.db, registry

{% endcodeblock %}

顺藤摸瓜，看RegistryManager.new 定义
{% codeblock lang:python %}
            try:
                # This should be a method on Registry
                openerp.modules.load_modules(registry.db, force_demo, status, update_module)
            except Exception:
                del cls.registries[db_name]
                raise
{% endcodeblock %}

然后看到 load_modules, 终于要加载了，嘿嘿。

{% codeblock lang:python %}

        # STEP 1: LOAD BASE (must be done before module dependencies can be computed for later steps) 
        graph = openerp.modules.graph.Graph()
        graph.add_module(cr, 'base', force)
        if not graph:
            _logger.critical('module base cannot be loaded! (hint: verify addons-path)')
            raise osv.osv.except_osv(_('Could not load base module'), _('module base cannot be loaded! (hint: verify addons-path)'))

        # processed_modules: for cleanup step after install
        # loaded_modules: to avoid double loading
        report = pool._assertion_report
        loaded_modules, processed_modules = load_module_graph(cr, graph, status, perform_checks=update_module, report=report)

        if tools.config['load_language']:
            for lang in tools.config['load_language'].split(','):
                tools.load_language(cr, lang)

        # STEP 2: Mark other modules to be loaded/updated
        if update_module:
            modobj = pool.get('ir.module.module')
            if ('base' in tools.config['init']) or ('base' in tools.config['update']):
                _logger.info('updating modules list')
                modobj.update_list(cr, SUPERUSER_ID)

            _check_module_names(cr, itertools.chain(tools.config['init'].keys(), tools.config['update'].keys()))

            mods = [k for k in tools.config['init'] if tools.config['init'][k]]
            if mods:
                ids = modobj.search(cr, SUPERUSER_ID, ['&', ('state', '=', 'uninstalled'), ('name', 'in', mods)])
                if ids:
                    modobj.button_install(cr, SUPERUSER_ID, ids)

            mods = [k for k in tools.config['update'] if tools.config['update'][k]]
            if mods:
                ids = modobj.search(cr, SUPERUSER_ID, ['&', ('state', '=', 'installed'), ('name', 'in', mods)])
                if ids:
                    modobj.button_upgrade(cr, SUPERUSER_ID, ids)

            cr.execute("update ir_module_module set state=%s where name=%s", ('installed', 'base'))


        # STEP 3: Load marked modules (skipping base which was done in STEP 1)
        # IMPORTANT: this is done in two parts, first loading all installed or
        #            partially installed modules (i.e. installed/to upgrade), to
        #            offer a consistent system to the second part: installing
        #            newly selected modules.
        #            We include the modules 'to remove' in the first step, because
        #            they are part of the "currently installed" modules. They will
        #            be dropped in STEP 6 later, before restarting the loading
        #            process.
        states_to_load = ['installed', 'to upgrade', 'to remove']
        processed = load_marked_modules(cr, graph, states_to_load, force, status, report, loaded_modules, update_module)
        processed_modules.extend(processed)
        if update_module:
            states_to_load = ['to install']
            processed = load_marked_modules(cr, graph, states_to_load, force, status, report, loaded_modules, update_module)
            processed_modules.extend(processed)

{% endcodeblock %}

这里，第一步是加载核心模块 ['base']，第二步加载需要升级或预载的模块，第三步加载已安装的模块。实际加载语句是：

{% codeblock lang:python %}
        processed = load_marked_modules(cr, graph, states_to_load, force, status, report, loaded_modules, update_module)
{% endcodeblock %}

查看 load_marked_module 定义：
{% codeblock lang:python %}
def load_marked_modules(cr, graph, states, force, progressdict, report, loaded_modules, perform_checks):
    """Loads modules marked with ``states``, adding them to ``graph`` and
       ``loaded_modules`` and returns a list of installed/upgraded modules."""
    processed_modules = []
    while True:
        cr.execute("SELECT name from ir_module_module WHERE state IN %s" ,(tuple(states),))
        module_list = [name for (name,) in cr.fetchall() if name not in graph]
        graph.add_modules(cr, module_list, force)
        _logger.debug('Updating graph with %d more modules', len(module_list))
        loaded, processed = load_module_graph(cr, graph, progressdict, report=report, skip_modules=loaded_modules, perform_checks=perform_checks)
        processed_modules.extend(processed)
        loaded_modules.extend(loaded)
        if not processed: break
    return processed_modules
{% endcodeblock %}

重点是 load_module_graph 

{% codeblock lang:python %}

    # register, instantiate and initialize models for each modules
    for index, package in enumerate(graph):
        module_name = package.name
        module_id = package.id

        if skip_modules and module_name in skip_modules:
            continue

        _logger.debug('module %s: loading objects', package.name)
        migrations.migrate_module(package, 'pre')
        load_openerp_module(package.name)

        models = pool.load(cr, package)

        loaded_modules.append(package.name)
        if hasattr(package, 'init') or hasattr(package, 'update') or package.state in ('to install', 'to upgrade'):
            init_module_models(cr, package.name, models)
        pool._init_modules.add(package.name)
        status['progress'] = float(index) / len(graph)

{% endcodeblock %}

主要是下面2句，

{% codeblock lang:python %}
        load_openerp_module(package.name)

        models = pool.load(cr, package)

{% endcodeblock %}

先看 load_openerp_module 

{% codeblock lang:python %}
    initialize_sys_path()
    try:
        mod_path = get_module_path(module_name)
        zip_mod_path = '' if not mod_path else mod_path + '.zip'
        if not os.path.isfile(zip_mod_path):
            __import__('openerp.addons.' + module_name)
        else:
            zimp = zipimport.zipimporter(zip_mod_path)
            zimp.load_module(module_name)
{% endcodeblock %}

上面代码中 import 了一个模块。如果您看过 digitalsatori 校长的大作 [OpenERP与Python 元编程](http://shine-it.net/index.php/topic,5771.msg14289.html), 下面就涉及到元类了：

 import 的时候 就会调用元类的构造函数

{% codeblock lang:python %}
    def __init__(self, name, bases, attrs):
        if not self._register:
            self._register = True
            super(MetaModel, self).__init__(name, bases, attrs)
            return

        # The (OpenERP) module name can be in the `openerp.addons` namespace
        # or not. For instance module `sale` can be imported as
        # `openerp.addons.sale` (the good way) or `sale` (for backward
        # compatibility).
        module_parts = self.__module__.split('.')
        if len(module_parts) > 2 and module_parts[0] == 'openerp' and \
            module_parts[1] == 'addons':
            module_name = self.__module__.split('.')[2]
        else:
            module_name = self.__module__.split('.')[0]
        if not hasattr(self, '_module'):
            self._module = module_name

        # Remember which models to instanciate for this module.
        if not self._custom:
            self.module_to_models.setdefault(self._module, []).append(self)

{% endcodeblock %}

上面的代码基本上就是将自身类加入到 module_to_models 字典中。

然后我们来看pool.load 
{% codeblock lang:python %}
    def load(self, cr, module):
        """ Load a given module in the registry.

        At the Python level, the modules are already loaded, but not yet on a
        per-registry level. This method populates a registry with the given
        modules, i.e. it instanciates all the classes of a the given module
        and registers them in the registry.

        """
        models_to_load = [] # need to preserve loading order
        # Instantiate registered classes (via the MetaModel automatic discovery
        # or via explicit constructor call), and add them to the pool.
        for cls in openerp.osv.orm.MetaModel.module_to_models.get(module.name, []):
            # models register themselves in self.models
            model = cls.create_instance(self, cr)
            if model._name not in models_to_load:
                # avoid double-loading models whose declaration is split
                models_to_load.append(model._name)
        return [self.models[m] for m in models_to_load]

{% endcodeblock %}

这里我们可以看到 MetaModel 的身影，cls.create_instance(self, cr) 这里就是动态构造对象的核心代码。

{% codeblock lang:python %}

        parent_names = getattr(cls, '_inherit', None)
        # 判断是否有继承父类 
        if parent_names:
            if isinstance(parent_names, (str, unicode)):
                name = cls._name or parent_names
                parent_names = [parent_names]
            else:
                name = cls._name
            if not name:
                raise TypeError('_name is mandatory in case of multiple inheritance')

            for parent_name in ((type(parent_names)==list) and parent_names or [parent_names]):
                # 读取父类
                parent_model = pool.get(parent_name)
                if not parent_model:
                    raise TypeError('The model "%s" specifies an unexisting parent class "%s"\n'
                        'You may need to add a dependency on the parent class\' module.' % (name, parent_name))
                if not getattr(cls, '_original_module', None) and name == parent_model._name:
                    cls._original_module = parent_model._original_module
                parent_class = parent_model.__class__
                nattr = {}
                # 复制父类属性
                for s in attributes:
                    new = copy.copy(getattr(parent_model, s, {}))
                    if s == '_columns':
                        # Don't _inherit custom fields.
                        for c in new.keys():
                            if new[c].manual:
                                del new[c]
                        # Duplicate float fields because they have a .digits
                        # cache (which must be per-registry, not server-wide).
                        for c in new.keys():
                            if new[c]._type == 'float':
                                new[c] = copy.copy(new[c])
                    if hasattr(new, 'update'):
                        new.update(cls.__dict__.get(s, {}))
                    elif s=='_constraints':
                        for c in cls.__dict__.get(s, []):
                            exist = False
                            for c2 in range(len(new)):
                                #For _constraints, we should check field and methods as well
                                if new[c2][2]==c[2] and (new[c2][0] == c[0] \
                                        or getattr(new[c2][0],'__name__', True) == \
                                            getattr(c[0],'__name__', False)):
                                    # If new class defines a constraint with
                                    # same function name, we let it override
                                    # the old one.

                                    new[c2] = c
                                    exist = True
                                    break
                            if not exist:
                                new.append(c)
                    else:
                        new.extend(cls.__dict__.get(s, []))
                    nattr[s] = new

                # Keep links to non-inherited constraints, e.g. useful when exporting translations
                nattr['_local_constraints'] = cls.__dict__.get('_constraints', [])
                nattr['_local_sql_constraints'] = cls.__dict__.get('_sql_constraints', [])
                # 调用元类构造函数
                cls = type(name, (cls, parent_class), dict(nattr, _register=False))
        else:
            cls._local_constraints = getattr(cls, '_constraints', [])
            cls._local_sql_constraints = getattr(cls, '_sql_constraints', [])

        if not getattr(cls, '_original_module', None):
            cls._original_module = cls._module
        # 构造对象
        obj = object.__new__(cls)
        # 初始化对象
        obj.__init__(pool, cr)
        return obj
{% endcodeblock %}


上面代码很重要，可以看到首先是判断该对象是否有继承父类，如果没有就直接构造，和动态没有什么关系。

如果有继承父类, 就复制父类属性, 这里就是动态构建类的做法。

假如有不同模块，都继承了同一个父类，那么如何保证类成员和属性是否加载完整或覆盖呢？ 答案在于这句代码：

{% codeblock lang:python %}
                parent_model = pool.get(parent_name)
{% endcodeblock %}

Registry.get 的定义
{% codeblock lang:python %}
    def get(self, model_name):
        """ Return a model for a given name or None if it doesn't exist."""
        return self.models.get(model_name)
{% endcodeblock %}

最后看看obj.__init__(pool, cr)初始化对象，做了什么动作？
{% codeblock lang:python %}
    def __init__(self, pool, cr):
        """ Initialize a model and make it part of the given registry.

        - copy the stored fields' functions in the osv_pool,
        - update the _columns with the fields found in ir_model_fields,
        - ensure there is a many2one for each _inherits'd parent,
        - update the children's _columns,
        - give a chance to each field to initialize itself.

        """
        pool.add(self._name, self)
        self.pool = pool
{% endcodeblock %}

pool.add(self._name, self) 定义如下：
{% codeblock lang:python %}
    def add(self, model_name, model):
        """ Add or replace a model in the registry."""
        self.models[model_name] = model
{% endcodeblock %}

到这里应该很非常清楚，Registry.models 保存了对象的 model 信息。这样多个对象继承同一父类时，按照加载顺序先后动态构建相关的类。

至此，OpenERP 启动时动态加载模块分析完成。如模块安装、升级、卸载等, 则是通过 signal_registry_change 和 check_registry_signaling 处理，重新载入 Registry, 然后重新构建 OpenERP 对象。

