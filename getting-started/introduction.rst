.. _intro:

========================
Celery 简介
========================

.. contents::
    :local:
    :depth: 1

何为任务队列？
=====================

任务队列是一种在线程或机器间分发任务的机制。

消息队列的输入是工作的一个单元，称为任务，独立的职程（Worker）进程持续
监视队列中是否有需要处理的新任务。

Celery 用消息通信，通常使用中间人（Broker）在客户端和职程间斡旋。这个过程
从客户端向队列添加消息开始，之后中间人把消息派送给职程。

Celery 系统可包含多个职程和中间人，以此获得高可用性和横向扩展能力。

Celery 是用 Python 编写的，但协议可以用任何语言实现。迄今，已有 Ruby 实现
的 RCelery_ 、node.js 实现的 node-celery_ 以及一个 `PHP 客户端`_ ，语言
互通也可以通过 :ref:`using webhooks <guide-webhooks>` 实现。


.. _RCelery: http://leapfrogdevelopment.github.com/rcelery/
.. _`PHP 客户端`: https://github.com/gjedeer/celery-php
.. _node-celery: https://github.com/mher/node-celery

我需要什么？
===============

.. sidebar:: 版本需求
    :subtitle: Celery 的 3.0 版本可运行在

    - Python ❨2.5, 2.6, 2.7, 3.2, 3.3❩
    - PyPy ❨1.8, 1.9❩
    - Jython ❨2.5, 2.7❩.

    这是最后一个支持 Python 2.5 的版本，也即从下个版本需要
    Python 2.6 或更新版本的 Python。最后一个支持 Python 2.4
    的版本为 Celery 2.2 系列。

*Celery* 需要一个发送和接受消息的传输者。RabbitMQ 和 Redis 中间人
的消息传输支持所有特性，但也提供大量其他实验性方案的支持，包括
用 SQLite 进行本地开发。

*Celery* 可以单机运行，也可以在多台机器上运行，甚至可以跨越数据中心运行。

上手
===========

如果这是你第一次尝试 Celery，或你从以前版本刚步入 Celery 3.0，那么你应该
阅读一下我们的入门教程：

- :ref:`first-steps`
- :ref:`next-steps`

Celery 是…
==============

.. _`mailing-list`: http://groups.google.com/group/celery-users

.. topic:: \ 

    - **简单**

        Celery 易于使用和维护，并且它 *不需要配置文件* 。

        Celery 有一个活跃、友好的社区来让你寻求帮助，包括一个
        `邮件列表 <mailing-list>`_ 和一个 :ref:`IRC 频道 <irc-channel>` 。


        下面是一个你可以实现的最简应用：

        .. code-block:: python

            from celery import Celery

            app = Celery('hello', broker='amqp://guest@localhost//')

            @app.task
            def hello():
                return 'hello world'

    - **高可用性**

        倘若连接丢失或失败，职程和客户端会自动重试，并且一些中间人
        通过 *主/主* 或 *主/从* 方式复制来提高可用性。

    - **快速**

        单个 Celery 进程每分钟可处理数以百万计的任务，而保持往返延迟
        在亚毫秒级（使用 RabbitMQ、py-librabbitmq 和优化过的设置）。

    - **灵活**

        *Celery* 几乎所有部分都可以扩展或单独使用。可以自制连接池、
        序列化、压缩模式、日志、调度器、消费者、生产者、自动扩展、
        中间人传输或更多。


.. topic:: 它支持

    .. hlist::
        :columns: 2

        - **中间人**

            - :ref:`RabbitMQ <broker-rabbitmq>`, :ref:`Redis <broker-redis>`,
            - :ref:`MongoDB <broker-mongodb>` （实验性）, ZeroMQ （实验性）
            - :ref:`CouchDB <broker-couchdb>` （实验性）, :ref:`SQLAlchemy <broker-sqlalchemy>` （实验性）
            - :ref:`Django ORM <broker-django>` （实验性）, :ref:`Amazon SQS <broker-sqs>`, （实验性）
            - 还有更多…

        - **并发**

            - prefork（多进程）,
            - Eventlet_, gevent_
            - 多线程/单线程

        - **结果存储**

            - AMQP, Redis
            - memcached, MongoDB
            - SQLAlchemy, Django ORM
            - Apache Cassandra

        - **序列化**

            - *pickle*, *json*, *yaml*, *msgpack*
            - *zlib*, *bzip2* 压缩
            - 密码学消息签名

特性
========

.. topic:: \ 

    .. hlist::
        :columns: 2

        - **监视**

            整条流水线的监视时间由职程发出，并用于内建或外部的工具
            告知你集群的工作状况——而且是实时的。

            :ref:`深入了解… <guide-monitoring>`.

        - **工作流**

            一系列功能强大的称为“Canvas”的原语（Primitive）用于构建
            或简单、或复杂的工作流。包括分组、连锁、分割等等。

            :ref:`深入了解… <guide-canvas>`.

        - **时间和速率限制**

            你可以控制每秒/分钟/小时执行的任务数，或任务的最长运行时间，
            并且可以为特定职程或不同类型的任务设置为默认值。

            :ref:`深入了解… <worker-time-limits>`.

        - **计划任务**

            你可以指定任务在若干秒后或在 :class:`~datetime.datetime`
            运行，或你可以基于单纯的时间间隔或支持分钟、小时、每周的第
            几天、每月的第几天以及每年的第几月的 crontab 表达式来使用
            周期任务来重现事件。

            :ref:`深入了解… <guide-beat>`.

        - **自动重载入**

            在开发中，职程可以配置为在源码修改时自动重载入，包含对 Linux
            上的 :manpage:`inotify(7)` 支持。

            :ref:`深入了解… <worker-autoreloading>`.

        - **自动扩展**

            根据负载自动重调职程池的大小或用户指定的测量值，用于限制
            共享主机/云环境的内存使用，或是保证给定的服务质量。

            :ref:`深入了解…  <worker-autoscaling>`.

        - **资源泄露保护**

            :option:`--maxtasksperchild` 选项用于控制用户任务泄露的诸如
            内存或文件描述符这些易超出掌控的资源。

            :ref:`深入了解… <worker-maxtasksperchild>`.

        - **用户组件**

            每个职程组件都可以自定义，并且额外组件可以由用户定义。职程是用
            “bootsteps” 构建的——一个允许细粒度控制职程内构件的依赖图。

.. _`Eventlet`: http://eventlet.net/
.. _`gevent`: http://gevent.org/

框架集成
=====================

Celery 易于与 Web 框架集成，其中的一些甚至已经有了集成包：

    +--------------------+------------------------+
    | `Django`_          | `django-celery`_       |
    +--------------------+------------------------+
    | `Pyramid`_         | `pyramid_celery`_      |
    +--------------------+------------------------+
    | `Pylons`_          | `celery-pylons`_       |
    +--------------------+------------------------+
    | `Flask`_           | 不需要                 |
    +--------------------+------------------------+
    | `web2py`_          | `web2py-celery`_       |
    +--------------------+------------------------+
    | `Tornado`_         | `tornado-celery`_      |
    +--------------------+------------------------+

集成包并非是严格必要的，但它们让开发更简便，并且有时它们在
:manpage:`fork(2)` 上添加了比如关闭数据库连接这样的重要回调。

.. _`Django`: http://djangoproject.com/
.. _`Pylons`: http://pylonshq.com/
.. _`Flask`: http://flask.pocoo.org/
.. _`web2py`: http://web2py.com/
.. _`Bottle`: http://bottlepy.org/
.. _`Pyramid`: http://docs.pylonsproject.org/en/latest/docs/pyramid.html
.. _`pyramid_celery`: http://pypi.python.org/pypi/pyramid_celery/
.. _`django-celery`: http://pypi.python.org/pypi/django-celery
.. _`celery-pylons`: http://pypi.python.org/pypi/celery-pylons
.. _`web2py-celery`: http://code.google.com/p/web2py-celery/
.. _`Tornado`: http://www.tornadoweb.org/
.. _`tornado-celery`: http://github.com/mher/tornado-celery/

快速跳转
=========

.. topic:: 我想要阅读⟶

    .. hlist::
        :columns: 2

        - :ref:`获取任务的返回值 <task-states>`
        - :ref:`在任务中使用日志 <task-logging>`
        - :ref:`学习最佳实践 <task-best-practices>`
        - :ref:`创建自定义的任务基类 <task-custom-classes>`
        - :ref:`向一组任务添加回调 <canvas-chord>`
        - :ref:`分割任务 <canvas-chunks>`
        - :ref:`优化职程 <guide-optimizing>`
        - :ref:`内建的任务状态列表 <task-builtin-states>`
        - :ref:`创建自定义任务状态 <custom-states>`
        - :ref:`设置自定义任务名称 <task-names>`
        - :ref:`跟踪开始任务 <task-track-started>`
        - :ref:`重试失败任务 <task-retry>`
        - :ref:`获取当前任务 ID <task-request-info>`
        - :ref:`获知任务分派到的队列 <task-request-info>`
        - :ref:`查看运行中职程列表 <monitoring-control>`
        - :ref:`清空消息 <monitoring-control>`
        - :ref:`检视职程的工作状况 <monitoring-control>`
        - :ref:`查看职程上注册的任务 <monitoring-control>`
        - :ref:`迁移任务到新中间人 <monitoring-control>`
        - :ref:`事件消息类型列表 <event-reference>`
        - :ref:`为 Celery 做贡献 <contributing>`
        - :ref:`配置设定的可用字段 <configuration>`
        - :ref:`任务失败的邮件通知 <conf-error-mails>`
        - :ref:`Celery 使用案例 <res-using-celery>`
        - :ref:`自制远程控制命令 <worker-custom-control-commands>`
        - :ref:`运行时修改职程的队列 <worker-queues>`

.. topic:: 跳转至 ⟶

    .. hlist::
        :columns: 4

        - :ref:`中间人 <brokers>`
        - :ref:`应用 <guide-app>`
        - :ref:`任务 <guide-tasks>`
        - :ref:`调用 <guide-calling>`
        - :ref:`职程 <guide-workers>`
        - :ref:`后台运行 <daemonizing>`
        - :ref:`监视 <guide-monitoring>`
        - :ref:`优化 <guide-optimizing>`
        - :ref:`安全 <guide-security>`
        - :ref:`路由 <guide-routing>`
        - :ref:`配置 <configuration>`
        - :ref:`Django <django>`
        - :ref:`贡献 <contributing>`
        - :ref:`信号 <signals>`
        - :ref:`FAQ <faq>`
        - :ref:`API 参考 <apiref>`

.. include:: ../includes/installation.txt