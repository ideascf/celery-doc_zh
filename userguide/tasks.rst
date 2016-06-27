.. _guide-tasks:

=====
Tasks
=====

Tasks are the building blocks of Celery applications.

task是一个能被任意可调用对象创建的类。他扮演了双重角色，在它被定义的时候，以及当一个task被调用(发送消息) 时和 当一个worker收到这个消息时。

每个task类都有一个唯一的名称，这个名称被消息引用，以便于worker能够找到正确的函数去执行。

task消息不会消失，直到这个消息被一个worker 确认(  :term:`acknowledged`  )。worker可以预先存储大量的消息，即使这个worker被杀死——因为致命的错误；否则，这个消息将被重投递给其他的worker）。

理想上task函数应该是幂等(  :term:`idempotent`  )(译者注：如HTTP中的GET。幂等，即多次调用间无联系，且结果可预料并唯一)，意味着使用相同的参数多次调用这个函数，不会造成预料外的结果。因为worker不能决断你的task是否是幂等的，所以默认的行为时：提前确认这个消息，在开始执行以前。 以便于一个已经开始的task不会再次被执行。

如果你能确定你的函数是幂等的，你可以设置  :attr:`acks_late`  选项，让worker在这个task*返回之后*再确认这个消息。
See also the FAQ entry :ref:`faq-acks_late-vs-retry`.

--

在这个章节中，你将学习所有关于定义task相关的知识，
下面是**目录清单**：

.. contents::
    :local:
    :depth: 1


.. _task-basics:

Basics
======

你可以轻松的使用  :meth:`~@task`  装饰器，来从任何callable创建一个task：

.. code-block:: python

    from .models import User

    @app.task
    def create_user(username, password):
        User.objects.create(username=username, password=password)

针对task，有很多的选项(  :ref:`options <task-options>`  )可以设置，这些设置可以作为装饰器的参数来传入：

.. code-block:: python

    @app.task(serializer='json')
    def create_user(username, password):
        User.objects.create(username=username, password=password)



.. sidebar:: 我应该如何导入task装饰器，以及"app"是什么？

    task装饰器可以在你的:class:`@Celery` application实例上使用，
    如果你不知道application实例是什么，请参阅:ref:`first-steps`.

    如果你正在使用Django，或者仍然在使用“旧”的基于模块级别的celery API，
    那么你可以这样导入task装饰器::

        from celery import task

        @task
        def add(x, y):
            return x + y

.. sidebar:: 多个装饰器

    当与task装饰器结合使用多个装饰器时，你必须确保`task`装饰器是最后被应用的。
    (在python中意味着，它必须位于列表中的第一个):

    .. code-block:: python

        @app.task
        @decorator2
        @decorator1
        def add(x, y):
            return x + y

.. _task-names:

Names
=====

每一个task必须拥有一个唯一的名称，如果你没有指定名称，那么一个新的名称将会根据函数名生成。

比如:

.. code-block:: python

    >>> @app.task(name='sum-of-two-numbers')
    >>> def add(x, y):
    ...     return x + y

    >>> add.name
    'sum-of-two-numbers'

最佳实践是使用模块的名字作为一个名字空间，这样就可以不和其他模块定义的同名task发生冲突。

.. code-block:: python

    >>> @app.task(name='tasks.add')
    >>> def add(x, y):
    ...     return x + y

你可以通过task的name属性来获取这个task的名称：

    >>> add.name
    'tasks.add'

这正是生成的名称，如果模块名称是“tasks.py”：

:file:`tasks.py`:

.. code-block:: python

    @app.task
    def add(x, y):
        return x + y

    >>> from tasks import add
    >>> add.name
    'tasks.add'

.. _task-naming-relative-imports:

Automatic naming and relative imports
-------------------------------------

相对导入和自动命名不是一个好的组合方式，所以，如果你正在使用相对导入，你应该明确的设置task的名称。

例如，假设client以".tasks"的方式导入模块“myapp.tasks”，worker以"myapp.tasks"的方式导入该模块，那么生成的名字将不能匹配，最终导致worker抛出  :exc:`~@NotRegistered`  异常。

这同样也是使用Django和使用project.myapp风格命名在  ``INSTALLED_APPS``  中的问题：

.. code-block:: python

    INSTALLED_APPS = ['project.myapp']

如果你以名称  ``project.myapp``   安装这个app，那么tasks模块应该以  ``project.myapp.tasks``  的方式被导入，所以你必须保证你始终以这种方式导入tasks:

.. code-block:: python

    >>> from project.myapp.tasks import mytask   # << GOOD

    >>> from myapp.tasks import mytask    # << BAD!!!

第二个样例将示范由于worker和client使用了不同的方式(名称)来导入模块，task被不同的命名了：

.. code-block:: python

    >>> from project.myapp.tasks import mytask
    >>> mytask.name
    'project.myapp.tasks.mytask'

    >>> from myapp.tasks import mytask
    >>> mytask.name
    'myapp.tasks.mytask'

所以，因为这个原因，你最好保证你导入模块方式的一致性，这也是一项Python的最佳实践。

同样的，你不应该使用老式的相对导入：

.. code-block:: python

    from module import foo   # BAD!

    from proj.module import foo  # GOOD!

新式的相对导入时优良并且可用的：

.. code-block:: python

    from .module import foo  # GOOD!

如果你想在一个已经广泛使用老式风格的项目中使用Celery，并且你没有太多的时间去重构这些已经存在的代码，那么你可以考虑为task明确的指定一个名称，而不是依赖于自动命名。

.. code-block:: python

    @task(name='proj.tasks.add')
    def add(x, y):
        return x + y

.. _task-request-info:

Context
=======

请求(  :attr:`~@Task.request`  )中包含信息和正在执行的任务的状态。

request定义了如下属性：

:id:  执行任务的唯一ID.

:group: The unique id a group, if this task is a member.

:chord: The unique id of the chord this task belongs to (if the task
        is part of the header).

:args: 位置参数.

:kwargs: 关键字参数.

:retries: 当前task已经重试的次数。
          一个从  `0`  开始的整数.

:is_eager:  如果你想task在本地执行，即client，而不是在worker中执行。那么请设置为  :const:`True`

:eta: ask的原始预计到达时间（ETA）。使用UTC时间（依赖于  `CELERY_ENABLE_UTC`  配置选项）

:expires: 原始的task的过期时间。使用UTC时间(同eta，依赖于  :setting:`CELERY_ENABLE_UTC`  )。

:logfile: worker记录日志的文件.  See `Logging`_.

:loglevel: 当前使用的日志级别.

:hostname: 执行task的worker的hostname.

:delivery_info: 交付信息的附加消息。这是一个映射表，包含了用来交付这个task，使用的交换和路由key。被  :meth:`~@Task.retry`  方法使用，用来重发这个task到桐乡的目的队列中。这个映射表(字典)中可以使用的key依赖于使用的消息中间件

:called_directly: 如果这个task不是被worker执行，这个标志被设置为真。（译者注：和is_eager相关）.

:callbacks: subtasks的列表，如果这个task成功，这个列表中的subtask将被调用.

:errback: subtasks的列表，如果这个task失败，这个列表中的subtask将被调用.

:utc: 如果调用者启用了UTC，那么这个标志设置为True(  :setting:`CELERY_ENABLE_UTC`  ).


.. versionadded:: 3.1

:headers:   消息的headers（可能为`Nonoe`）.

:reply_to:   发送回复的目的地(队列名称).

:correlation_id: Usually the same as the task id, often used in amqp
                 to keep track of what a reply is for.


一个在访问上下文信息的样例task：

.. code-block:: python

    @app.task(bind=True)
    def dump_context(self, x, y):
        print('Executing task id {0.id}, args: {0.args!r} kwargs: {0.kwargs!r}'.format(
                self.request))

``bind`` 参数，意味着这个函数将成为一个"bound method"，所以你可以在一个task类型的实例中，访问它的属性和方法。


.. _task-logging:

logging
=======

worker将自动的为你建立logging，或者你可以手动的配置logging。

一个名为“celery.task”的特殊logger可以被使用，你可以继承这个logger，去自动获取task的名称、唯一task ID作为日志的一部分。

最佳实践是：在你模块的顶部，为你所有的tasks创建一个公共的logger：

.. code-block:: python

    from celery.utils.log import get_task_logger

    logger = get_task_logger(__name__)

    @app.task
    def add(x, y):
        logger.info('Adding {0} + {1}'.format(x, y))
        return x + y

Celery 使用Python的标准logger库，你可以在  :mod:`logging`  模块中找到它的文档。

你也可以使用  :func:`print`  函数，任何写到标准输出和标注错误输出，都会被重定向到logging系统(你可以禁用这个特性，详情参见  :setting:`CELERY_REDIRECT_STDOUTS`  )

.. note::

    The worker will not update the redirection if you create a logger instance
    somewhere in your task or task module.

    If you want to redirect ``sys.stdout`` and ``sys.stderr`` to a custom
    logger you have to enable this manually, for example:

    .. code-block:: python

        import sys

        logger = get_task_logger(__name__)

        @app.task(bind=True)
        def add(self, x, y):
            old_outs = sys.stdout, sys.stderr
            rlevel = self.app.conf.CELERY_REDIRECT_STDOUTS_LEVEL
            try:
                self.app.log.redirect_stdouts_to_logger(logger, rlevel)
                print('Adding {0} + {1}'.format(x, y))
                return x + y
            finally:
                sys.stdout, sys.stderr = old_outs


.. _task-retry:

Retrying
========

:meth:`~@Task.retry`   方法可以用来重新执行这个task，例如发生了可恢复的错误。

当你调用``retry``方法的时候，它将发送一个新的消息，使用相同的task ID，并且它将保证这个新消息，被发送到和原task一样的队列中去。

一个task被重试的同时，也会被作为一个task状态(   :ref:`task-states`  )被记录。所以你可以追踪这个task的进度，使用result实例。


这是一个使用``retry``的样例：

.. code-block:: python

    @app.task(bind=True)
    def send_twitter_status(self, oauth, tweet):
        try:
            twitter = Twitter(oauth)
            twitter.update_status(tweet)
        except (Twitter.FailWhaleError, Twitter.LoginError) as exc:
            raise self.retry(exc=exc)

.. note::

    :meth:`~@Task.retry` 调用将会抛出一个异常，所以任何位于retry之后的代码都将不可达。
    抛出的异常是 :exc:`~@Retry`，它不会被当做一个错误来处理，而是作为一个语义——告知worker
    这个task将被重试。以便在启用了backend的时候，它能够存储正确的状态。

    这是标准的操作流程，除非retry函数的``throw``参数被设置为 :const:`False`。

task装饰器的bind参数，给予了访问``self``(task实例)的能力。

``exc``方法被用来传递异常信息——在日志记录以及存储task结果时使用。
异常信息和traceback信息都将出现在task的状态中(如果启用的backend)

如果这个task拥有``max_retries``属性，那么当达到最大重试次数时
将会再次抛出当前的异常。除非以下几种情况：

- ``exc`` 参数没有传入

    在这种情况下 :exc:`~@MaxRetriesExceeded` 异常将会被抛出

- 当前无异常

    假如当前没有原始异常(original exception)去再次抛出，``exc``参数将会被用来替代之，
    所以：

    .. code-block:: python

        self.retry(exc=Twitter.LoginError())

    将会抛出``exc``参数传入的异常。

.. _task-retry-custom-delay:

Using a custom retry delay
--------------------------

当一个task触发了重试，它可以在执行重试操作前等待一段给定的时间，默认的延时时间被
:attr:`~@Task.default_retry_delay` 属性定义。 默认情况这个值被设置为3分钟。
注意这个延时时间配置的单位是秒（int 或 float）。

你可以提供`countdown`参数给 :meth:`~@Task.retry` 去覆盖默认的延时时间。

.. code-block:: python

    @app.task(bind=True, default_retry_delay=30 * 60)  # retry in 30 minutes.
    def add(self, x, y):
        try:
            …
        except Exception as exc:
            raise self.retry(exc=exc, countdown=60)  # override the default and
                                                     # retry in 1 minute

.. _task-options:

List of Options
===============

task装饰器可以传入若干的选项，来改变这个task的行为。
例如你可以使用 :attr:`rate_limit`选项来设置速率限制。

任何传递给task装饰器的关键字参数，实际上都将被设置为这个resulting task class的一个属性。
这里提供了一份内建属性的列表。

General
-------

.. _task-general-options:

.. attribute:: Task.name

    这个task的注册名称。

    你可以手动的设置这个名称，或者根据当前的module和class的名称自动生成。
    参见 :ref:`task-names`。

.. attribute:: Task.request

    如果这个task被执行，这个属性将包含和当前请求相关的一些信息。
    采用的Thread local storage。

    参见 :ref:`task-request-info`.

.. attribute:: Task.abstract

    抽象类不会被注册，而是用做新的task类型的基类。

.. attribute:: Task.max_retries

    放弃这个task之前的最大重试尝试次数。
    如果重试次数达到这个值，异常 :exc:`~@MaxRetriesExceeded` 将会被抛出。
    *注意:* 由于在异常触发时不会自动重试，所以你必须手动调用 :meth:`~@Task.retry`。

    默认值是3.
    设置为 :const:`None` 将会禁用重试次数限制，即这个task将会一直重试直到成功。

.. attribute:: Task.throws

    预料中的异常类组成的可选元组——不应该被视作一个实际错误。
    Optional tuple of expected error classes that should not be regarded
    as an actual error.

    这个list中的异常，将被视作失败报告给result backend，
    但是worker不会将其作为一个错误记录这个事件，并且没有任何的traceback信息被包含。

    例如:

    .. code-block:: python

        @task(throws=(KeyError, HttpNotFound)):
        def get_foo():
            something()

    Error types:

    - 预期中的错误 (in ``Task.throws``)
        以``INFO``级别记录，不包含traceback信息。

    - 非预期中的错误

        以``ERROR``级别记录，包含traceback信息。

.. attribute:: Task.trail

    默认情况下，task将会记录被调用的subtask(``task.request.children``)。
    并且，将会和最终结果一起被存储到result backend中，调用端可用的访问方式
    ``AsyncResult.children``。

    这个list可能伴随启动大量的subtasks而变动十分庞大，你可以设置这个属性为False来禁用之。

.. attribute:: Task.default_retry_delay

    以秒为单位的默认执行重试等待时间。可以为 :class:`int` 或 :class:`float`。
    默认为3分钟的延时。

.. attribute:: Task.rate_limit

    为这个task类型设置速率限制，将会限制指定帧(frame)时间内可执行该tasks的数量。
    当速率限制生效时，task仍然可以完成，但是可能会需要消耗一定的时间等待启动。

    :const:`None`意味着无速率限制。
    如果这是一个整数或浮点数，将会被解释为：“每秒任务数”。

    可以使用`"/s"`,`"/m"`,`"/h"`追加到数值后面，来指定为按秒、分、时进行速率限制。
    task将会在指定的时间帧内，被均匀地配发出去。

    例如： `"100/m"`(每分钟一百个task)。
    将强制同一woker实例上的两个task的启动时间间隔不小于600ms。

    默认是 :setting:`CELERY_DEFAULT_RATE_LIMIT` 配置，
    如果没有特别的指出速率限制，默认将会被禁用速率限制。

    注意这是*每个worker实例*的速率限制，不是一个全局的速率限制。
    开启一个全局的速率限制(比如：调用一个拥有每秒最大请求次数的API)，你必须限制到一个给定队列。
    you must restrict to a given queue.

.. attribute:: Task.time_limit


    这个task以秒为单位的hard执行限制时间（hard time limit）。
    如果没有设置，将使用worker的默认值。

.. attribute:: Task.soft_time_limit

    这个task以秒为单位的soft执行限制时间(soft time limit)。
    如果没有设置，将使用worker的默认值。

.. attribute:: Task.ignore_result

    不存储task的状态。
    注意：这意味着你不能使用 :class:`~celery.result.AsyncResult` 去检查这个task是否已经ready，
    或者是去获得这个task的返回值。

.. attribute:: Task.store_errors_even_if_ignored

    如果设置为 :const:`True`，即使这个task被配置为ignore result，也会存储errors信息。

.. attribute:: Task.send_error_emails

    发送email，每当这个类型的task执行失败的时候。
    默认为 :setting:`CELERY_SEND_TASK_ERROR_EMAILS`  配置。
    查阅 :ref:`conf-error-mails` 获取更多信息

.. attribute:: Task.ErrorMail

    如果这个task启用了send_error_emails，那么这是定义发送error mails逻辑的类。
    this is the class defining the logic to send error mails.

.. attribute:: Task.serializer

    一个字符串标记默认使用的序列化方法。

    默认为 :setting:`CELERY_TASK_SERIALIZER` 配置。
    可以是`picke`、`json`、`yaml`或者其他自定义的已经被注册到 :mod:`kombu.serialization.registry`的序列化方法。

    查阅 :ref:`calling-serializers` 获取更多信息.

.. attribute:: Task.compression

    一个字符串，标记默认使用的压缩方案。

    默认为 :setting:`CELERY_MESSAGE_COMPRESSION`。
    可以是`gzip`、`bzip2`、或者其他自定义的已经注册到 :mod:`kombu.compression` 注册表的压缩方案。

    查阅 :ref:`calling-compression` 获取更多信息。

.. attribute:: Task.backend

    这个任务的结果的store backend。
    默认为 :setting:`CELERY_RESULT_BACKEND` 配置。

.. attribute:: Task.acks_late

    如果设置为 :const:`True` ，这个task的消息将会在task*执行后*被确认，
    而*不是开始前*（默认的行为。译者注：参照幂等）

    注意,这意味着个任务可能被执行两次,如果worker在执行中途崩溃，
    可能这对于某些application来说是可以接受的。

    全局的默认设置可以通过 :setting:`CELERY_ACKS_LATE` 来重写。

.. _task-track-started:

.. attribute:: Task.track_started

    If :const:`True` the task will report its status as "started"
    when the task is executed by a worker.
    The default value is :const:`False` as the normal behaviour is to not
    report that level of granularity. Tasks are either pending, finished,
    or waiting to be retried.  Having a "started" status can be useful for
    when there are long running tasks and there is a need to report which
    task is currently running.

    The host name and process id of the worker executing the task
    will be available in the state metadata (e.g. `result.info['pid']`)

    The global default can be overridden by the
    :setting:`CELERY_TRACK_STARTED` setting.


.. seealso::

    The API reference for :class:`~@Task`.

.. _task-states:

States
======

Celery can keep track of the tasks current state.  The state also contains the
result of a successful task, or the exception and traceback information of a
failed task.

There are several *result backends* to choose from, and they all have
different strengths and weaknesses (see :ref:`task-result-backends`).

During its lifetime a task will transition through several possible states,
and each state may have arbitrary metadata attached to it.  When a task
moves into a new state the previous state is
forgotten about, but some transitions can be deducted, (e.g. a task now
in the :state:`FAILED` state, is implied to have been in the
:state:`STARTED` state at some point).

There are also sets of states, like the set of
:state:`FAILURE_STATES`, and the set of :state:`READY_STATES`.

The client uses the membership of these sets to decide whether
the exception should be re-raised (:state:`PROPAGATE_STATES`), or whether
the state can be cached (it can if the task is ready).

You can also define :ref:`custom-states`.

.. _task-result-backends:

Result Backends
---------------

If you want to keep track of tasks or need the return values, then Celery
must store or send the states somewhere so that they can be retrieved later.
There are several built-in result backends to choose from: SQLAlchemy/Django ORM,
Memcached, RabbitMQ (amqp), MongoDB, and Redis -- or you can define your own.

No backend works well for every use case.
You should read about the strengths and weaknesses of each backend, and choose
the most appropriate for your needs.


.. seealso::

    :ref:`conf-result-backend`

RabbitMQ Result Backend
~~~~~~~~~~~~~~~~~~~~~~~

The RabbitMQ result backend (amqp) is special as it does not actually *store*
the states, but rather sends them as messages.  This is an important difference as it
means that a result *can only be retrieved once*; If you have two processes
waiting for the same result, one of the processes will never receive the
result!

Even with that limitation, it is an excellent choice if you need to receive
state changes in real-time.  Using messaging means the client does not have to
poll for new states.

There are several other pitfalls you should be aware of when using the
RabbitMQ result backend:

* Every new task creates a new queue on the server, with thousands of tasks
  the broker may be overloaded with queues and this will affect performance in
  negative ways. If you're using RabbitMQ then each queue will be a separate
  Erlang process, so if you're planning to keep many results simultaneously you
  may have to increase the Erlang process limit, and the maximum number of file
  descriptors your OS allows.

* Old results will be cleaned automatically, based on the
  :setting:`CELERY_TASK_RESULT_EXPIRES` setting.  By default this is set to
  expire after 1 day: if you have a very busy cluster you should lower
  this value.

For a list of options supported by the RabbitMQ result backend, please see
:ref:`conf-amqp-result-backend`.


Database Result Backend
~~~~~~~~~~~~~~~~~~~~~~~

Keeping state in the database can be convenient for many, especially for
web applications with a database already in place, but it also comes with
limitations.

* Polling the database for new states is expensive, and so you should
  increase the polling intervals of operations such as `result.get()`.

* Some databases use a default transaction isolation level that
  is not suitable for polling tables for changes.

  In MySQL the default transaction isolation level is `REPEATABLE-READ`, which
  means the transaction will not see changes by other transactions until the
  transaction is committed.  It is recommended that you change to the
  `READ-COMMITTED` isolation level.


.. _task-builtin-states:

Built-in States
---------------

.. state:: PENDING

PENDING
~~~~~~~

Task is waiting for execution or unknown.
Any task id that is not known is implied to be in the pending state.

.. state:: STARTED

STARTED
~~~~~~~

Task has been started.
Not reported by default, to enable please see :attr:`@Task.track_started`.

:metadata: `pid` and `hostname` of the worker process executing
           the task.

.. state:: SUCCESS

SUCCESS
~~~~~~~

Task has been successfully executed.

:metadata: `result` contains the return value of the task.
:propagates: Yes
:ready: Yes

.. state:: FAILURE

FAILURE
~~~~~~~

Task execution resulted in failure.

:metadata: `result` contains the exception occurred, and `traceback`
           contains the backtrace of the stack at the point when the
           exception was raised.
:propagates: Yes

.. state:: RETRY

RETRY
~~~~~

Task is being retried.

:metadata: `result` contains the exception that caused the retry,
           and `traceback` contains the backtrace of the stack at the point
           when the exceptions was raised.
:propagates: No

.. state:: REVOKED

REVOKED
~~~~~~~

Task has been revoked.

:propagates: Yes

.. _custom-states:

Custom states
-------------

You can easily define your own states, all you need is a unique name.
The name of the state is usually an uppercase string.  As an example
you could have a look at :mod:`abortable tasks <~celery.contrib.abortable>`
which defines its own custom :state:`ABORTED` state.

Use :meth:`~@Task.update_state` to update a task's state::

    @app.task(bind=True)
    def upload_files(self, filenames):
        for i, file in enumerate(filenames):
            if not self.request.called_directly:
                self.update_state(state='PROGRESS',
                    meta={'current': i, 'total': len(filenames)})


Here I created the state `"PROGRESS"`, which tells any application
aware of this state that the task is currently in progress, and also where
it is in the process by having `current` and `total` counts as part of the
state metadata.  This can then be used to create e.g. progress bars.

.. _pickling_exceptions:

Creating pickleable exceptions
------------------------------

A rarely known Python fact is that exceptions must conform to some
simple rules to support being serialized by the pickle module.

Tasks that raise exceptions that are not pickleable will not work
properly when Pickle is used as the serializer.

To make sure that your exceptions are pickleable the exception
*MUST* provide the original arguments it was instantiated
with in its ``.args`` attribute.  The simplest way
to ensure this is to have the exception call ``Exception.__init__``.

Let's look at some examples that work, and one that doesn't:

.. code-block:: python


    # OK:
    class HttpError(Exception):
        pass

    # BAD:
    class HttpError(Exception):

        def __init__(self, status_code):
            self.status_code = status_code

    # OK:
    class HttpError(Exception):

        def __init__(self, status_code):
            self.status_code = status_code
            Exception.__init__(self, status_code)  # <-- REQUIRED


So the rule is:
For any exception that supports custom arguments ``*args``,
``Exception.__init__(self, *args)`` must be used.

There is no special support for *keyword arguments*, so if you
want to preserve keyword arguments when the exception is unpickled
you have to pass them as regular args:

.. code-block:: python

    class HttpError(Exception):

        def __init__(self, status_code, headers=None, body=None):
            self.status_code = status_code
            self.headers = headers
            self.body = body

            super(HttpError, self).__init__(status_code, headers, body)

.. _task-semipredicates:

Semipredicates
==============

The worker wraps the task in a tracing function which records the final
state of the task.  There are a number of exceptions that can be used to
signal this function to change how it treats the return of the task.

.. _task-semipred-ignore:

Ignore
------

The task may raise :exc:`~@Ignore` to force the worker to ignore the
task.  This means that no state will be recorded for the task, but the
message is still acknowledged (removed from queue).

This can be used if you want to implement custom revoke-like
functionality, or manually store the result of a task.

Example keeping revoked tasks in a Redis set:

.. code-block:: python

    from celery.exceptions import Ignore

    @app.task(bind=True)
    def some_task(self):
        if redis.ismember('tasks.revoked', self.request.id):
            raise Ignore()

Example that stores results manually:

.. code-block:: python

    from celery import states
    from celery.exceptions import Ignore

    @app.task(bind=True)
    def get_tweets(self, user):
        timeline = twitter.get_timeline(user)
        if not self.request.called_directly:
            self.update_state(state=states.SUCCESS, meta=timeline)
        raise Ignore()

.. _task-semipred-reject:

Reject
------

The task may raise :exc:`~@Reject` to reject the task message using
AMQPs ``basic_reject`` method.  This will not have any effect unless
:attr:`Task.acks_late` is enabled.

Rejecting a message has the same effect as acking it, but some
brokers may implement additional functionality that can be used.
For example RabbitMQ supports the concept of `Dead Letter Exchanges`_
where a queue can be configured to use a dead letter exchange that rejected
messages are redelivered to.

.. _`Dead Letter Exchanges`: http://www.rabbitmq.com/dlx.html

Reject can also be used to requeue messages, but please be very careful
when using this as it can easily result in an infinite message loop.

Example using reject when a task causes an out of memory condition:

.. code-block:: python

    import errno
    from celery.exceptions import Reject

    @app.task(bind=True, acks_late=True)
    def render_scene(self, path):
        file = get_file(path)
        try:
            renderer.render_scene(file)

        # if the file is too big to fit in memory
        # we reject it so that it's redelivered to the dead letter exchange
        # and we can manually inspect the situation.
        except MemoryError as exc:
            raise Reject(exc, requeue=False)
        except OSError as exc:
            if exc.errno == errno.ENOMEM:
                raise Reject(exc, requeue=False)

        # For any other error we retry after 10 seconds.
        except Exception as exc:
            raise self.retry(exc, countdown=10)

Example requeuing the message:

.. code-block:: python

    from celery.exceptions import Reject

    @app.task(bind=True, acks_late=True)
    def requeues(self):
        if not self.request.delivery_info['redelivered']:
            raise Reject('no reason', requeue=True)
        print('received two times')

Consult your broker documentation for more details about the ``basic_reject``
method.


.. _task-semipred-retry:

Retry
-----

The :exc:`~@Retry` exception is raised by the ``Task.retry`` method
to tell the worker that the task is being retried.

.. _task-custom-classes:

Custom task classes
===================

All tasks inherit from the :class:`@Task` class.
The :meth:`~@Task.run` method becomes the task body.

As an example, the following code,

.. code-block:: python

    @app.task
    def add(x, y):
        return x + y


will do roughly this behind the scenes:

.. code-block:: python

    class _AddTask(app.Task):

        def run(self, x, y):
            return x + y
    add = app.tasks[_AddTask.name]


Instantiation
-------------

A task is **not** instantiated for every request, but is registered
in the task registry as a global instance.

This means that the ``__init__`` constructor will only be called
once per process, and that the task class is semantically closer to an
Actor.

If you have a task,

.. code-block:: python

    from celery import Task

    class NaiveAuthenticateServer(Task):

        def __init__(self):
            self.users = {'george': 'password'}

        def run(self, username, password):
            try:
                return self.users[username] == password
            except KeyError:
                return False

And you route every request to the same process, then it
will keep state between requests.

This can also be useful to cache resources,
e.g. a base Task class that caches a database connection:

.. code-block:: python

    from celery import Task

    class DatabaseTask(Task):
        abstract = True
        _db = None

        @property
        def db(self):
            if self._db is None:
                self._db = Database.connect()
            return self._db


that can be added to tasks like this:

.. code-block:: python


    @app.task(base=DatabaseTask)
    def process_rows():
        for row in process_rows.db.table.all():
            …

The ``db`` attribute of the ``process_rows`` task will then
always stay the same in each process.

Abstract classes
----------------

Abstract classes are not registered, but are used as the
base class for new task types.

.. code-block:: python

    from celery import Task

    class DebugTask(Task):
        abstract = True

        def after_return(self, *args, **kwargs):
            print('Task returned: {0!r}'.format(self.request))


    @app.task(base=DebugTask)
    def add(x, y):
        return x + y


Handlers
--------

.. method:: after_return(self, status, retval, task_id, args, kwargs, einfo)

    Handler called after the task returns.

    :param status: Current task state.
    :param retval: Task return value/exception.
    :param task_id: Unique id of the task.
    :param args: Original arguments for the task that returned.
    :param kwargs: Original keyword arguments for the task
                   that returned.

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                    instance, containing the traceback (if any).

    The return value of this handler is ignored.

.. method:: on_failure(self, exc, task_id, args, kwargs, einfo)

    This is run by the worker when the task fails.

    :param exc: The exception raised by the task.
    :param task_id: Unique id of the failed task.
    :param args: Original arguments for the task that failed.
    :param kwargs: Original keyword arguments for the task
                       that failed.

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                           instance, containing the traceback.

    The return value of this handler is ignored.

.. method:: on_retry(self, exc, task_id, args, kwargs, einfo)

    This is run by the worker when the task is to be retried.

    :param exc: The exception sent to :meth:`~@Task.retry`.
    :param task_id: Unique id of the retried task.
    :param args: Original arguments for the retried task.
    :param kwargs: Original keyword arguments for the retried task.

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                    instance, containing the traceback.

    The return value of this handler is ignored.

.. method:: on_success(self, retval, task_id, args, kwargs)

    Run by the worker if the task executes successfully.

    :param retval: The return value of the task.
    :param task_id: Unique id of the executed task.
    :param args: Original arguments for the executed task.
    :param kwargs: Original keyword arguments for the executed task.

    The return value of this handler is ignored.

on_retry
~~~~~~~~

.. _task-how-they-work:

How it works
============

Here comes the technical details, this part isn't something you need to know,
but you may be interested.

All defined tasks are listed in a registry.  The registry contains
a list of task names and their task classes.  You can investigate this registry
yourself:

.. code-block:: python

    >>> from proj.celery import app
    >>> app.tasks
    {'celery.chord_unlock':
        <@task: celery.chord_unlock>,
     'celery.backend_cleanup':
        <@task: celery.backend_cleanup>,
     'celery.chord':
        <@task: celery.chord>}

This is the list of tasks built-in to celery.  Note that tasks
will only be registered when the module they are defined in is imported.

The default loader imports any modules listed in the
:setting:`CELERY_IMPORTS` setting.

The entity responsible for registering your task in the registry is the
metaclass: :class:`~celery.task.base.TaskType`.

If you want to register your task manually you can mark the
task as :attr:`~@Task.abstract`:

.. code-block:: python

    class MyTask(Task):
        abstract = True

This way the task won't be registered, but any task inheriting from
it will be.

When tasks are sent, no actual function code is sent with it, just the name
of the task to execute.  When the worker then receives the message it can look
up the name in its task registry to find the execution code.

This means that your workers should always be updated with the same software
as the client.  This is a drawback, but the alternative is a technical
challenge that has yet to be solved.

.. _task-best-practices:

Tips and Best Practices
=======================

.. _task-ignore_results:

Ignore results you don't want
-----------------------------

If you don't care about the results of a task, be sure to set the
:attr:`~@Task.ignore_result` option, as storing results
wastes time and resources.

.. code-block:: python

    @app.task(ignore_result=True)
    def mytask(…):
        something()

Results can even be disabled globally using the :setting:`CELERY_IGNORE_RESULT`
setting.

.. _task-disable-rate-limits:

Disable rate limits if they're not used
---------------------------------------

Disabling rate limits altogether is recommended if you don't have
any tasks using them.  This is because the rate limit subsystem introduces
quite a lot of complexity.

Set the :setting:`CELERY_DISABLE_RATE_LIMITS` setting to globally disable
rate limits:

.. code-block:: python

    CELERY_DISABLE_RATE_LIMITS = True

You find additional optimization tips in the
:ref:`Optimizing Guide <guide-optimizing>`.

.. _task-synchronous-subtasks:

Avoid launching synchronous subtasks
------------------------------------

Having a task wait for the result of another task is really inefficient,
and may even cause a deadlock if the worker pool is exhausted.

Make your design asynchronous instead, for example by using *callbacks*.

**Bad**:

.. code-block:: python

    @app.task
    def update_page_info(url):
        page = fetch_page.delay(url).get()
        info = parse_page.delay(url, page).get()
        store_page_info.delay(url, info)

    @app.task
    def fetch_page(url):
        return myhttplib.get(url)

    @app.task
    def parse_page(url, page):
        return myparser.parse_document(page)

    @app.task
    def store_page_info(url, info):
        return PageInfo.objects.create(url, info)


**Good**:

.. code-block:: python

    def update_page_info(url):
        # fetch_page -> parse_page -> store_page
        chain = fetch_page.s() | parse_page.s() | store_page_info.s(url)
        chain()

    @app.task()
    def fetch_page(url):
        return myhttplib.get(url)

    @app.task()
    def parse_page(page):
        return myparser.parse_document(page)

    @app.task(ignore_result=True)
    def store_page_info(info, url):
        PageInfo.objects.create(url=url, info=info)


Here I instead created a chain of tasks by linking together
different :func:`~celery.subtask`'s.
You can read about chains and other powerful constructs
at :ref:`designing-workflows`.

.. _task-performance-and-strategies:

Performance and Strategies
==========================

.. _task-granularity:

Granularity
-----------

The task granularity is the amount of computation needed by each subtask.
In general it is better to split the problem up into many small tasks, than
have a few long running tasks.

With smaller tasks you can process more tasks in parallel and the tasks
won't run long enough to block the worker from processing other waiting tasks.

However, executing a task does have overhead. A message needs to be sent, data
may not be local, etc. So if the tasks are too fine-grained the additional
overhead may not be worth it in the end.

.. seealso::

    The book `Art of Concurrency`_ has a section dedicated to the topic
    of task granularity [AOC1]_.

.. _`Art of Concurrency`: http://oreilly.com/catalog/9780596521547

.. [AOC1] Breshears, Clay. Section 2.2.1, "The Art of Concurrency".
   O'Reilly Media, Inc. May 15, 2009.  ISBN-13 978-0-596-52153-0.

.. _task-data-locality:

Data locality
-------------

The worker processing the task should be as close to the data as
possible.  The best would be to have a copy in memory, the worst would be a
full transfer from another continent.

If the data is far away, you could try to run another worker at location, or
if that's not possible - cache often used data, or preload data you know
is going to be used.

The easiest way to share data between workers is to use a distributed cache
system, like `memcached`_.

.. seealso::

    The paper `Distributed Computing Economics`_ by Jim Gray is an excellent
    introduction to the topic of data locality.

.. _`Distributed Computing Economics`:
    http://research.microsoft.com/pubs/70001/tr-2003-24.pdf

.. _`memcached`: http://memcached.org/

.. _task-state:

State
-----

Since celery is a distributed system, you can't know in which process, or
on what machine the task will be executed.  You can't even know if the task will
run in a timely manner.

The ancient async sayings tells us that “asserting the world is the
responsibility of the task”.  What this means is that the world view may
have changed since the task was requested, so the task is responsible for
making sure the world is how it should be;  If you have a task
that re-indexes a search engine, and the search engine should only be
re-indexed at maximum every 5 minutes, then it must be the tasks
responsibility to assert that, not the callers.

Another gotcha is Django model objects.  They shouldn't be passed on as
arguments to tasks.  It's almost always better to re-fetch the object from
the database when the task is running instead,  as using old data may lead
to race conditions.

Imagine the following scenario where you have an article and a task
that automatically expands some abbreviations in it:

.. code-block:: python

    class Article(models.Model):
        title = models.CharField()
        body = models.TextField()

    @app.task
    def expand_abbreviations(article):
        article.body.replace('MyCorp', 'My Corporation')
        article.save()

First, an author creates an article and saves it, then the author
clicks on a button that initiates the abbreviation task::

    >>> article = Article.objects.get(id=102)
    >>> expand_abbreviations.delay(article)

Now, the queue is very busy, so the task won't be run for another 2 minutes.
In the meantime another author makes changes to the article, so
when the task is finally run, the body of the article is reverted to the old
version because the task had the old body in its argument.

Fixing the race condition is easy, just use the article id instead, and
re-fetch the article in the task body:

.. code-block:: python

    @app.task
    def expand_abbreviations(article_id):
        article = Article.objects.get(id=article_id)
        article.body.replace('MyCorp', 'My Corporation')
        article.save()

    >>> expand_abbreviations(article_id)

There might even be performance benefits to this approach, as sending large
messages may be expensive.

.. _task-database-transactions:

Database transactions
---------------------

Let's have a look at another example:

.. code-block:: python

    from django.db import transaction

    @transaction.commit_on_success
    def create_article(request):
        article = Article.objects.create(…)
        expand_abbreviations.delay(article.pk)

This is a Django view creating an article object in the database,
then passing the primary key to a task.  It uses the `commit_on_success`
decorator, which will commit the transaction when the view returns, or
roll back if the view raises an exception.

There is a race condition if the task starts executing
before the transaction has been committed; The database object does not exist
yet!

The solution is to *always commit transactions before sending tasks
depending on state from the current transaction*:

.. code-block:: python

    @transaction.commit_manually
    def create_article(request):
        try:
            article = Article.objects.create(…)
        except:
            transaction.rollback()
            raise
        else:
            transaction.commit()
            expand_abbreviations.delay(article.pk)

.. note::
    Django 1.6 (and later) now enables autocommit mode by default,
    and ``commit_on_success``/``commit_manually`` are deprecated.

    This means each SQL query is wrapped and executed in individual
    transactions, making it less likely to experience the
    problem described above.

    However, enabling ``ATOMIC_REQUESTS`` on the database
    connection will bring back the transaction-per-request model and the
    race condition along with it.  In this case, the simple solution is
    using the ``@transaction.non_atomic_requests`` decorator to go back
    to autocommit for that view only.

.. _task-example:

Example
=======

Let's take a real world example; A blog where comments posted needs to be
filtered for spam.  When the comment is created, the spam filter runs in the
background, so the user doesn't have to wait for it to finish.

I have a Django blog application allowing comments
on blog posts.  I'll describe parts of the models/views and tasks for this
application.

blog/models.py
--------------

The comment model looks like this:

.. code-block:: python

    from django.db import models
    from django.utils.translation import ugettext_lazy as _


    class Comment(models.Model):
        name = models.CharField(_('name'), max_length=64)
        email_address = models.EmailField(_('email address'))
        homepage = models.URLField(_('home page'),
                                   blank=True, verify_exists=False)
        comment = models.TextField(_('comment'))
        pub_date = models.DateTimeField(_('Published date'),
                                        editable=False, auto_add_now=True)
        is_spam = models.BooleanField(_('spam?'),
                                      default=False, editable=False)

        class Meta:
            verbose_name = _('comment')
            verbose_name_plural = _('comments')


In the view where the comment is posted, I first write the comment
to the database, then I launch the spam filter task in the background.

.. _task-example-blog-views:

blog/views.py
-------------

.. code-block:: python

    from django import forms
    from django.http import HttpResponseRedirect
    from django.template.context import RequestContext
    from django.shortcuts import get_object_or_404, render_to_response

    from blog import tasks
    from blog.models import Comment


    class CommentForm(forms.ModelForm):

        class Meta:
            model = Comment


    def add_comment(request, slug, template_name='comments/create.html'):
        post = get_object_or_404(Entry, slug=slug)
        remote_addr = request.META.get('REMOTE_ADDR')

        if request.method == 'post':
            form = CommentForm(request.POST, request.FILES)
            if form.is_valid():
                comment = form.save()
                # Check spam asynchronously.
                tasks.spam_filter.delay(comment_id=comment.id,
                                        remote_addr=remote_addr)
                return HttpResponseRedirect(post.get_absolute_url())
        else:
            form = CommentForm()

        context = RequestContext(request, {'form': form})
        return render_to_response(template_name, context_instance=context)


To filter spam in comments I use `Akismet`_, the service
used to filter spam in comments posted to the free weblog platform
`Wordpress`.  `Akismet`_ is free for personal use, but for commercial use you
need to pay.  You have to sign up to their service to get an API key.

To make API calls to `Akismet`_ I use the `akismet.py`_ library written by
`Michael Foord`_.

.. _task-example-blog-tasks:

blog/tasks.py
-------------

.. code-block:: python

    from celery import Celery

    from akismet import Akismet

    from django.core.exceptions import ImproperlyConfigured
    from django.contrib.sites.models import Site

    from blog.models import Comment


    app = Celery(broker='amqp://')


    @app.task
    def spam_filter(comment_id, remote_addr=None):
        logger = spam_filter.get_logger()
        logger.info('Running spam filter for comment %s', comment_id)

        comment = Comment.objects.get(pk=comment_id)
        current_domain = Site.objects.get_current().domain
        akismet = Akismet(settings.AKISMET_KEY, 'http://{0}'.format(domain))
        if not akismet.verify_key():
            raise ImproperlyConfigured('Invalid AKISMET_KEY')


        is_spam = akismet.comment_check(user_ip=remote_addr,
                            comment_content=comment.comment,
                            comment_author=comment.name,
                            comment_author_email=comment.email_address)
        if is_spam:
            comment.is_spam = True
            comment.save()

        return is_spam

.. _`Akismet`: http://akismet.com/faq/
.. _`akismet.py`: http://www.voidspace.org.uk/downloads/akismet.py
.. _`Michael Foord`: http://www.voidspace.org.uk/
