.. _guide-beat:

========================
 Periodic Tasks（周期性任务）
========================

.. contents::
    :local:

Introduction
============

:program:`celery beat` 是一个调度器。
它会周期性的启动一个 `task` ——被集群中可用的 `worker` 节点执行。

默认下，条目(entry)是从配置选项 :setting:`CELERYBEAT_SCHEDULE` 取出，
当然,可以自定义存储位置也是可以的，比如储存这些条目到SQL数据库中。

你必须确保在同一时刻，只有一个调度器在一个调度表（schedule）上运行，否者你可能会遇到*重复*的task。
使用集中的方案，意味着调度表（schedule）不必是同步的，并且这个服务模块可以无锁操作。

.. _beat-timezones:

Time Zones
==========

默认情况，周期的任务调度使用UTC时区，但是你可以使用配置选项  :setting:`CELERY_TIMEZONE` 来改变时区。

更改时区为 `Europe/London` 的例子:

.. code-block:: python

    CELERY_TIMEZONE = 'Europe/London'


这个设置必须添加到你的app（译者注：Celery实例），要么直接使用``app.conf.CELERY_TIMEZONE='Europe/London'``来配置，
要么添加这个配置到你的配置模块中（如果你使用``app.config_from_object``）。
详情参见 :ref:`celeryt-configuration`,了解更多配置选项相关的信息。


默认的调度器（存储调度表在 :file:`celerybeat-schedule` 文件中）将会自动探测时区的改变，
如果发生改变，会重置这个调度表；但是其它的调度器可能没有这么聪明（比如Django.database.scheduler），
所以在这种情况下你需要要手动的重置这个调度表。

.. admonition:: Django Users

    Celery推荐使用并且和Django1.4引入的新配置``USE_TZ``相兼容。
    Celery recommends and is compatible with the new ``USE_TZ`` setting introduced
    in Django 1.4.

    对于Django用户来说，将使用 ``TIME_ZONE`` 作为Celery的时区，
    或者你可以用 :setting:`CELERY_TIMEZONE` 单独的为Celery指定时区。

    当时区相关的配置发生改变时，数据库调度器不会自动重置，所以你必须手动的这样做:

    .. code-block:: bash

        $ python manage.py shell
        >>> from djcelery.models import PeriodicTask
        >>> PeriodicTask.objects.update(last_run_at=None)

.. _beat-entries:

Entries
=======

为了周期性的调度 `task`，你必须添加entry到配置项 :setting:`CELERYBEAT_SCHEDULE` 中。

例如： 每30秒运行一次 `tasks.add`.

.. code-block:: python

    from datetime import timedelta

    CELERYBEAT_SCHEDULE = {
        'add-every-30-seconds': {
            'task': 'tasks.add',
            'schedule': timedelta(seconds=30),
            'args': (16, 16)
        },
    }

    CELERY_TIMEZONE = 'UTC'


.. note::

    如果你对这些设置应该放到哪里有疑问，请阅读 :ref:`celerytut-configuration`。
    要么你直接在app中设置这些选项，要么你将这些配置放置在一个独立模块中。

    如果你的位置参数置包含一个变量，那么请使用(arg,) 作为`args`.

为一个entry使用一个 :class:`~datetime.timedelta`，意味着这个task每30秒都会被发送一次
(第一个`task`将会在`celery beat`启动后30秒发送），然后之后的`task`每30秒发送一次。

一个类`crontab`的调度表同样可用，详见章节： `Crontab schedules`_。

类似``cron``，当第一个task没有在下一个task开始之前没有完成，那么`task`可能会发生重叠(译者注：多个task同时在执行)。
如果这种情况对你来说是一个问题的话，你应该使用锁机制去确保同一时间只会有一个实例可以执行这个`task`。
（详见样例： :ref:`cookbook-task-serial`）。

.. _beat-entry-fields:

Available Fields
----------------

* `task`

    被执行的`task`的名字

* `schedule`

    执行频率。

    可以是以秒为单位的整数、:class:`~datetime.timedelta` 、 :class:`~celery.schedules.crontab`。
    你也可以通过继承 :class:`~celery.schedules.schedule`的接口，来定义你自己的schedule类型。

* `args`

    位置参数 (:class:`list` or :class:`tuple`)

* `kwargs`

    关键字参数 (:class:`dict`)

* `options`

    执行设置选项 (:class:`dict`)。
    Execution options (:class:`dict`).

    可以是任意被 :meth:`~celery.task.base.Task.apply_async` 方法支持的参数，
    比如 `exchange`, `routing_key`, `expires` 等等

* `relative`

    默认情况，:class:`~datetime.timedelta` 调度条目，将按"by the clock"的方式被调度。
    这意味着执行周期将四舍五入到最近的秒、分、时、天，具体取决与执行周期参数 `timedelta`
    By default :class:`~datetime.timedelta` schedules are scheduled
    "by the clock". This means the frequency is rounded to the nearest
    second, minute, hour or day depending on the period of the timedelta.

    如果`relative`参数被设置为true，那么执行频率讲不会被四舍五入，并且是到 :program:`celery beat`
    启动时刻的相对时间。
    If `relative` is true the frequency is not rounded and will be
    relative to the time when :program:`celery beat` was started.

.. _beat-crontab:

Crontab schedules
=================

如果你想要更全面的控制task的执行，例如，一天中的特定时间或一周的特定一天，
你可以使用 :class:`~celery.schedules.crontab` 调度类型:

.. code-block:: python

    from celery.schedules import crontab

    CELERYBEAT_SCHEDULE = {
        # Executes every Monday morning at 7:30 A.M
        'add-every-monday-morning': {
            'task': 'tasks.add',
            'schedule': crontab(hour=7, minute=30, day_of_week=1),
            'args': (16, 16),
        },
    }

crontab表达式的语法是非常灵活。 例如:

+-----------------------------------------+--------------------------------------------+
| **Example**                             | **Meaning**                                |
+-----------------------------------------+--------------------------------------------+
| ``crontab()``                           | Execute every minute.                      |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0, hour=0)``           | Execute daily at midnight.                 |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0, hour='*/3')``       | Execute every three hours:                 |
|                                         | midnight, 3am, 6am, 9am,                   |
|                                         | noon, 3pm, 6pm, 9pm.                       |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0,``                   | Same as previous.                          |
|         ``hour='0,3,6,9,12,15,18,21')`` |                                            |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute='*/15')``              | Execute every 15 minutes.                  |
+-----------------------------------------+--------------------------------------------+
| ``crontab(day_of_week='sunday')``       | Execute every minute (!) at Sundays.       |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute='*',``                 | Same as previous.                          |
|         ``hour='*',``                   |                                            |
|         ``day_of_week='sun')``          |                                            |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute='*/10',``              | Execute every ten minutes, but only        |
|         ``hour='3,17,22',``             | between 3-4 am, 5-6 pm and 10-11 pm on     |
|         ``day_of_week='thu,fri')``      | Thursdays or Fridays.                      |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0, hour='*/2,*/3')``   | Execute every even hour, and every hour    |
|                                         | divisible by three. This means:            |
|                                         | at every hour *except*: 1am,               |
|                                         | 5am, 7am, 11am, 1pm, 5pm, 7pm,             |
|                                         | 11pm                                       |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0, hour='*/5')``       | Execute hour divisible by 5. This means    |
|                                         | that it is triggered at 3pm, not 5pm       |
|                                         | (since 3pm equals the 24-hour clock        |
|                                         | value of "15", which is divisible by 5).   |
+-----------------------------------------+--------------------------------------------+
| ``crontab(minute=0, hour='*/3,8-17')``  | Execute every hour divisible by 3, and     |
|                                         | every hour during office hours (8am-5pm).  |
+-----------------------------------------+--------------------------------------------+
| ``crontab(0, 0, day_of_month='2')``     | Execute on the second day of every month.  |
|                                         |                                            |
+-----------------------------------------+--------------------------------------------+
| ``crontab(0, 0,``                       | Execute on every even numbered day.        |
|         ``day_of_month='2-30/3')``      |                                            |
+-----------------------------------------+--------------------------------------------+
| ``crontab(0, 0,``                       | Execute on the first and third weeks of    |
|         ``day_of_month='1-7,15-21')``   | the month.                                 |
+-----------------------------------------+--------------------------------------------+
| ``crontab(0, 0, day_of_month='11',``    | Execute on 11th of May every year.         |
|          ``month_of_year='5')``         |                                            |
+-----------------------------------------+--------------------------------------------+
| ``crontab(0, 0,``                       | Execute on the first month of every        |
|         ``month_of_year='*/3')``        | quarter.                                   |
+-----------------------------------------+--------------------------------------------+

参见 :class:`celery.schedules.crontab` 了解更多详情。

.. _beat-starting:

Starting the Scheduler
======================

启动 :program:`celery beat` 服务:

.. code-block:: bash

    $ celery -A proj beat

你也可以使用`worker`的 `-B` 选项，来使用`worker`内置的`beat`。
如果你永远不会运行一个以上的worker节点，那么这是非常方便的。
但是这不是通常的做法，并且不推荐在生产线中使用：

.. code-block:: bash

    $ celery -A proj worker -B


Beat需要存储这些`task`的上次运行/启动时间，在本地数据库文件中(默认命名为: `celerybeat-schedule`)。
所以`beat`需要有当前目录的写权限，或者你可以为这个文件指定一个路径:

.. code-block:: bash

    $ celery -A proj beat -s /home/celery/var/run/celerybeat-schedule


.. note::

    后台化`beat` 参见 :ref:`daemonizing`.

.. _beat-custom-schedulers:

Using custom scheduler classes
------------------------------

可以在命令行参数中指定自定义的scheduler类（使用 `-S` 参数）。
默认的scheduler是 :class:`celery.beat.PersistentScheduler` ，
这个调度器简单地在本地数据库文件（:mod:`shelve`）中保存上次的启动时间。

`django-celery` 也封装了一个调度器 —— 在Django数据库中存储调度表：

.. code-block:: bash

    $ celery -A proj beat -S djcelery.schedulers.DatabaseScheduler

采用`django-celery`的调度器，你可以在`Django Admin`中增加、修改、移除这些周期性`task`。
