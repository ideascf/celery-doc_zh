.. _guide-calling:

===============
 Calling Tasks
===============

.. contents::
    :local:
    :depth: 1


.. _calling-basics:

Basics
======

这篇文档描述了Celery的通用"Calling API"——被task实力
和:ref:`canvas <guide-canvas>`使用。

API定义了标准的执行选项集合以及三个方法：

    - ``apply_async(args[, kwargs[, …]])``

        发送一个task message。

    - ``delay(*args, **kwargs)``

        发送task message的快捷方式，但是不支持执行选项。

    - *calling* (``__call__``)

        Applying an object supporting the calling API (e.g. ``add(2, 2)``)
        意味着这个task会被当前进程执行，而不是被一个worker(即，不会发送message)。

.. _calling-cheat:

.. topic:: Quick Cheat Sheet

    - ``T.delay(arg, kwarg=value)``
        always a shortcut to ``.apply_async``.

    - ``T.apply_async((arg, ), {'kwarg': value})``

    - ``T.apply_async(countdown=10)``
        executes 10 seconds from now.

    - ``T.apply_async(eta=now + timedelta(seconds=10))``
        executes 10 seconds from now, specifed using ``eta``

    - ``T.apply_async(countdown=60, expires=120)``
        executes in one minute from now, but expires after 2 minutes.

    - ``T.apply_async(expires=now + timedelta(days=2))``
        expires in 2 days, set using :class:`~datetime.datetime`.


Example
-------

:meth:`~@Task.delay` 方法非常方便，因为它看起来像是一个普通的函数调用:
function:

.. code-block:: python

    task.delay(arg1, arg2, kwarg1='x', kwarg2='y')

而使用 :meth:`~@Task.apply_async` 你必须这样写:

.. code-block:: python

    task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})

.. sidebar:: Tip

    如果这个task在当前进程中没有被注册，你也可以使用 :meth:`~@send_task`
    通过这个task的名称去调用这个task。


所以`delay`无疑是方便的，除非你想设置扩展执行参数，这时你必须使用``apply_async``方法。

文档剩下的内容将详细地深入探究task的执行选项。
本文中的所有的例子使用一个名为`add`的task——接受两个参数并返回他们的和：

.. code-block:: python

    @app.task
    def add(x, y):
        return x + y


.. topic:: There's another way…

    当你阅读到:ref:`Canvas <guide-canvas>` 章节的时候，你将了解到更多关于这个知识。
    但是 :class:`~celery.subtask` 是用来传递一个task调用的signature
    (例如： 通过网络发送它)，并且它也支持这些 `Calling API` :

    .. code-block:: python

        task.s(arg1, arg2, kwarg1='x', kwargs2='y').apply_async()

.. _calling-links:

Linking (callbacks/errbacks)
============================

Celery支持将多个task链接在一起，以便一个task fllows 另一个task。
callback task将被调用——使用父task的结果作为局部参数（partial argument)：

.. code-block:: python

    add.apply_async((2, 2), link=add.s(16))

.. sidebar:: What is ``s``?

    这里使用的``add.s``被称为subtask，我将在 :ref:`canvas guide <guide-canvas>`
    章节详细的描述subtask以及 :class:`~celery.chain`——一种更简便的串联task的方式。

    实际上``link``执行选项是非常原始的，你可能根本不会直接用到它，
    而是使用 :class:`~celery.chain`。

这里，第一个task的结果(4)，将会被传递给下一个task用来和16相加。
形成这样的一个表达式 :math:`(2 + 2) + 16 = 20`


当task抛出异常时，也可能引起一个callback(*errback*)被调用，
但时errback的行为和正常的callback有些差异：它将传递父task的ID而不是结果作为参数。
这是因为，并不是所有抛出的异常都能够被序列化。并且使用这种errback，必须确保result backend被启用。
并且这个task(异常处理的task)，必须替父task取回结果。

这是一个使用error callback的例子:

.. code-block:: python

    @app.task(bind=True)
    def error_handler(self, uuid):
        result = self.app.AsyncResult(uuid)
        print('Task {0} raised exception: {1!r}\n{2!r}'.format(
              uuid, result.result, result.traceback))

it can be added to the task using the ``link_error`` execution
option:

.. code-block:: python

    add.apply_async((2, 2), link_error=error_handler.s())


另外，``link`` 和 ``link_error`` 选项都可以使用list::

    add.apply_async((2, 2), link=[add.s(16), other_task.s()])

callbacks/errbacks将会依序被调用，并且所有callbacks将会结合父task返回的结果作为局部参数被调用。

.. _calling-eta:

ETA and countdown
=================

ETA(estimated time of arrival) 允许你设置一个特定的日期时间
——task最早被执行的时间。`countdown` 是一个以秒为单位设置eta快捷方式。

.. code-block:: python

    >>> result = add.apply_async((2, 2), countdown=3)
    >>> result.get()    # this takes at least 3 seconds to return
    20


task会被确保在指定的日期和时间*之后*被执行，但不一定会在那个精确的时刻被执行。
导致打破deadline的原因包括：有大量的元素在队列中等待、严重的网络延时。
为了保证你的task及时的被执行，你应该监控队列的拥塞程度。
使用 `Munin` 或者其他类似的工具监听警告，以便能够采取恰当的动作去减轻负载。
详情参见 :ref:`monitoring-munin`.

虽然 `countdown` 是一个整数，但是 `eta` 必须为一个 :class:`~datetime.datetime` 对象
——指定一个确切的日期和时间（包含毫秒和时区信息）:

.. code-block:: python

    >>> from datetime import datetime, timedelta

    >>> tomorrow = datetime.utcnow() + timedelta(days=1)
    >>> add.apply_async((2, 2), eta=tomorrow)

.. _calling-expiration:

Expiration
==========

`expires` 参数定义了一optional的过期时间，它可以是一个距task publish时的秒数，
也可以是一个指定的 :class:`~datetime.datetime`:

.. code-block:: python

    >>> # Task expires after one minute from now.
    >>> add.apply_async((10, 10), expires=60)

    >>> # Also supports datetime
    >>> from datetime import datetime, timedelta
    >>> add.apply_async((10, 10), kwargs,
    ...                 expires=datetime.now() + timedelta(days=1)

当worker收到一个过期的 `task` 时，`worker` 会标志这个`task`
为 :state:`REVOKED` (:exc:`~@TaskRevokedError`).

.. _calling-retry:

Message Sending Retry
=====================

当连接失败时候，Celery将会自动重发message。并且重试的行为是可以配置的
，比如重试的间隔时间、重试的最大次数、亦或全部禁用。

要禁用重试，你可以设置执行参数 ``retry`` 为 :const:`False`:

.. code-block:: python

    add.apply_async((2, 2), retry=False)

.. topic:: Related Settings

    .. hlist::
        :columns: 2

        - :setting:`CELERY_TASK_PUBLISH_RETRY`
        - :setting:`CELERY_TASK_PUBLISH_RETRY_POLICY`

Retry Policy
------------

控制重试行为的重试策略是一个字典，它包含了如下keys：

- `max_retries`

    放弃重试之前的最大重试次数，这种情况下会抛出导致这次重试的异常。

    如果这个值被设置为0 或者 :const:`None` ，将一直重试下去。

    默认值是重试3次。

- `interval_start`

    两次重试之间的*初始*间隔时间。单位：秒(int 或 float)
    默认值是0，意味着第一次重试将会立即执行。

- `interval_step`

    每一次重试之后，这个值将会加到重试延时(译者注：interval_start)上。
    默认值是0.2。单位：秒(int 或 float)

- `interval_max`

    两次重试之间的最大间隔时间。默认值是0.2。单位：秒(int 或 float)

默认的重试策略等价于:

.. code-block:: python

    add.apply_async((2, 2), retry=True, retry_policy={
        'max_retries': 3,
        'interval_start': 0,
        'interval_step': 0.2,
        'interval_max': 0.2,
    })


用来重试这个task的耗时最多0.4秒（译者注：3次重试的总耗时， 0， 0.2， 0.2）。
这个值默认被设置的相对较小，因为如果broker连接down掉引起了链接失败，就可能导致重试大量堆积。
比如：大量的web服务器进程等待重试，从而阻塞了其它请求。

.. _calling-serializers:

Serializers
===========

.. sidebar::  Security

    The pickle module allows for execution of arbitrary functions,
    please see the :ref:`security guide <guide-security>`.

    Celery also comes with a special serializer that uses
    cryptography to sign your messages.

在客户端和worker之间传递的数据需要被序列化，所以Celery中的每个message都包括一个 ``content_type``
—— 描述编码这个message的序列化方法。

默认的序列化工具是 :mod:`pickle`，但是你可以通过设置 :setting:`CELERY_TASK_SERIALIZER` 来改变默认的序列化工具；
或者为每一个单独的task，甚至为每个消息特别设置。

内置支持的序列化工具有：:mod:`pickle`, `JSON`, `YAML` 以及 `msgpack`。
当然你可以添加你自定义的序列化工具——注册它们到 `kombu` 的序列化注册表中
（详见:`kombu:guide-serialization`）。

每个选项都有各自的优缺点：


json -- JSON被大多数的变成语言支持，并且是Python标准库的一部分(自 2.6以后)，
    并且使用现代的Python库拥有不错的性能,例如： :mod:`cjson` or :mod:`simplejson`。

    JSON主要的缺点是：它限制你只能使用如下的一些数据类型：
    strings、unicode、floats、boolean、dictionaries以及lists。
    Decimals 和 Dates 不被支持。

    并且，二进制数据将使用Base64编码转换，这将导致传输的数据量相比原生支持二进制数据的序列化方式增加34%。

    然而，如果你的数据符合前面提到的一些限制并且你需要跨语言的支持，*默认的*配置JSON可能时你最好的选择。

    See http://json.org for more information.


pickle -- 如果你不想支持出Python以外的任何语言，使用pickle编码将赋予你如下好处：
    支持所有Python内建的数据类型(除了类)、当发送一个二进制文件并且时更小的messages、
    相对于JSON稍微的性能提升。

    See http://docs.python.org/library/pickle.html for more information.


yaml -- YAML拥有很多和JSON类似的特性，除了它原生支持更多的数据类型（包括： 日期、循环引用等）

    然而，Python的YAML库会比JSON库的执行效率稍微慢一点。

    如果你需要更多的表达能力并且需要保持跨平台的能力，那么YAML可能比上面的更加适合。

    See http://yaml.org/ for more information.


msgpack -- msgpack是一个二进制序列化格式——在特性上更接近与JSON。
    然而它仍然非常的新，并且选择应该当做实验用途。

    See http://msgpack.org/ for more information.

使用的编码方式会作为message header的一部分，所以`worker`知道如何区反序列化任意`task`。
如果你使用一个自定义序列化工具，这个序列化工具必须同时在`worker`上可用。

将按照下面列出的顺序，去选择发送task时使用序列化方案：

    1. `serializer` 执行选项(译者注： 调用这个task时候传入)
    2. :attr:`@-Task.serializer` 属性 (译者注： 创建这个task时候的选项)
    3. :setting:`CELERY_TASK_SERIALIZER` 设置


例如,在调用task的时候指定序列化方案::

.. code-block:: python

    >>> add.apply_async((10, 10), serializer='json')

.. _calling-compression:

Compression
===========

Celery可以压缩消息，可选的方法有：*gzip*、*bzip2*。你也可以创建自定义的压缩方案，
然后注册它们到 :func:`kombu compression registry <kombu.compression.register>`中。

将按照下面列出的顺序，去选择发送task时使用压缩方案：

    1. `compression` 执行选项(译者注： 调用这个task时候传入)
    2. :attr:`@-Task.compression` 属性 (译者注： 创建这个task时候的选项)
    3. :setting:`CELERY_MESSAGE_COMPRESSION` 设置


例如,在调用task的时候指定压缩方案::

    >>> add.apply_async((2, 2), compression='zlib')

.. _calling-connections:

Connections
===========

.. sidebar:: Automatic Pool Support

    从版本2.3开始，支持自动连接池。所以你不必为了重用连接而去手动的处理连接和publisher。

    从2.5开始，连接池默认被启用。

    参见 :setting:`BROKER_POOL_LIMIT` 设置，获得更多信息。

你可以通过创建一个publisher来手动的处理连接：

.. code-block:: python


    results = []
    with add.app.pool.acquire(block=True) as connection:
        with add.get_publisher(connection) as publisher:
            try:
                for args in numbers:
                    res = add.apply_async((2, 2), publisher=publisher)
                    results.append(res)
    print([res.get() for res in results])


不过这个特别例子中可以使用group来更好的描述：

.. code-block:: python

    >>> from celery import group

    >>> numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
    >>> res = group(add.s(i) for i in numbers).apply_async()

    >>> res.get()
    [4, 8, 16, 32]

.. _calling-routing:

Routing options
===============

Celery可以路由task到不同的`queue`中。

使用执行选项 ``queue`` 来做简单的路由选择（name <-> name）是成熟的::

    add.apply_async(queue='priority.high')

你可以使用`worker`的 :option:`-Q` 参数，来将 `worker`分配到这个``priority.high``队列：

.. code-block:: bash

    $ celery -A proj worker -l info -Q celery,priority.high

.. seealso::

    在代码里面硬编码的队列名称是不推荐的做法，
    最佳实践是使用 *配置路由器* 配置选项(:setting:`CELERY_ROUTES`)。

    详情参见 :ref:`guide-routing`.

Advanced Options
----------------

接下来的这些选项是针对高级用户——想使用AMQP的全部路由功能。
感兴趣的可以阅读 :ref:`routing guide <guide-routing>` 了解更多。

- exchange

    Name of exchange (or a :class:`kombu.entity.Exchange`) to
    send the message to.

- routing_key

    用来决定路由的路由KEY
    Routing key used to determine.

- priority

    从`0`到`9`的数字，`0`是最高优先级。

    支持这个特性的`broker`： redis、beanstalk。
