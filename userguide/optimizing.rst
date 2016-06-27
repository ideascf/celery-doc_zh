.. _guide-optimizing:

==============
 Optimizing（优化）
==============

Introduction
============
默认的配置做了很多妥协。这不是对于任意一个独立的案例都是最优的，但是适合大多数情况。

这里有一些针对特殊案例的优化。

优化可以影响运行环境的不同属性（性能指标），如：任务的执行时间、内存使用、高性能是的响应速度。
Optimizations can apply to different properties of the running environment,
be it the time tasks take to execute, the amount of memory used, or
responsiveness at times of high load.

Ensuring Operations
===================

在 `Programming Pearls`_ 书中，jon bentley 通过问下面的问题，提出了 `back-of-the-envelope calculations` 理论:

    ❝ How much water flows out of the Mississippi River in a day? ❞

这个练习 [*]_ 的目的是展示： 一个系统在一个时间片里能处理多少数据是有限制的。
`Back of the envelope caculations` 可以作为一种提升时间效率的手段。
The point of this exercise [*]_ is to show that there is a limit
to how much data a system can process in a timely manner.
Back of the envelope calculations can be used as a means to plan for this
ahead of time.

在Celery里，如果一个任务需要10分钟来完成，并且每分钟有10个新任务产生，这个队列将永远也不会为空。
这就是为什么监视队列长度，是非常重要的的原因。

其中一种方案是使用:ref:`using Munin <monitoring-munin>`。
你应该建立告警机制 —— 当队列长度到达一个不可接受的长度时，立即通知你。
这样，你可以立即采取某种操作，比如：增加新的 `worker` 节点 或 移除不必要的任务。

.. [*] The chapter is available to read for free here:
       `The back of the envelope`_.  The book is a classic text. Highly
       recommended.

.. _`Programming Pearls`: http://www.cs.bell-labs.com/cm/cs/pearls/

.. _`The back of the envelope`:
    http://books.google.com/books?id=kse_7qbWbjsC&pg=PA67

.. _optimizing-general-settings:

General Settings
================

.. _optimizing-librabbitmq:

librabbitmq
-----------

如果你正在使用RabbitMQ（AMQP）作为`broker`，你可以安装使用采用C编写的优化的 :mod:`librabbitmq` 模块:

.. code-block:: bash

    $ pip install librabbitmq

如果 :mod:`librabbitmq` 已经安装，`AMQP transport` 将自动的使用该模块，
或者，你可以使用``pyamqp://`` 或 ``librabbitmq://``前缀，来指定直接指定一个你想使用的.

.. _optimizing-connection-pools:

Broker Connection Pools
-----------------------

从版本2.5版本开始，`broker`的连接池默认被启用。

你可以调整配置项 :setting:`BROKER_POOL_LIMIT`，去最小化竞争(to minimize contention)，
并且这个值的设置，应该基于使用中间人连接的激活的线程数 或 greenthread数。

.. _optimizing-transient-queues:

Using Transient Queues
----------------------

默认情况下，Celery创建的`queue`是持久化的。 这意味着：`broker`将会把`message`写入磁盘以确保即使`broker`被重启，
这个`task`（消息对应的任务）也能被执行。

但是某些情况下丢失`message`是无所谓的，所以不是所有的`task`都需要持久化。
你可以为这些任务创建一个`transient` `queue`以提升性能:

.. code-block:: python

    from kombu import Exchange, Queue

    CELERY_QUEUES = (
        Queue('celery', routing_key='celery'),
        Queue('transient', routing_key='transient',
              delivery_mode=1),
    )


``delivery_mode`` 决定发送到这个队列消息会被如何交付。
1 意味着这个消息不会被写入磁盘，2（默认）意味这个消息会被写入磁盘。

指引一个`task`到新的临时`queue`，你可以指定这个``queue``参数（或使用配置项 :setting:`CELERY_ROUTES`）

.. code-block:: python

    task.apply_async(args, queue='transient')

了解更多详情，阅读 :ref:`routing guide <guide-routing>`

.. _optimizing-worker-settings:

Worker Settings
===============

.. _optimizing-prefetch-limit:

Prefetch Limits
---------------

*Prefetch* 是继承于`AMQP`的术语，但常常被用户误解。

预取限制是一个`worker`它自己可以存储的`task`（message）数量的限制。
如果被设置为0，`worker`将保持消费消息，不考虑可能有其它可用的职程节点可能已经（sooner, note1）处理了这些消息，或者说 这个消息可能根本不会装入内存。
If it is zero, the worker will keep
consuming messages, not respecting that there may be other
available worker nodes that may be able to process them sooner [*]_,
or that the messages may not even fit in memory.

`worker`的默认预取数量定义在配置项 :setting:`CELERYD_PREFETCH_MULTIPLIER` —— 几倍于并发数的倍数（进程/线程/协程）。

如果你有大量的长持续时间的`task`，你可能需要设置倍数为1. 这意味着在每个`worker`进程同一时间将仅仅reserve一个`task`。
If you have many tasks with a long duration you want
the multiplier value to be 1, which means it will only reserve one
task per worker process at a time.

然而，如果你有大量的短执行时间的`task`，并且吞吐率和往返延时对你是非常重要的，
那么这个倍速应该设置的较大。 如果消息已经被预取并且存在于内存中，那么`worker`每秒能够处理更多的`task`。
你不得不在实践中去寻找最适合你的倍数。在这种场景下，倍数设置为50或150可能是更有效的，亦或64/128。

如果你既有长执行时间的任务，也有短执行时间的任务，最佳的方案是使用两个被独立配置的`worker`节点，
并且根据执行时间来路由任务。参见 :ref:`guide-routing`。

.. [*] RabbitMQ and other brokers deliver messages round-robin,
       so this doesn't apply to an active system.  If there is no prefetch
       limit and you restart the cluster, there will be timing delays between
       nodes starting. If there are 3 offline nodes and one active node,
       all messages will be delivered to the active node.

.. [*] 并发相关的设置是：  配置项  :setting:`CELERYD_CONCURRENCY` 或 启动参数 :option:`-c`。


Reserve one task at a time
--------------------------

当采用`early acknowledgement`（默认的）时，预取倍数1,意味着这个`worker`的每个进程将至多reserve
一个额外的任务。

当用户询问是否可以禁用"任务预取"时，通常他们真正想要的是让`worker`，仅仅reserve子进程
(译者注：不一定时进程也有可能时线程和协程)数量的任务数。

但是，不启用`late acknowledgement`，实现没有执行完成的`task`重试，这是不可能实现的。
（译者注：early acknowledgement会在收到任务后，立即发送ack，即又去取新的任务。也就是：一个正在执行的task（已经ack了的），和一个预取的task（还没有ack的））。
对于一个已经开始的`task`，如果这个`worker`在执行过程中crash，那么这个任务将被重试，所以这个任务必须是`idempotent`_幂等的。
（查阅 :ref:`faq-acks_late-vs-retry`）

.. _`idempotent`: http://en.wikipedia.org/wiki/Idempotent

你可以通过使用如下配置选项来启用这个行为:

.. code-block:: python

    CELERY_ACKS_LATE = True
    CELERYD_PREFETCH_MULTIPLIER = 1

.. _prefork-pool-prefetch:

Prefork pool prefetch settings
------------------------------

`prefork pool` 将异步的发送尽可能多的`task`到processes，这意味着，
事实上，进程在预取`task`（译者注： 因为buffer的存在，任务被发送到了进程的pipe  buffer中）

这对性能是有好处的，但是它也意味着任务可能因为等待其它耗时任务完成，而被卡住::

    -> send T1 to Process A
    # A executes T1
    -> send T2 to Process B
    # B executes T2
    <- T2 complete

    -> send T3 to Process A
    # A still executing T1, T3 stuck in local buffer and
    # will not start until T1 returns

`worker`将会发送`task`到这些进程 —— 只要它的`pipe buffer`可写。`pipe buffer`的容量基于操作系统：
有些可能只有64Kb，但是在最新的linux版本中，这个buffer的容量是1MB（仅仅能被系统级的修改）。

你可以通过启用celery worker的:option:`-0fair`选项，来禁用这个预取行为

.. code-block:: bash

    $ celery -A proj worker -l info -Ofair

使用这个选项的`worker`，将仅仅写到那些可以立即执行`task`的进程，即禁用预取行为。
