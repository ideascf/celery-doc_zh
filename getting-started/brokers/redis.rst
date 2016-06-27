.. _broker-redis:

=============
使用 Redis
=============

.. _broker-redis-installation:

安装
============

对 Redis 的支持需要额外的依赖。你可以用 ``celery[redis]``
:ref:`捆绑 <bundles>` 同时安装 Celery 和这些依赖：


.. code-block:: bash

    $ pip install -U celery[redis]

.. _broker-redis-configuration:

配置
=============

配置非常简单，只需要设置 Redis 数据库的位置::

    BROKER_URL = 'redis://localhost:6379/0'

URL 的格式为::

    redis://:password@hostname:port/db_number

URL Scheme 后的所有字段都是可选的，并且默认为 localhost 的 6479 端口，使用
数据库 0。

.. _redis-visibility_timeout:

可见性超时
------------------

可见性超时时间定义了等待职程在消息分派到其他职程之前确认收到任务的秒数。一
定要阅读下面的 :ref:`redis-caveats` 一节。


这个选项通过 :setting:`BROKER_TRANSPORT_OPTIONS` 设置::

    BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 3600}  # 1 hour.

Redis 的默认可见性超时时间是 1 小时。

.. _redis-results-configuration:

结果
-------

如果你也想在 Redis 中存储任务的状态和返回值，你应该配置这些选项::

    CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

Redis 结果后端支持的选项列表见 :ref:`conf-redis-result-backend` 。

.. _redis-caveats:

警示
=======

- 广播信息默认队所有虚拟主机可见。

    你需要设置一个传输选项来给消息加上前缀，这样消息只会被活动的
    虚拟主机收到::

        BROKER_TRANSPORT_OPTIONS = {'fanout_prefix': True}

    注意，你将不能与运行老版本的职程或没有启用这个选项的职程通信。

    这个选项在以后将会使默认的，迁移宜早不宜迟。

- 如果任务没有在 :ref:`redis-visibility_timeout` 内确认接收，任务
  会被重新委派给另一个职程并执行。

    这会在预计到达时间/倒计时/重试这些执行时间超出可见性超时时间
    的任务上导致问题；事实上如果超时，任务将循环重新执行。

    所以你需要增大可见性超时时间，以符合你计划使用的最长预计到达
    时间。

    注意 Celery 会在职程关闭的时候重新分派消息，所以较长的可见性
    超时时间只会造成在断电或强制终止职程之后“丢失”任务重新委派的
    延迟。

    周期任务不会受可见性超时影响，因为这是一个与预计到达时间/倒
    计时不同的概念。

    你可以配置同名的传输选项来增大这个时间::

        BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 43200}

    这个值必须是整数，单位是秒。

- 监视事件（用于 flower 或其他工具）是全局的，并且不会受虚拟主机
  设置的影响。

    这是 Redis 带来的限制。Redis PUB/SUB 信道是全局的，并且不受
    数据库序号影响。

- Redis 在某些情况会从数据库中驱除键。

    如果你遇到了类似这样的错误::

        InconsistencyError, Probably the key ('_kombu.binding.celery') has been
        removed from the Redis database.

    你可以配置 Redis 服务器的 ``timeout`` 参数为 0 来避免键被驱
    逐。