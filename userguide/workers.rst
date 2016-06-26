.. _guide-workers:

===============
 Workers Guide
===============

.. contents::
    :local:
    :depth: 1

.. _worker-starting:

Starting the worker
===================

.. sidebar:: Daemonizing

    你可能想使用一些后台化工具在后台启动`worker`。
    参见:ref:`daemonizing`,使用常用的后台化工具帮助detach `worker`。

你可以执行这个命令，在前台启动`worker`:

.. code-block:: bash

    $ celery -A proj worker -l info

要想查看`worker`的所有可用的命令行参数，请参照 :mod:`~celery.bin.worker`，或:

.. code-block:: bash

    $ celery worker --help

你也可以在同一台机器上启动多个`worker`实例。如果你要这么做，请确保为每一个独立的`worker`
分配一个唯一的名称（通过 :option:`--hostname|-n` 参数选项指定host name）

.. code-block:: bash

    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker1.%h
    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker2.%h
    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker3.%h

hostname参数可以使用以下变量扩展:

    - ``%h``:  包含domain的Hostname
    - ``%n``:  仅Hostname.
    - ``%d``:  仅Hostname.

例如，如果当前的Hostname是``george.example.com``，那么将被扩展为:

    - ``worker1.%h`` -> ``worker1.george.example.com``
    - ``worker1.%n`` -> ``worker1.george``
    - ``worker1.%d`` -> ``worker1.example.com``

.. admonition:: Note for :program:`supervisord` users.

   The ``%`` sign must be escaped by adding a second one: `%%h`.

.. _worker-stopping:

Stopping the worker
===================

应该使用:sig:`TERM`信号来关闭`worker`。

当关闭操作开始后，`worker`将在真正关闭之前，继续完成当前执行中的`tasks`；
所以如果这些`tasks`是非常重要的，你应该在执行一些毁灭性操作(比如发送:sig:`KILL`信号)之前等待它们执行完成。

如果在预计的时间之后，`worker`仍然没有关闭（例如：因为`task`由于陷入无限循环而被卡主），
你可以使用:sig:`KILL`信号来强制性结束这个`worker`；但是*注意*当前执行中的`task`将会被丢失
（除非这个 `task` 使用了 :attr:`~@Task.acks_late`选项）。

由于进程不能捕获处理:sig:`KILL`信号，`worker`将不能收割（reap）它的子进程，所以请确保手动完成。
你可以使用如下命令(trick)完成:

.. code-block:: bash

    $ ps auxww | grep 'celery worker' | awk '{print $2}' | xargs kill -9

.. _worker-restarting:

Restarting the worker
=====================

你可以先发送`TERM`信号去停止`worker`，然后在手动的启动一个新的`worker`实例。
开发过程中最便利的`worker`管理方式是：使用`celery multi`:

    .. code-block:: bash

        $ celery multi start 1 -A proj -l info -c4 --pidfile=/var/run/celery/%n.pid
        $ celery multi restart 1 --pidfile=/var/run/celery/%n.pid

对于生产环境部署时，你应该使用一个init脚本或者其它管理系统（参见 :ref:`daemonizing`）。

相比停止然后再重启一个`worker`，你也可以使用:sig:`HUP`信号来重启`worker`，
*但是请注意*： 这样`worker`在重启过程中仍然处于可响应状态，因此可能会导致一些问题，
并且不推荐你在生产环境中使用：

.. code-block:: bash

    $ kill -HUP $pid

.. note::

    使用:sig:`HUP`信号重启`worker`，仅当职程以守护进程方式运行与后台时可用
    （它没有控制终端）

    由于平台的一些限制，:sig:`HUP` 在OS X平台是被禁用了的。


.. _worker-process-signals:

Process Signals
===============

`worker`主进程处理了一下信号:

+--------------+-------------------------------------------------+
| :sig:`TERM`  | Warm shutdown, wait for tasks to complete.      |
+--------------+-------------------------------------------------+
| :sig:`QUIT`  | Cold shutdown, terminate ASAP                   |
+--------------+-------------------------------------------------+
| :sig:`USR1`  | Dump traceback for all active threads.          |
+--------------+-------------------------------------------------+
| :sig:`USR2`  | Remote debug, see :mod:`celery.contrib.rdb`.    |
+--------------+-------------------------------------------------+

.. _worker-files:

Variables in file paths
=======================

 :option:`--logfile`, :option:`--pidfile` 以及 :option:`--statedb`，
 这些命令行文件路径参数 可以包含一些可扩展变量:

Node name replacements
----------------------

- ``%h``:  包含Domain的Hostname.
- ``%n``:  仅Hostname.
- ``%d``:  仅Domain.
- ``%i``:  Prefork pool process index 或 0（主进程）.
- ``%I``:  包含分隔符的Prefork pool process index.

比如，如果当前的hostname时``george.example.com``，那么将有如下展开:

- ``--logfile=%h.log`` -> :file:`george.example.com.log`
- ``--logfile=%n.log`` -> :file:`george.log`
- ``--logfile=%d`` -> :file:`example.com.log`

.. _worker-files-process-index:

Prefork pool process index
--------------------------

The prefork pool process index specifiers will expand into a different
filename depending on the process that will eventually need to open the file.

这可以用来为每个子进程指定一个日志文件。

Note that the numbers will stay within the process limit even if processes
exit or if autoscale/maxtasksperchild/time limits are used.  I.e. the number
is the *process index* not the process count or pid.

* ``%i`` - Pool process index or 0 if MainProcess.

    Where ``-n worker1@example.com -c2 -f %n-%i.log`` will result in
    three log files:

        - :file:`worker1-0.log` (main process)
        - :file:`worker1-1.log` (pool process 1)
        - :file:`worker1-2.log` (pool process 2)

* ``%I`` - Pool process index with separator.

    Where ``-n worker1@example.com -c2 -f %n%I.log`` will result in
    three log files:

        - :file:`worker1.log` (main process)
        - :file:`worker1-1.log` (pool process 1)
        - :file:`worker1-2.log` (pool process 2)

.. _worker-concurrency:

Concurrency
===========

默认使用多进程来并发执行`task`，但是你也可以使用 :ref:`Eventlet <concurrency-eventlet>`。
`worker`的进程数/线程数，可以使用 :option:`--concurrency` 来指定，默认是当前主机可用CPU个数。

.. admonition:: Number of processes (multiprocessing/prefork pool)

    通常情况，进程池中进程数越多越好，但是存在一个零界点 —— 继续增加进程数将会对性能带来消极影响。
    事实上有些证据表明，运行多个`worker`实例会比运行一个职程更好。
    例如，运行3个`worker`，每个 `worker` 拥有包含10个进程的进程池。
    可以自行实验去找到最合适你自己的数值，因为这些数值取决于应用、工作负载、task运行次数、以及其它因数。

.. _worker-remote-control:

Remote control
==============

.. versionadded:: 2.0

.. sidebar:: The ``celery`` command

    :program:`celery` 被用于在命令行执行远程控制命令。 它支持以下列出的所有命令。
    参见 :ref:`monitoring-control` 了解更多详情。

pool support: *prefork, eventlet, gevent*,
blocking:*threads/solo* (see note)，
broker support: *amqp, redis*

`worker`拥有使用高优先级的广播消息队列来远程控制的能力。
命令被发送给所有`worker` 或者 指定的`worker`列表。

命令也可以有答复。客户端可以等待并收集这些答复。
由于没有集权中心去获得集群中的`worker`总数，也没有方法估计有多少`worker`可能会发送回复，
所以客户端有一个可配置的等待超时时间 —— 等待回复到达的deadline。默认超时时间是1秒钟。
如果一个`worker`没有在deadline之前回复，这并不意味着这个`worker`没有回复或者挂掉了，
而可能仅仅是因为网络延时或这个职程正在处理耗时的命令； 所以请适当地调整这个超时时间。

除了这个超时时间以外，客户端还可以指定一个等待的最大回复数。如果指定了操作目标，
那么这个限制会被设置为操作目标的数量。

.. note::

    虽然solo和threads pool也支持远程控制命令，但是任一task的执行将会阻塞掉其它等待中的控制命令。
    所以，在这种情况，你应该适当延长客户端等待回复的超时时间。

.. _worker-broadcast-fun:

The :meth:`~@control.broadcast` function.
----------------------------------------------------

这是一个客户端函数，用来发送命令到`worker`。一些拥有高级接口远程控制命令，
底层也是使用:meth:`~@control.broadcast`，比如： :meth:`~@control.rate_limit` and :meth:`~@control.ping`.

发送 :control:`rate_limit` 命令以及的关键字参数::

    >>> app.control.broadcast('rate_limit',
    ...                          arguments={'task_name': 'myapp.mytask',
    ...                                     'rate_limit': '200/m'})

这将异步地发送这个命令，不会等待回复。如需要回复，你必须设置`reply`参数::

    >>> app.control.broadcast('rate_limit', {
    ...     'task_name': 'myapp.mytask', 'rate_limit': '200/m'}, reply=True)
    [{'worker1.example.com': 'New rate limit set successfully'},
     {'worker2.example.com': 'New rate limit set successfully'},
     {'worker3.example.com': 'New rate limit set successfully'}]

使用`destination`参数,你可以指定一个接受这个命令的`worker`列表::

    >>> app.control.broadcast('rate_limit', {
    ...     'task_name': 'myapp.mytask',
    ...     'rate_limit': '200/m'}, reply=True,
    ...                             destination=['worker1@example.com'])
    [{'worker1.example.com': 'New rate limit set successfully'}]


当然，使用高级接口去设置速率限制是更方便的，但是有一些命令只能通过:meth:`~@control.broadcast`来完成。

Commands
========

.. control:: revoke

``revoke``: Revoking tasks
--------------------------
:pool support: all
:broker support: *amqp, redis*
:command: :program:`celery -A proj control revoke <task_id>`

所有`worker`节点都会在内存中或持续化在磁盘上，保持被撤销的`task`的ID。
(参见 see :ref:`worker-persistent-revokes`)

当一个`worker`收到一个撤销请求时，它将跳过执行这个`task`，
但是不会终止一个已经正在执行的`task`，除非你设置了`terminate`选项。

.. note::

    terminate选项是当一个`task`被卡主时，管理员的最后手段。
    它不是用来中断这个`task`的，而是用来中断正在执行这个`task`的*进程*，
    并且这个进程在此同时，可能已经正在处理另一个`task`。所以，因为这些原因，
    永远不要在程序中这样调用。

如果`terminate`选项被设置，处理这个`task`的`worker`子进程将被中断。
默认的中断信号是`TERM`，但是你可以通过`signal`参数来指定。
信号可以是任何在python标准库的:mod:`signal`模块中定义的信号的*大写*名称。

中断一个task，也会撤销它。
Terminating a task also revokes it.

**Example**

::

    >>> result.revoke()

    >>> AsyncResult(id).revoke()

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed')

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
    ...                    terminate=True)

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
    ...                    terminate=True, signal='SIGKILL')




Revoking multiple tasks
-----------------------

.. versionadded:: 3.1


revoke方法同时也接受一个list参数，将同时撤销多个`task`。

**Example**

::

    >>> app.control.revoke([
    ...    '7993b0aa-1f0b-4780-9af0-c47c0858b3f2',
    ...    'f565793e-b041-4b2b-9ca4-dca22762a55d',
    ...    'd9d35e03-2997-42d0-a13e-64a66b88a618',
    ])


从3.1版本开始引入的``GroupResult.revoke``方法就利用的这个特性.

.. _worker-persistent-revokes:

Persistent revokes
------------------

撤销task，通过发送一个广播消息给所有的`worker`，这些`worker`将在内存中保持被一份撤销的`task`列表。
当一个`worker`启动时，它会和其他集群中的其它`worker`同步这些被撤销的`task`。

由于被撤销的`task`列表存放于内存中，一旦所有`worker`发生重启，这个列表将会丢失。
如果你想在重启后仍然存有这个列表，你必须为 :program:`celery worker` 使用`-statedb`参数指定一个文件
—— 用来存储这些信息的文件:

.. code-block:: bash

    celery -A proj worker -l info --statedb=/var/run/celery/worker.state

或者，如果你使用 :program:`celery multi`，你想为每个`worker`实例创建一个单独的文件，
那么你可以使用`%n`变量，来扩展为当前节点的名称:

.. code-block:: bash

    celery multi start 2 -l info --statedb=/var/run/celery/%n.state


See also :ref:`worker-files`

注意： 要想revoke可以工作，必须确保远程控制命令可以工作。
目前，远程控制明了仅仅被  RabbitMQ (amqp) 和 Redis 支持。

.. _worker-time-limits:

Time Limits
===========

.. versionadded:: 2.0

pool support: *prefork/gevent*

.. sidebar:: Soft, or hard?

    时间限制相关的设置有两个值： `soft` 和 `hard`。
    `soft`时间限制，允许这个task(译者注：执行超时的task)，在它被杀掉之前，捕获一个异常来做清理工作。
    `hard`时间限制超时，是不可捕获的并且会强制中断这个task。
    The time limit is set in two values, `soft` and `hard`.
    The soft time limit allows the task to catch an exception
    to clean up before it is killed: the hard timeout is not catchable
    and force terminates the task.

一个task(A single task)可能会一直运行下去，如果有大量的`task`等待一些永远不会发生的事件，
将导致无限期地阻塞这个`worker`处理新的`task`。防止这种情况发生的最好方法是：启用*时间限制*。

时间限制（`--time-limit`）是`task`最大的的执行时间（秒为单位），达到最大时间后，
执行这个`task`的进程将中断自己并创建一个新的进程。
你也可以启用软时间限制（`--soft-time-limit`），这让你在进程中断自己之前，
可以捕获这个异常并处理一些清理工作。
You can also enable a soft time limit (`--soft-time-limit`),
this raises an exception the task can catch to clean up before the hard
time limit kills it:

.. code-block:: python

    from myapp import app
    from celery.exceptions import SoftTimeLimitExceeded

    @app.task
    def mytask():
        try:
            do_work()
        except SoftTimeLimitExceeded:
            clean_up_in_a_hurry()

时间限制同样可以使用配置项 :setting:`CELERYD_TASK_TIME_LIMIT` 或 :setting:`CELERYD_TASK_SOFT_TIME_LIMIT`
 来设置。

.. note::

    在windows平台和其它不支持``SIGUSR1``信号的平台下，时间限制特性当前不被支持。


Changing time limits at runtime
-------------------------------
.. versionadded:: 2.3

broker support: *amqp, redis*


有一个可以可以让你改变指定`task`的`soft`和`hard`时间限制的命令 —— 名为``time_limit``。

比如，改变``tasks.crawl_the_web`` task的`soft`时间限制为1分钟、`hard`时间限制为2分钟::

    >>> app.control.time_limit('tasks.crawl_the_web',
                               soft=60, hard=120, reply=True)
    [{'worker1.example.com': {'ok': 'time limits set successfully'}}]

仅对时间限制设置*更改后*启动的`task`有效。

.. _worker-rate-limits:

Rate Limits
===========

.. control:: rate_limit

Changing rate-limits at runtime
-------------------------------

比如，改变`myapp.mytask`的速率限制为: 每分钟最多执行200个:
Example changing the rate limit for the `myapp.mytask` task to execute
at most 200 tasks of that type every minute:

.. code-block:: python

    >>> app.control.rate_limit('myapp.mytask', '200/m')

上面的操作中没有指定目标(worker)，所以这个集群中的所有`worker`实例都将会受到影响。
如果你仅仅想针对一些`worker`生效，你可以在``destination``参数中指定这些目标:

.. code-block:: python

    >>> app.control.rate_limit('myapp.mytask', '200/m',
    ...            destination=['celery@worker1.example.com'])

.. warning::

    这对开启了 :setting:`CELERY_DISABLE_RATE_LIMITS` 选项的`worker`无效。

.. _worker-maxtasksperchild:

Max tasks per child setting
===========================

.. versionadded:: 2.0

pool support: *prefork*

With this option you can configure the maximum number of tasks
a worker can execute before it's replaced by a new process.

This is useful if you have memory leaks you have no control over
for example from closed source C extensions.

The option can be set using the workers `--maxtasksperchild` argument
or using the :setting:`CELERYD_MAX_TASKS_PER_CHILD` setting.

.. _worker-autoscaling:

Autoscaling
===========

.. versionadded:: 2.2

pool support: *prefork*, *gevent*

*autoscaler*组件被用来基于负载动态的调整pool的容量:

- 当负载较高时，`autoscaler`自动增加更多的process来处理工作
- 当负载降低时自动移除多余的processs.

通过 :option:`--autoscale` 选项来启用，这个选项需要带入两个数字：
最大 和 最小的pool processes数量::

        --autoscale=AUTOSCALE
             Enable autoscaling by providing
             max_concurrency,min_concurrency.  Example:
               --autoscale=10,3 (always keep 3 processes, but grow to
              10 if necessary).

你可以通过继承 :class:`~celery.worker.autoscaler.Autoscaler`，来定义你自己的缩放规则。
一些常用的度量标准包括：平均负载、可用内存。
可以使用配置项 :setting:`CELERYD_AUTOSCALER` 来指定你自定义的`autoscaler`。

.. _worker-queues:

Queues
======

一个`worker`实例可以从多个`queue`中消耗`task`。 默认情况下，
它将从所有定义在配置项 :setting:`CELERY_QUEUES` 的`queue`中消耗`task`
（如果没有指定，默认的`queue`名称为``celery``）。

你可以在启动的时候指定这个`worker`要从哪些`queue`中消费`task`，
在 :option:`-Q` 选项中，使用 `,` 来指定一个 `queue`列表:

.. code-block:: bash

    $ celery -A proj worker -l info -Q foo,bar,baz

如果这个`queue` 在配置项 :setting:`CELERY_QUEUES` 中被定义，就使用这个队列配置(译者注： 配置项中该队列的配置)。
否则，celery将自动为你生成一个新的队列（这取决于配置项 :setting:`CELERY_CREATE_MISSING_QUEUES`）

你可以通过远程控制命令 :control:`add_consumer` 和 :control:`cancel_consumer`，
在运行时告知`worker` 开始或停止从一个队列中消耗task。

.. control:: add_consumer

Queues: Adding consumers
------------------------

控制命令 :control:`add_consumer` 将通知一个或多个`worker`，开始从一个`queue`中消耗`task`。
*这个操作是幂等的。*

你可以使用 :program:`celery control` 程序，通知集群中的所有`worker`开始从名为``foo``的`queue`中消耗`task`:

.. code-block:: bash

    $ celery -A proj control add_consumer foo
    -> worker1.local: OK
        started consuming from u'foo'

如果你想指定要操作的特定`worker`，你可以使用 :option:`--destination` 参数:

.. code-block:: bash

    $ celery -A proj control add_consumer foo -d worker1.local

同样地，你可以使用:meth:`app.control.add_consumer()`方法动态的修改::

    >>> app.control.add_consumer('foo', reply=True)
    [{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]

    >>> app.control.add_consumer('foo', reply=True,
    ...                          destination=['worker1@example.com'])
    [{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]


至今为止，我只展示了动态的调整`queue`，如果你想控制更多，你可以指定`exchange`，`routing_key`和其他选项::

    >>> app.control.add_consumer(
    ...     queue='baz',
    ...     exchange='ex',
    ...     exchange_type='topic',
    ...     routing_key='media.*',
    ...     options={
    ...         'queue_durable': False,
    ...         'exchange_durable': False,
    ...     },
    ...     reply=True,
    ...     destination=['w1@example.com', 'w2@example.com'])


.. control:: cancel_consumer

Queues: Cancelling consumers
----------------------------

你可以使用 :control:`cancel_consumer` 控制命令取消`worker`对一个`queue`的消费。

强制让集群中的所有`worker`取消从指定`queue`中消费，你可以使用 :program:`celery control` 程序

.. code-block:: bash

    $ celery -A proj control cancel_consumer foo

你可以使用  :option:`--destination` 参数，来指定一个特定的`worker`或一个`worker`的列表:

.. code-block:: bash

    $ celery -A proj control cancel_consumer foo -d worker1.local

同样地，你可以使用 :meth:`@control.cancel_consumer` 方法来取消对一个`queue`的消费:

.. code-block:: bash

    >>> app.control.cancel_consumer('foo', reply=True)
    [{u'worker1.local': {u'ok': u"no longer consuming from u'foo'"}}]

.. control:: active_queues

Queues: List of active queues
-----------------------------

你可以通过使用 :control:`active_queues` 控制命令，获得`worker`当前消耗的`queue`列表:

.. code-block:: bash

    $ celery -A proj inspect active_queues
    [...]

类似于其他远程控制命令，这个命令也支持 :option:`--destination` 参数去指定要操作的`worker`:

.. code-block:: bash

    $ celery -A proj inspect active_queues -d worker1.local
    [...]


同样地，你可以在程序中使用:meth:`app.control.inspect.active_queues()`方法::

    >>> app.control.inspect().active_queues()
    [...]

    >>> app.control.inspect(['worker1.local']).active_queues()
    [...]

.. _worker-autoreloading:

Autoreloading
=============

.. versionadded:: 2.5

pool support: *prefork, eventlet, gevent, threads, solo*

使用 :option:`--autoreload` 选项启动 :program:`celery worker` ，
将使`worker`监测所有被被导入的`task`模块的文件改动
（也包括其他非task模块，但被加入到配置项 :setting:`CELERY_IMPORTS` 中的；
或者使用命令行参数 :option:`-I|--include` 包括的）。

这是实验性质的特性，应仅仅用于调试开发；因为在python中reloading模块行为是未定义的，
并且可能导致难以诊断的bug和崩溃，所以在生成环境中不推荐使用`autoreload`。
Celery使用类似于Django ``runserver``命令同样的方法来使用`autoreload`。

当`auto-reload`特性被启用时，`worker`将启动一个而外的线程 —— 用于监测文件的改动的线程。
只要改动被监测到，新模块将被导入，已经导入的模块将被重新导入；
如果`prefork pool`被使用，那么子进程将完成正在执行的工作并退出；
以至可以被全新已经重载代码的的processes替换。
if the prefork pool is used the child processes will finish the work
they are doing and exit, so that they can be replaced by fresh processes
effectively reloading the code.

文件系统通知后端是插件化的，目前有3中实现：

* inotify (Linux)

    如果安装了:mod:`pyinotify`，就使用这个。
    如果你的运行平台是Linux，那么这是推荐的实现方案。
    使用下面的命令安装:mod:`pyinotify`库:

    .. code-block:: bash

        $ pip install pyinotify

* kqueue (OS X/BSD)

* stat

    最差的实现方案 —— 简单的使用``stat``方法去轮询所有文件，代价是非常昂贵的。

你可以通过设置环境变量 :envvar:`CELERYD_FSNOTIFY` 来强制指定一个实现方案:

.. code-block:: bash

    $ env CELERYD_FSNOTIFY=stat celery worker -l info --autoreload

.. _worker-autoreload:

.. control:: pool_restart

Pool Restart Command
--------------------

.. versionadded:: 2.5

需要配置项 :setting:`CELERYD_POOL_RESTARTS` 被启用。

远程控制命令 :control:`pool_restart` 发送一个重启请求到`worker`所有的子进程。
这对强制`worker` 导入新模块或重载已经导入的模块是非常有用的。
这个命令*不会打断*执行中的task。

Example
~~~~~~~

运行如下命令，将引起 `foo` 和 `bar` 模块被`woker`进程导入:

.. code-block:: python

    >>> app.control.broadcast('pool_restart',
    ...                       arguments={'modules': ['foo', 'bar']})

设置``reaload``参数为True，去重载一个已经被导入的模：

.. code-block:: python

    >>> app.control.broadcast('pool_restart',
    ...                       arguments={'modules': ['foo'],
    ...                                  'reload': True})

如果你不指定任何模块，那么所有已知的task模块将被导入/重新导入：

.. code-block:: python

    >>> app.control.broadcast('pool_restart', arguments={'reload': True})

``modules``参数是一个要modify的模块的列表。``reload``参数决定是否重载已经被导入的模块。
默认情况下``reload``是被禁用的。`pool_restart`命令使用python的:func:`reload()`函数去重载模块，
或者你可以通过传入``reloader``参数，提供你自定义的reloader。

.. note::

    模块重载伴随一些注意事项 —— 记录在 :func:`reload` 文档中。
    请阅读文档并确保你的模块是适合被重载的。

.. seealso::

    - http://pyunit.sourceforge.net/notes/reloading.html
    - http://www.indelible.org/ink/python-reloading/
    - http://docs.python.org/library/functions.html#reload


.. _worker-inspect:

Inspecting workers
==================

:class:`@control.inspect` 使你可以`inspect` 运行中的 `worker`。
它使用远程控制命令的高级选项.

你也可以使用``celery``命令去inspect worker，并且它支持和:class:`@control`同样的命令接口。

.. code-block:: python

    # Inspect all nodes.
    >>> i = app.control.inspect()

    # Specify multiple nodes to inspect.
    >>> i = app.control.inspect(['worker1.example.com',
                                'worker2.example.com'])

    # Specify a single node to inspect.
    >>> i = app.control.inspect('worker1.example.com')

.. _worker-inspect-registered-tasks:

Dump of registered tasks
------------------------

你可以使用:meth:`~@control.inspect.registered` 得到在`worker`中注册的`task`的列表::

    >>> i.registered()
    [{'worker1.example.com': ['tasks.add',
                              'tasks.sleeptask']}]

.. _worker-inspect-active-tasks:

Dump of currently executing tasks
---------------------------------

你可以使用 :meth:`~@control.inspect.active` 获得当前正在active的`tasks`::

    >>> i.active()
    [{'worker1.example.com':
        [{'name': 'tasks.sleeptask',
          'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
          'args': '(8,)',
          'kwargs': '{}'}]}]

.. _worker-inspect-eta-schedule:

Dump of scheduled (ETA) tasks
-----------------------------

你可以使用 :meth:`~@control.inspect.scheduled` 获得当前等待被调度的`task`

    >>> i.scheduled()
    [{'worker1.example.com':
        [{'eta': '2010-06-07 09:07:52', 'priority': 0,
          'request': {
            'name': 'tasks.sleeptask',
            'id': '1a7980ea-8b19-413e-91d2-0b74f3844c4d',
            'args': '[1]',
            'kwargs': '{}'}},
         {'eta': '2010-06-07 09:07:53', 'priority': 0,
          'request': {
            'name': 'tasks.sleeptask',
            'id': '49661b9a-aa22-4120-94b7-9ee8031d219d',
            'args': '[2]',
            'kwargs': '{}'}}]}]

.. note::

    这些任务是拥有`eta/countdown`参数的任务，不是周期任务（periodic tasks）。

.. _worker-inspect-reserved:

Dump of reserved tasks
----------------------

已经保存的任务，是已经提取但是还没有被执行的任务。

你可以通过 :meth:`~@control.inspect.reserved` 来获得这些`task`的列表::

    >>> i.reserved()
    [{'worker1.example.com':
        [{'name': 'tasks.sleeptask',
          'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
          'args': '(8,)',
          'kwargs': '{}'}]}]


.. _worker-statistics:

Statistics
----------

远程控制命令 ``inspect stats`` (或 :meth:`~@control.inspect.stats`)
将返回给你一个有用（或许没那么有用）的关于`worker`的长列表统计信息:

.. code-block:: bash

    $ celery -A proj inspect stats

so useful) statistics about the worker:

- ``broker``

    `broker`相关的段落

    * ``connect_timeout``

        建立一个新连接的超时时间（秒为单位的int/float）。

    * ``heartbeat``

        当前心跳值（被客户端设置）
        Current heartbeat value (set by client).

    * ``hostname``

        Hostname of the remote broker.

    * ``insist``

        No longer used.

    * ``login_method``

        Login method used to connect to the broker.

    * ``port``

        Port of the remote broker.

    * ``ssl``

        SSL enabled/disabled.

    * ``transport``

        Name of transport used (e.g. ``amqp`` or ``redis``)

    * ``transport_options``

        Options passed to transport.

    * ``uri_prefix``

        Some transports expects the host name to be an URL, this applies to
        for example SQLAlchemy where the host name part is the connection URI:

            redis+socket:///tmp/redis.sock

        In this example the uri prefix will be ``redis``.

    * ``userid``

        User id used to connect to the broker with.

    * ``virtual_host``

        Virtual host used.

- ``clock``

    Value of the workers logical clock.  This is a positive integer and should
    be increasing every time you receive statistics.

- ``pid``

    Process id of the worker instance (Main process).

- ``pool``

    Pool-specific section.

    * ``max-concurrency``

        Max number of processes/threads/green threads.

    * ``max-tasks-per-child``

        Max number of tasks a thread may execute before being recycled.

    * ``processes``

        List of pids (or thread-id's).

    * ``put-guarded-by-semaphore``

        Internal

    * ``timeouts``

        Default values for time limits.

    * ``writes``

        Specific to the prefork pool, this shows the distribution of writes
        to each process in the pool when using async I/O.

- ``prefetch_count``

    Current prefetch count value for the task consumer.

- ``rusage``

    System usage statistics.  The fields available may be different
    on your platform.

    From :manpage:`getrusage(2)`:

    * ``stime``

        Time spent in operating system code on behalf of this process.

    * ``utime``

        Time spent executing user instructions.

    * ``maxrss``

        The maximum resident size used by this process (in kilobytes).

    * ``idrss``

        Amount of unshared memory used for data (in kilobytes times ticks of
        execution)

    * ``isrss``

        Amount of unshared memory used for stack space (in kilobytes times
        ticks of execution)

    * ``ixrss``

        Amount of memory shared with other processes (in kilobytes times
        ticks of execution).

    * ``inblock``

        Number of times the file system had to read from the disk on behalf of
        this process.

    * ``oublock``

        Number of times the file system has to write to disk on behalf of
        this process.

    * ``majflt``

        Number of page faults which were serviced by doing I/O.

    * ``minflt``

        Number of page faults which were serviced without doing I/O.

    * ``msgrcv``

        Number of IPC messages received.

    * ``msgsnd``

        Number of IPC messages sent.

    * ``nvcsw``

        Number of times this process voluntarily invoked a context switch.

    * ``nivcsw``

        Number of times an involuntary context switch took place.

    * ``nsignals``

        Number of signals received.

    * ``nswap``

        The number of times this process was swapped entirely out of memory.


- ``total``

    List of task names and a total number of times that task have been
    executed since worker start.


Additional Commands
===================

.. control:: shutdown

Remote shutdown
---------------

这个命令将优雅的关闭远程的`worker`:

.. code-block:: python

    >>> app.control.broadcast('shutdown') # shutdown all workers
    >>> app.control.broadcast('shutdown, destination="worker1@example.com")

.. control:: ping

Ping
----

这个命令向所有存活的`worker`发送一个ping请求。`worker`回复一个'pong'字符串，并且仅仅是这样。
默认将使用1秒作为超时时间，除非你指定一个超时时间:

.. code-block:: python

    >>> app.control.ping(timeout=0.5)
    [{'worker1.example.com': 'pong'},
     {'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

:meth:`~@control.ping` 也支持`destination`参数，所以你可以指定你要ping哪个`worker::

    >>> ping(['worker2.example.com', 'worker3.example.com'])
    [{'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

.. _worker-enable-events:

.. control:: enable_events
.. control:: disable_events

Enable/disable events
---------------------

你可以通过使用`enable_events`，`disable_events`命令, 来启用或禁用事件。
这是非常有用的 —— 使用 :program:`celery events`/:program:`celerymon` 临时监测一个`worker`。

.. code-block:: python

    >>> app.control.enable_events()
    >>> app.control.disable_events()

.. _worker-custom-control-commands:

Writing your own remote control commands
========================================

远程控制命令被注册到控制面板，并且带入一个单独的参数：
当前的:class:`~celery.worker.control.ControlDispatch`实例。
如有必要，你可以从那里访问active的 :class:`~celery.worker.consumer.Consumer`。

这是一个增加预取任务总数的远程控制命令的示例:

.. code-block:: python

    from celery.worker.control import Panel

    @Panel.register
    def increase_prefetch_count(state, n=1):
        state.consumer.qos.increment_eventually(n)
        return {'ok': 'prefetch count incremented'}
