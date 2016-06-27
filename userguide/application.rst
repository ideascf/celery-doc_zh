.. _guide-app:

=============
 Application
=============

.. contents::
    :local:
    :depth: 1

Celery库在使用前必须实例化，这个实例通常被叫做application(更简洁的叫做app)。

Celery的实例application是线程安全(thread-safe)的，所以同一个进程的多个Celery实例可以同时拥有不同的配置、组件、任务。

来吧，现在开始创建一个：

.. code-block:: python

    >>> from celery import Celery
    >>> app = Celery()
    >>> app
    <Celery __main__:0x100469fd0>

最后一行显示了这个application的文本描述/表现，这个文本描述中包含了celery的“类”(``Celery``)，当前main module的名称(``__main__``)，以及这个对象的内存地址(``0x100469fd0``)。

Main Name
=========

main module name，是以上信息中最重要的一条，让我们来瞅瞅为什么它是什么。

当你在Celery中发送一个任务消息时，这个消息中除了包含这个任务的名称以外，不会包含任何源码信息.
具体的工作机制和Internet中域名机制类似：每个职程(worker)都维护了一份被称为*注册表*(registry)，即任务名到函数的映射表。

无论何时你定义一个任务，这个任务将被加入到本地注册表中(local registry):

.. code-block:: python

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add
    <@task: __main__.add>

    >>> add.name
    __main__.add

    >>> app.tasks['__main__.add']
    <@task: __main__.add>

在这里，你将再次看到``__main__``无论何时，如果Celery不能探测到这个函数属于哪个模块，那么将会使用main module name，去生成这个任务名的前缀(beginning)

This is only a problem in a limited set of use cases:

    #. 如果这个定义这个任务的模块，被作为一个程序运行.
    #. 这个application在Python Shell中被创建。

比如这里，定义任务的模块，也用来启动一个职程(worker)
 :meth:`@worker_main`:

:file:`tasks.py`:

.. code-block:: python

    from celery import Celery
    app = Celery()

    @app.task
    def add(x, y): return x + y

    if __name__ == '__main__':
        app.worker_main()

如果定义任务的模块被执行执行，那么其中的任务将被以"``__main__``"为前缀命名；但是当这个模块被其它模块(process)导入，即调用一个任务，这些任务将被以"``tasks``"为前缀命名("tasks"是这个模块的名称)。::

    >>> from tasks import add
    >>> add.name
    tasks.add

当然，你也可以为main module指定一个其它的名称:

.. code-block:: python

    >>> app = Celery('tasks')
    >>> app.main
    'tasks'

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add.name
    tasks.add

.. seealso:: :ref:`task-names`

配置
===

Celery提供了几种可以改变Celery如何工作的配置选项。这些配置选项可以直接作用于Celery实例(application)，也可以使用专门的配置模块。

Celery.conf配置管理模块是可用的 :attr:`@conf`::

    >>> app.conf.CELERY_TIMEZONE
    'Europe/London'

你可以直接通过app.conf来设置配置::

    >>> app.conf.CELERY_ENABLE_UTC = True

和使用``update``方法，来一次更新多个配置::

    >>> app.conf.update(
    ...     CELERY_ENABLE_UTC=True,
    ...     CELERY_TIMEZONE='Europe/London',
    ...)

这个配置对象由多个字典组成，他们的生效顺序为:

    #. 运行中作出的改变.
    #. 配置模块(如果存在)
    #. 默认的配置 (:mod:`celery.app.defaults`).

你也可以使用 
:meth:`@add_defaults`
方法，来添加一个新的默认sources.

.. seealso::

    Go to the :ref:`Configuration reference <configuration>` for a complete
    listing of all the available settings, and their default values.

``config_from_object``
----------------------

:meth:`@config_from_object` 方法从一个配置对象中加载配置。

配置对象可以是： 模块、任何包含配置属性的对象。

注意：在 :meth:`~@config_from_object` 函数调用之前，作出的所有配置将会被清除。如果你想的是增加配置(译者注：而不是重新配置)，你应该先调用config_from_object()。

样例1：使用模块的名字
~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    from celery import Celery

    app = Celery()
    app.config_from_object('celeryconfig')


The ``celeryconfig`` module may then look like this:

:file:`celeryconfig.py`:

.. code-block:: python

    CELERY_ENABLE_UTC = True
    CELERY_TIMEZONE = 'Europe/London'

样例2：使用配置模块
~~~~~~~~~~~~~~~~~~

.. tip::

    使用模块的名字来加载配置是更推荐的方法，因为这样当你使用prefork pool的时候，不必做额外的序列化。If you’re experiencing configuration pickle errors then please try using the name of a module instead.

.. code-block:: python

    from celery import Celery

    app = Celery()
    import celeryconfig
    app.config_from_object(celeryconfig)

样例3：使用配置类/对象
~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    from celery import Celery

    app = Celery()

    class Config:
        CELERY_ENABLE_UTC = True
        CELERY_TIMEZONE = 'Europe/London'

    app.config_from_object(Config)
    # or using the fully qualified name of the object:
    #   app.config_from_object('module:Config')

``config_from_envvar``
----------------------

:meth:`@config_from_envvar`  函数将从一个环境变量中获取配置模块的名字(译者注：不是从环境变量中获取配置选项，而是获取配置模块的名字)。

比如——从环境变量:envvar:`CELERY_CONFIG_MODULE`指定模块名字的模块中加载配置:

.. code-block:: python

    import os
    from celery import Celery

    #: Set default configuration module name
    os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

    app = Celery()
    app.config_from_envvar('CELERY_CONFIG_MODULE')

You can then specify the configuration module to use via the environment:

.. code-block:: bash

    $ CELERY_CONFIG_MODULE="celeryconfig.prod" celery worker -l info

.. _app-censored-config:

Censored configuration
----------------------

如果你想打印配置，以作为调试信息或类似的目的，你可能想过滤掉类似密码和API 密钥的敏感信息。

Celery自带了一些工具用来呈现配置信息，其中一个是  :meth:`~celery.app.utils.Settings.humanize`  :

.. code-block:: python

    >>> app.conf.humanize(with_defaults=False, censored=True)

这个函数将以制表后的字符串形式，返回配置信息。默认情况下只返回发生改变的配置选项，但你可以通过设置``with_defaults`` 参数为True，来包含默认的配置选项。

如果你想得到一个字典而不是字符串，你可以使用  :meth:`~celery.app.utils.Settings.table`  方法:

.. code-block:: python

    >>> app.conf.table(with_defaults=False, censored=True)

请小心，Celery不能移除所有的敏感信息，它仅仅是使用正则表达式去搜索匹配配置字典的key。如果你打算添加一些包含敏感信息的自定义配置，你应该按如下方法命名他们的KEY，以便于Celery把他们标记为敏感信息.

如果配置选项的KEY中包含如下字符串，配置信息将被consored

``API``, ``TOKEN``, ``KEY``, ``SECRET``, ``PASS``, ``SIGNATURE``, ``DATABASE``

惰性
===

Celery的application实例是惰性的，意味着它将不被计算直到真正需要被计算。

创建一个:class:`@Celery`实例，仅仅完成以下几件事:

    #. 创建一个逻辑clock实例，用于events.
    #. 创建一个任务注册表(registry).
    #. 这只自身为当前的app(current app).（除非set_as_current参数被设置为``set_as_current``）
    #. 调用  :meth:`@on_init`  回调函数(默认情况什么也不做)

调用  :meth:`@task`  装饰器不会立即创建一个任务，创建操作将推迟到这个任务被使用的时候，或者这个application被*finalized*.

接下来的这个示例，将展示任务不会被创建，直到开始使用这个任务，或者访问一个属性(这里是调用  :meth:`repr`  ):

.. code-block:: python

    >>> @app.task
    >>> def add(x, y):
    ...    return x + y

    >>> type(add)
    <class 'celery.local.PromiseProxy'>

    >>> add.__evaluated__()
    False

    >>> add        # <-- causes repr(add) to happen
    <@task: __main__.add>

    >>> add.__evaluated__()
    True

application的*Finalization*发生在明确的  :meth:`@finalize`  调用，或因为访问application的  :attr:`@tasks`  属性时，隐式的被调用。

Finalizing 这个对象将会:

    #. 拷贝必须被多个application共享的任务

        任务默认是被分享的，除非在调用task装饰器的时``shared``参数为False，这样这个任务将成为这个application的私有的。

    #. 计算所有被挂起的任务装饰器(译者注：上文中提到task装饰器，并不会立即创建一个任务)。

    #. 确保所有任务被绑定(bound to)到当前application(current app)

        任务被绑定到application，以让application能够从配置中读取默认值。
        Tasks are bound to apps so that it can read default
        values from the configuration.

.. _default-app:

.. topic:: The "default app".

    Celery 并不总是这样工作，"default app"被使用，是因为存在一些模块级别的API，以及向后兼容老版本API的原因。

    Celery始终创建一个特殊的application，即“default app”，当使用者没有创建application时被使用。

    :mod:`celery.task`模块是为了适应老版本的API而提供的，如果你创建了application，那么请不要使用这个模块。你应该一直使用application实例的方法，而不是基于模块(译者注：celery.task)的方法。

    例如，老版本的“Task”基类允许使用大量兼容性的特性，这些特性可能和新特性不兼容，比如任务的方法:

    .. code-block:: python

        from celery.task import Task   # << OLD Task base class.

        from celery import Task        # << NEW base class.

    新版本的基类是推荐使用的，即使你正在使用老版本的基于模块的API.


Breaking the chain
==================

虽然可以通过使用current app来实现，但是最好的实践是：始终传递application实例到任何一个需要它的地方。

我称之为“app chain”，因为在向这些依赖于application实例的模块传递application时，它"创建"了这样的一条链。

接下来的样例，我认为是的糟糕实践:

.. code-block:: python

    from celery import current_app

    class Scheduler(object):

        def run(self):
            app = current_app

取而代之的是让 ``app`` 成为一个参数：

.. code-block:: python

    class Scheduler(object):

        def __init__(self, app):
            self.app = app

Celery内部使用  :func:`celery.app.app_or_default`  函数，以便于everything也可以工作于基于模块的兼容API(译者注：为了兼容老版本API)。

.. code-block:: python

    from celery.app import app_or_default

    class Scheduler(object):
        def __init__(self, app=None):
            self.app = app_or_default(app)

在开发的过程中，你可以设置环境变量  :envvar:`CELERY_TRACE_APP`  ，让Celery抛出一个异常，当这个"app chain"被打破的时候：

.. code-block:: bash

    $ CELERY_TRACE_APP=1 celery worker -l info


.. topic:: Evolving the API

    自从Celery被创建以来的3年，Celery发生了大量的改变。

    比如，在刚刚开始的时候，你可以使用任何callable的对象作为一个任务：

    .. code-block:: python

        def hello(to):
            return 'hello {0}'.format(to)

        >>> from celery.execute import apply_async

        >>> apply_async(hello, ('world!', ))

    或者，你可以创建一个``Task``类去设置一些必须的选项，或者覆盖其它行为：

    .. code-block:: python

        from celery.task import Task
        from celery.registry import tasks

        class Hello(Task):
            send_error_emails = True

            def run(self, to):
                return 'hello {0}'.format(to)
        tasks.register(Hello)

        >>> Hello.delay('world!')

    后来，显而易见，允许传递任意的callable对象是反模式(anti-pattern)，因为这样让使用非pickle序列化方案去序列化变得非常困难，所以这个特性在2.0的时候被移除了，取而代之的是task装饰器：

    .. code-block:: python

        from celery.task import task

        @task(send_error_emails=True)
        def hello(x):
            return 'hello {0}'.format(to)

Abstract Tasks
==============

所有被  :meth:`~@task`  装饰器创建的任务，将继承于  :attr:`~@Task`  类。

你可以通过使用``base``参数，来指定一个不同的基类：

.. code-block:: python

    @app.task(base=OtherTask):
    def add(x, y):
        return x + y

创建一个自定义的task类，应该从继承与neutral 基类,即  :class:`celery.Task`  (译者注：不要使用application.Task，即实例中的Task类)：

.. code-block:: python

    from celery import Task

    class DebugTask(Task):
        abstract = True

        def __call__(self, *args, **kwargs):
            print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
            return super(DebugTask, self).__call__(*args, **kwargs)


.. tip::

    如果你重载了task类的``__call__``方法，调用基类的``__call__``方法是非常重要的，因为基类的``__call__``方法将建立任务被直接调用时使用的默认请求。


neutral 基类是特殊的，因为它不属于(bound to)任何一个application。Concrete subclasses of this class will be bound, so you should always mark generic base classes as abstract。

一旦task被bound to一个application，它(task)将读取配置选项去设置默认值等等。

改变一个application的默认基类是可能的，可以通过改变  :meth:`@Task`  属性来实现：

.. code-block:: python

    >>> from celery import Celery, Task

    >>> app = Celery()

    >>> class MyBaseTask(Task):
    ...    abstract = True
    ...    send_error_emails = True

    >>> app.Task = MyBaseTask
    >>> app.Task
    <unbound MyBaseTask>

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add
    <@task: __main__.add>

    >>> add.__class__.mro()
    [<class add of <Celery __main__:0x1012b4410>>,
     <unbound MyBaseTask>,
     <unbound Task>,
     <type 'object'>]
