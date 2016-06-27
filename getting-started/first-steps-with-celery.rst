.. _tut-celery:
.. _first-steps:

=========================
Celery 初步
=========================

Celery 是一个“自带电池”的的任务队列。它易于使用，所以你可以无视
其所解决问题的复杂程度而轻松入门。它遵照最佳实践设计，所以你的
产品可以扩展，或与其他语言集成，并且它自带了在生产环境中运行这样
一个系统所需的工具和支持。

在此教程中，你会了解使用 Celery 的最基础部分。包括：

- 选择和安装消息传输方式（中间人）。
- 安装 Celery 并创建第一个任务
- 运行职程并调用任务。
- 追踪任务在不同状态间的迁移，并检视返回值。

Celery 起初可能令人却步——但不要担心——此教程会快速带你入门。此教程
刻意简化，所以不会让你在高级特性上困扰。在你完成了此教程的学习后，
阅读文档其余部分是个明智的选择，例如展示 Celery 能力的
:ref:`next-steps` 教程。

.. contents::
    :local:

.. _celerytut-broker:

选择中间人
=================

Celery 需要一个发送和接收消息的解决方案，其通常以独立服务形式出现，
称为 *消息中间人* 。

可行的选择包括：

RabbitMQ
--------

`RabbitMQ`_ 功能完备、稳定、耐用，并且安装简便，是生产环境的绝佳选择。
配合 Celery 使用 RabbitMQ 的详情见:

    :ref:`broker-rabbitmq`

.. _`RabbitMQ`: http://www.rabbitmq.com/

如果你使用 Ubuntu 或 Debian，可以执行这条命令来安装 RabbitMQ：

.. code-block:: bash

    $ sudo apt-get install rabbitmq-server

命令执行完成后，中间人就已经运行在后台，准备好传输消息：
``Starting rabbitmq-server: SUCCESS`` 。

如果你不使用 Ubuntu 或 Debian 也无须担心，你可以访问这个网站来寻找同样
简单的其他平台上（包括 Microsoft Windows）的安装指南：

    http://www.rabbitmq.com/download.html


Redis
-----

`Redis`_ 也是功能完备的，但更易受突然中断或断电带来数据丢失的影响。使用
Redis 的详细信息见：

    :ref:`broker-redis`

.. _`Redis`: http://redis.io/


使用数据库
----------------

不推荐把数据库用于消息队列，但对于很小的项目可能是合适的。你的选择包括：

* :ref:`broker-sqlalchemy`
* :ref:`broker-django`

例如如果你已经使用了 Django 的数据库后端，用它作为你的消息中间人在开发
时会很方便，即使在生产环境中你会采用更稳健的系统。


其他中间人
-------------

除了上面列出的之外，还有其他的实验性传输实现可供选择，包括
:ref:`Amazon SQS <broker-sqs>` 、 :ref:`broker-mongodb`
和 :ref:`IronMQ <broker-ironmq>` 。

完整列表见 :ref:`broker-overview` 。

.. _celerytut-installation:

安装 Celery
=================

Celery 提交到了 Python Package Index（PyPI）上，所以你可以用标准的
Python 工具，诸如 ``pip`` 或 ``easy_install`` 来安装：

.. code-block:: bash

    $ pip install celery

应用
===========

首先你需要一个 Celery 实例，称为 Celery 应用或直接简称应用。既然这个实例
用于你想在 Celery 中做一切事——比如创建任务、管理职程——的入口点，它必须
可以被其他模块导入。

在此教程中，你的一切都容纳在单一模块里，对于更大的项目，你会想创建
:ref:`独立模块 <project-layout>` 。

让我们创建 :file:`tasks.py` ：

.. code-block:: python

    from celery import Celery

    app = Celery('tasks', broker='amqp://guest@localhost//')

    @app.task
    def add(x, y):
        return x + y

:class:`~celery.app.Celery` 的第一个参数是当前模块的名称，这个参数是必须
的，这样的话名称可以自动生成。第二个参数是中间人关键字参数，指定你所使用
的消息中间人的 URL，此处使用了 RabbitMQ，也是默认的选项。更多可选的中间
人见上面的 :ref:`celerytut-broker` 一节。例如，对于 RabbitMQ 你可以写
``amqp://localhost`` ，而对于 Redis 你可以写 ``redis://localhost`` .

你定义了一个单一任务，称为 ``add`` ，返回两个数字的和。

.. _celerytut-running-the-worker:

运行 Celery 职程服务器
================================

你现在可以用 ``worker`` 参数执行我们的程序：

.. code-block:: bash

    $ celery -A tasks worker --loglevel=info

.. note::

    如果职程没有启动，请查阅 :ref:`celerytut-troubleshooting` 一节。

在盛传环境中你会想要让职程作为守护程序在后台运行。你需要用你所在平台提供
的工具来实现，或是像 `supervisord`_ 这样的东西（更多信息见
:ref:`daemonizing`）。


想要查看完整的命令行参数列表，如此：

.. code-block:: bash

    $  celery worker --help

也有几个其他的命令，帮助也是可用的：

.. code-block:: bash

    $ celery help

.. _`supervisord`: http://supervisord.org

.. _celerytut-calling:

调用任务
================

你可以用 :meth:`~@Task.delay` 方法来调用任务。

这是 :meth:`~@Task.apply_async` 方法的快捷方式，该方法允许你更
好地控制任务执行（见 :ref:`guide-calling` ）::

    >>> from tasks import add
    >>> add.delay(4, 4)

这个任务已经由之前启动的职程执行，并且你可以查看职程的控制台输出
来验证。

调用任务会返回一个 :class:`~@AsyncResult` 实例，可用于检查任务的
状态，等待任务完成或获取返回值（如果任务失败，则为异常和回溯）。
但这个功能默认是不开启的，你需要设置一个 Celery 的结果后端，下一
节将会详细介绍。

.. _celerytut-keeping-results:

保存结果
===============

如果你想要保持追踪任务的状态，Celery 需要在某个地方存储或发送这些
状态。可以从内建的几个结果后端选择：`SQLAlchemy`_/`Django`_ ORM、
`Memcached`_ 、 `Redis`_ 、 AMQP（ `RabbitMQ`_ ）或 `MongoDB`_ ，
或者你可以自制。

.. _`Memcached`: http://memcached.org
.. _`MongoDB`: http://www.mongodb.org
.. _`SQLAlchemy`: http://www.sqlalchemy.org/
.. _`Django`: http://djangoproject.com

下例中你将会使用 `amqp` 结果后端来发送状态消息。后端通过 :class:`@Celery`
的 ``backend`` 参数来指定。如果你选择使用配置模块，则通过
:setting:`CELERY_RESULT_BACKEND` 选项来设置::

    app = Celery('tasks', backend='amqp', broker='amqp://')

或者如果你想要把 Redis 用作结果后端，但仍然用 RabbitMQ 作为消息中间人
（常见的搭配）::

    app = Celery('tasks', backend='redis://localhost', broker='amqp://')

更多关于结果后端的内容见 :ref:`task-result-backends` 。

配置好结果后端后，让我们再次调用任务。这次你会得到调用任务后返回的
:class:`~@AsyncResult` 实例::

    >>> result = add.delay(4, 4)

:meth:`~@AsyncResult.ready` 方法查看任务是否完成处理::

    >>> result.ready()
    False

你可以等待任务完成，但这很少使用，因为它把异步调用变成了同步调用::

    >>> result.get(timeout=1)
    8

倘若任务抛出了一个异常， :meth:`~@AsyncResult.get` 会重新抛出异常，
但你可以指定 ``propagate`` 参数来覆盖这一行为::

    >>> result.get(propagate=False)

如果任务抛出了一个异常，你也可以获取原始的回溯信息::

    >>> result.traceback
    …

完整的结果对象参考见 :mod:`celery.result` 。

.. _celerytut-configuration:

配置
=============

Celery，如同家用电器一般，并不需要太多的操作。它有一个输入和一个输出，
你必须把输入连接到中间人上，如果想则把输出连接到结果后端上。但如果你
仔细观察后盖，有一个盖子露出许多滑块、转盘和按钮：这就是配置。

默认配置对大多数使用案例已经足够好了，但有许多事情需要微调来让 Celery
如你所愿地工作。阅读可用选项是熟悉可以配置什么的明智之举。你可以在
:ref:`configuration` 参考中查阅这些选项。

配置可以直接在应用上设置，也可以使用一个独立的配置模块。

例如你可以通过修改 :setting:`CELERY_TASK_SERIALIZER` 选项来配置序列化任
务载荷的默认的序列化方式：

.. code-block:: python

    app.conf.CELERY_TASK_SERIALIZER = 'json'

如果你一次性设置多个选项，你可以使用 ``update`` ：

.. code-block:: python

    app.conf.update(
        CELERY_TASK_SERIALIZER='json',
        CELERY_ACCEPT_CONTENT=['json'],  # Ignore other content
        CELERY_RESULT_SERIALIZER='json',
        CELERY_TIMEZONE='Europe/Oslo',
        CELERY_ENABLE_UTC=True,
    )

对于大型项目，采用独立配置模块更为有效，事实上你会为硬编码周期任务间隔和
任务路由选项感到沮丧，因为中心化保存配置更合适。尤其是对于库而言，这使得
用户控制任务行为成为可能，你也可以想象系统管理员在遇到系统故障时对配置做
出简单修改。

你可以调用 :meth:`~@Celery.config_from_object` 来让 Celery 实例
加载配置模块：

.. code-block:: python

    app.config_from_object('celeryconfig')

配置模块通常称为 ``celeryconfig`` ，你也可以使用任意的模块名。

名为 ``celeryconfig.py`` 的模块必须可以从当前目录或 Python 路径加载，它可
以是这样：

:file:`celeryconfig.py`:

.. code-block:: python

    BROKER_URL = 'amqp://'
    CELERY_RESULT_BACKEND = 'amqp://'

    CELERY_TASK_SERIALIZER = 'json'
    CELERY_RESULT_SERIALIZER = 'json'
    CELERY_ACCEPT_CONTENT=['json']
    CELERY_TIMEZONE = 'Europe/Oslo'
    CELERY_ENABLE_UTC = True

要验证你的配置文件可以正确工作，且不包含语法错误，你可以尝试导入它：

.. code-block:: bash

    $ python -m celeryconfig

配置选项的完整参考见 :ref:`configuration` 。

要证明配置文件的强大，比如这个例子展示了如何把“脏活”路由到专
用的队列：

:file:`celeryconfig.py`:

.. code-block:: python

    CELERY_ROUTES = {
        'tasks.add': 'low-priority',
    }

或者，你可以限制任务的速率，这样每分钟只允许处理 10 个该类型的任务：

:file:`celeryconfig.py`:

.. code-block:: python

    CELERY_ANNOTATIONS = {
        'tasks.add': {'rate_limit': '10/m'}
    }

如果你使用 RabbitMQ 或 Redis 作为中间人，那么你也可以在运行时直接在
职程上设置速率限制：

.. code-block:: bash

    $ celery control rate_limit tasks.add 10/m
    worker@example.com: OK
        new rate limit set successfully

任务路由的详情见 :ref:`guide-routing` 。关于注解的更多见
:setting:`CELERY_ANNOTATIONS` 选项。关于远程控制命令和如何监视职程行为
的更多见 :ref:`guide-monitoring` 。

何去何从
=====================

如果你想要进一步了解，你应该继续阅读 :ref:`进阶 <next-steps>` 教程，
之后你可以学习 :ref:`用户指南 <guide>` 。

.. _celerytut-troubleshooting:

故障处理
===============

:ref:`faq` 章节也有一份故障处理提示。

职程无法启动：权限错误
---------------------------------------

- 如果你使用 Debian、Ubuntu 或其他 Debian 系的发行版：

    Debian 最近把 ``/dev/shm/`` 特殊文件重命名为 ``/run/shm`` 。

    简单的处置方式就是创建一个符号链接：

    .. code-block:: bash

        # ln -s /run/shm /dev/shm

- 其他：

    如果你提供了 :option:`--pidfile` 、 :option:`--logfile` 或
    ``--statedb`` 参数中的任意一个，那么你必须确保它们指向了启动
    职程的那个用户可写可读的文件/目录。

结果后端没有奏效或任务总处于 ``PENDING`` （待处理）状态
----------------------------------------------------------------------

所有任务默认都是 ``PENDING`` 的，所以状态会更好地命名为“未知”。
Celery 在任务发出时不更新任何状态，并且任何没有历史状态的任务被
假定为待处理（毕竟你能获知任务 ID）。

1) 确保任务没有启用 ``ignore_result`` 。

    启用这个选项会强制所有职程跳过状态更行。

2) 确保没有启用 :setting:`CELERY_IGNORE_RESULT` 选项。

3) 确保你没有仍在运行旧职程。

    偶然启动多个职程序是很容易的，所以确保之前的职程在你启动新的职程时
    已经恰当地关闭了。

    为配置所期望的结果后端的旧职程可能会一直运行并劫持任务。

    `--pidfile` 参数可以设置为一个绝对路径来避免该状况。

4) 确保客户端配置了正确的结果后端。

    如果因为某些原因，客户端被配置使用了与职程不同的结果后端，那么你讲
    收不到结果，所以请确保检视后端是否正确：

    .. code-block:: python

        >>> result = task.delay(…)
        >>> print(result.backend)