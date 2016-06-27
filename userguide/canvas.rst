.. _guide-canvas:

=============================
 Canvas: Designing Workflows(画板：设计工作流)
=============================

.. contents::
    :local:
    :depth: 2

.. _canvas-subtasks:

.. _canvas-signatures:

Signatures
==========

.. versionadded:: 2.0

你刚刚在 :ref:`calling <guide-calling>` 章节学习了如何使用``delay``方法调用一个`task`，
当然这也是你最常需要的。但是有时你可能想传递task invocation的`signature`名到另一个process，
或者作为一个参数传递给另一个函数。

:func:`~celery.signature`函数封装了位置参数、关键字参数、以及task的执行选项，
以便能够传递给函数 或者 甚至序列化后通过网络发送出去。

`signatures`通常被称作"subtasks"，因为他们使用一个task，描述另一个要被调用的task。



- 你能够以如下方式，使用它的名字为``add``task创建一个`signature`::

        >>> from celery import signature
        >>> signature('tasks.add', args=(2, 2), countdown=10)
        tasks.add(2, 2)

  这个`signature`包含2个参数： 位置参数``（2, 2）``、设置为10的执行选项`countdown`。


- 或者使用``subtask``方法来创建一个`signature`::

        >>> add.subtask((2, 2), countdown=10)
        tasks.add(2, 2)

- 同样也有使用star arguments的快捷方式::

        >>> add.s(2, 2)
        tasks.add(2, 2)

- 关键字参数也是被支持的::

        >>> add.s(2, 2, debug=True)
        tasks.add(2, 2, debug=True)


- 对于任意一个`signature`实例，你可以inspect不同的字段::

        >>> s = add.subtask((2, 2), {'debug': True}, countdown=10)
        >>> s.args
        (2, 2)
        >>> s.kwargs
        {'debug': True}
        >>> s.options
        {'countdown': 10}


- `signature`支持"Calling API",这意味着它支持``delay``、``apply_async``以及被直接调用::

    直接调用这个签名，将会在当前进程中执行这个task::

        >>> add(2, 2)
        4
        >>> add.s(2, 2)()
        4


  ``delay``是我们最爱的针对``apply_async``的快捷方式::

        >>> result = add.delay(2, 2)
        >>> result.get()
        4


  ``apply_async`` 携带和 :meth:`Task.apply_async <@Task.apply_async>`方法同样的参数::

        >>> add.apply_async(args, kwargs, **options)
        >>> add.subtask(args, kwargs, **options).apply_async()

        >>> add.apply_async((2, 2), countdown=1)
        >>> add.subtask((2, 2), countdown=1).apply_async()


- 虽然你不能通过 :meth:`~@Task.s`直接定义执行选项，但是链式调用``set``方法可以做同样的事::

    >>> add.s(2, 2).set(countdown=1)
    proj.tasks.add(2, 2)

Partials
--------

使用`signature`，你可以在一个`worker`中执行这个`task`::

    >>> add.s(2, 2).delay()
    >>> add.s(2, 2).apply_async(countdown=1)

或者你可以直接在当前进程中调用它::

    >>> add.s(2, 2)()
    4

给``apply_async``/``delay``方法，指定位置参数、关键字参数和执行选项来创建partials:

- 任何添加的位置参数将会prepended到这个`signature`的`args`中::

    >>> partial = add.s(2)          # incomplete signature
    >>> partial.delay(4)            # 4 + 2
    >>> partial.apply_async((4,))  # same

- 任何添加的关键字参数将会被合并到这个`signature`的`kwargs`中，后添加覆盖先添加的::

    >>> s = add.s(2, 2)
    >>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
    >>> s.apply_async(kwargs={'debug': True})  # same

- 任何添加的执行选项将会被合并到这个`signature`的`options`中，后添加的覆盖先添加的::

    >>> s = add.subtask((2, 2), countdown=10)
    >>> s.apply_async(countdown=1)  # countdown is now 1

你可以可以克隆`signature`来创建一个衍生物:

    >>> s = add.s(2)
    proj.tasks.add(2)

    >>> s.clone(args=(4,), kwargs={'debug': True})
    proj.tasks.add(4, 2, debug=True)

Immutability
------------

.. versionadded:: 3.0

partials打算被用在callbacks、任一tasks linked、携带父任务结果调用的chrod callbacks。
有时你想指定指定一个callback —— 不接受额外的参数，在这种情况下你可以设置这个签名为不可变的::

    >>> add.apply_async((2, 2), link=reset_buffers.subtask(immutable=True))

``.si()`` 快捷方式也可以被用来创建一个不可变`signature`::

    >>> add.apply_async((2, 2), link=reset_buffers.si())

当一个`signature`是不可变时，只有执行选项可以被设置，所以使用partial args/kwargs调用这个
`signature`是不可能的。

.. note::

    在这篇入门中，我常常对`signature`使用操作符`~`。
    你不应该在你的生产环境中这样使用，而是仅仅作为在Python shell中体验功能的一个便捷方式::

        >>> ~sig

        >>> # is the same as
        >>> sig.delay().get()


.. _canvas-callbacks:

Callbacks
---------

.. versionadded:: 3.0

Caalbacks可以使用``apply_async``方法的``link``参数，添加到任一`task`::

    add.apply_async((2, 2), link=other_task.s())

这个回调函数仅当这个task执行成功时被调用，并且将携带这个父task的返回值作为参数调用这个回调函数。

就我先前提到的，任一添加到`signature`中的参数，将会被prepended这个签名的args中。
(译者注： 后添加的参数，会在args的左边)

假设你有这样的一个`siganture`::

    >>> sig = add.s(10)

然后`sig.delay(result)` 变为::

    >>> add.apply_async(args=(result, 10))

...

现在，我们使用partial参数的callback，作为callback来调用我们的``add`` task::

    >>> add.apply_async((2, 2), link=add.s(8))

如我们所期待的，首先将会生成一个task去计算 :math:`2 + 2`，然后另一个任务去计算 :math:`4 + 8`.
（译者注： 第一个task返回的结果，成为了第二个task的最左边的参数）

The Primitives
==============

.. versionadded:: 3.0

.. topic:: Overview

    - ``group``

        携带一个需要被并行调用的task的*列表*作为参数的`signature`。

    - ``chain``

        链原语使我们能将多个`signature`连接在一起，以便一个接着一个的被调用，本质上来说是生成一个回调函数*调用链*。

    - ``chord``

        chord类似group，但是带有回调函数。chord由一个头部group和主体构成：
        在所有头部中包含的task执行完成之后，主体task将被调用。

    - ``map``

        map原语的工作方式类似于内建的``map``函数，不同之处是：
        会创建一个临时的task —— 需要一个list参数。
        比如： ``task.map([1, 2])`` 导致一个单独的task被调用，list中的参数会按需传递给这个task并调用，
        即结果是::

            res = [task(1), task(2)]

    - ``starmap``

        除了传输会被作为``*args``传入以外，工作方式和map类似。
        例如``add.starmap([(2, 2), (4, 4)])的结果为一个单一task被调用::

            res = [add(2, 2), add(4, 4)]

    - ``chunks``

        chunking 分割一个长的参数列表为小部分，比如这个操作::

            >>> items = zip(xrange(1000), xrange(1000))  # 1000 items
            >>> add.chunks(items, 10)

        将会分割items这个列表为多个chunk（每个包含10个元素），即产生100个task。
        （每个task处理这个列表中的10个元素）


这些原语他们自己本身也是一个`signature`，所以他们能以任意数量、任意方式组合成更复杂的工作流。

这是一些例子：

- Simple chain

    这是一个简单的链，第一个执行的task，传递它的返回值给在chain中的下一个task，以此类推。

    .. code-block:: python

        >>> from celery import chain

        # 2 + 2 + 4 + 8
        >>> res = chain(add.s(2, 2), add.s(4), add.s(8))()
        >>> res.get()
        16

    也可以用管道的方式来书写::

        >>> (add.s(2, 2) | add.s(4) | add.s(8))().get()
        16

- Immutable signatures

    `signature`可以是partial，所以参数到`args`或`kwargs`中，但有时你可能不想这样，
    比如：你不想task链中的上一个task的结果，作为这个task的参数。

    在这种情况下，你可以标记这个签名为不可变的，以便这个task的参数不能再发生改变::

        >>> add.subtask((2, 2), immutable=True)

    同样的``.si``更方便的做这件事::

        >>> add.si(2, 2)

    现在你可以创建一个各个task相互独立的task链::

        >>> res = (add.si(2, 2) | add.si(4, 4) | add.s(8, 8))()
        >>> res.get()
        16

        >>> res.parent.get()
        8

        >>> res.parent.parent.get()
        4

- Simple group

    你可以轻松的创建一个task的group去并行的执行这些task::

        >>> from celery import group
        >>> res = group(add.s(i, i) for i in xrange(10))()
        >>> res.get(timeout=1)
        [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

- Simple chord

    chord原语让我们能够增加一个回调函数 —— 当group中的所有task执行完成之后被调用，
    这常常被用于一些难以并发的算法中::

        >>> from celery import chord
        >>> res = chord((add.s(i, i) for i in xrange(10)), xsum.s())()
        >>> res.get()
        90

    上面的这个例子创建了10个会同时执行的task，当它们全部完成后，
    这些task的返回值会组成一个列表，传递给``xsum``task。

    chord的body同样可以是不可变的， 以便group的返回值不会传递给这个回调函数::

        >>> chord((import_contact.s(c) for c in contacts),
        ...       notify_complete.si(import_id)).apply_async()

    注意，上面使用的``.si``方法讲创建一个不可便的`signature`。

- Blow your mind by combining

    同样地，`chain`也可以被partial::

        >>> c1 = (add.s(4) | mul.s(8))

        # (16 + 4) * 8
        >>> res = c1(16)
        >>> res.get()
        160

    这意味着，你可以组合多个chain::

        # ((4 + 16) * 2 + 4) * 8
        >>> c2 = (add.s(4, 16) | mul.s(2) | (add.s(4) | mul.s(8)))

        >>> res = c2()
        >>> res.get()
        352

    把一个`group`和另一个`task`组合为一个链，将自动升级为`chord`::

        >>> c3 = (group(add.s(i, i) for i in xrange(10)) | xsum.s())
        >>> res = c3()
        >>> res.get()
        90

    `group`和`chord`同样接受partial参数，所以，链中的上一个`task`的返回值,
    将传递给`chain`中的下一个`group`中的所有`task`::


        >>> new_user_workflow = (create_user.s() | group(
        ...                      import_contacts.s(),
        ...                      send_welcome_email.s()))
        ... new_user_workflow.delay(username='artv',
        ...                         first='Art',
        ...                         last='Vandelay',
        ...                         email='art@vandelay.com')


    如果你不想把上一个task的返回值传递给这个group，你可以设置这个group中的所有`signature`为不可变::

        >>> res = (add.s(4, 4) | group(add.si(i, i) for i in xrange(10)))()
        >>> res.get()
        <GroupResult: de44df8c-821d-4c84-9a6a-44769c738f98 [
            bc01831b-9486-4e51-b046-480d7c9b78de,
            2650a1b8-32bf-4771-a645-b0a35dcc791b,
            dcbee2a5-e92d-4b03-b6eb-7aec60fd30cf,
            59f92e0a-23ea-41ce-9fad-8645a0e7759c,
            26e1e707-eccf-4bf4-bbd8-1e1729c3cce3,
            2d10a5f4-37f0-41b2-96ac-a973b1df024d,
            e13d3bdb-7ae3-4101-81a4-6f17ee21df2d,
            104b2be0-7b75-44eb-ac8e-f9220bdfa140,
            c5c551a5-0386-4973-aa37-b65cbeb2624b,
            83f72d71-4b71-428e-b604-6f16599a9f37]>

        >>> res.parent.get()
        8


.. _canvas-chain:

Chains
------

.. versionadded:: 3.0

多个task可以被连接在一起，在实践中意味着增加一个回调task::

    >>> res = add.apply_async((2, 2), link=mul.s(16))
    >>> res.get()
    4

被连接的task将会调用——使用父task的结果作为第一个参数，在上一个例子中，
结果为``mul(4, 16)``，因为父task的结果为4。

你也可以使用``link_error``参数,添加*error callbacks*::

    >>> add.apply_async((2, 2), link_error=log_error.s())

    >>> add.subtask((2, 2), link_error=log_error.s())

由于只有当序列化方式使用的是pickle时，异常才能够被序列化，
所以传给error callbacks task的参数是父task的ID:

.. code-block:: python

    from __future__ import print_function
    import os
    from proj.celery import app

    @app.task
    def log_error(task_id):
        result = app.AsyncResult(task_id)
        result.get(propagate=False)  # make sure result written.
        with open(os.path.join('/var/errors', task_id), 'a') as fh:
            print('--\n\n{0} {1} {2}'.format(
                task_id, result.result, result.traceback), file=fh)

为了更加容易的将多个task链接在一起，这里有一个名为 :class:`~celery.chain`
的特殊函数（signature）—— 让你将多个task链在一起:

.. code-block:: python

    >>> from celery import chain
    >>> from proj.tasks import add, mul

    # (4 + 4) * 8 * 10
    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))
    proj.tasks.add(4, 4) | proj.tasks.mul(8) | proj.tasks.mul(10)


调用这个链，将会在当前进程中调用这些task，并且返回这个链中的最后一个task的结果
(译者注： 和apply_async函数中的link参数不同!, link参数指定的callback会被异步调用，即callback.delay())::

    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()
    >>> res.get()
    640

它同时也设置了``parent``属性，以便于你能够获取中间结果::

    >>> res.parent.get()
    64

    >>> res.parent.parent.get()
    8

    >>> res.parent.parent
    <AsyncResult: eeaad925-6778-4ad1-88c8-b2a63d017933>

也可以使用 ``|`` 管道操作符来创建一个`chain`::

    >>> (add.s(2, 2) | mul.s(8) | mul.s(10)).apply_async()

.. note::

    synchronize on groups是不可能的，所以把一个`group`和另一个`signature`链接在一起，
    将自动升级为一个chord:

    .. code-block:: python

        # will actually be a chord when finally evaluated
        res = (group(add.s(i, i) for i in range(10)) | xsum.s()).delay()

Trails
~~~~~~

tasks将在result backend中保持对subtask调用的跟踪（除非使用 :attr:`Task.trail <~@Task.trail>`禁用），
并且这可以从这个结果实例（译者注：返回的AsyncResult对象）中被访问::

    >>> res.children
    [<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>]

    >>> res.children[0].get()
    64

这个结果实例同时也拥有 :meth:`~@AsyncResult.collect`方法
—— 将这个结果视为一个图，让你可以迭代这个结果::

    >>> list(res.collect())
    [(<AsyncResult: 7b720856-dc5f-4415-9134-5c89def5664e>, 4),
     (<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>, 64)]

默认情况下，当这个图没有全部形成时（译者注：没有全部执行完成），:meth:`~@AsyncResult.collect`
将会抛出一个:exc:`~@IncompleteStream` 异常，但是你也可以获得这个图的一个中间状态
（译者注：只包含执行完成了的task）::

    >>> for result, value in res.collect(intermediate=True)):
    ....


Graphs
~~~~~~

另外，你可以将这个结果图视为一个:class:`~celery.datastructures.DependencyGraph`来操作:

.. code-block:: python

    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()

    >>> res.parent.parent.graph
    285fa253-fcf8-42ef-8b95-0078897e83e6(1)
        463afec2-5ed4-4036-b22d-ba067ec64f52(0)
    872c3995-6fa0-46ca-98c2-5a19155afcf0(2)
        285fa253-fcf8-42ef-8b95-0078897e83e6(1)
            463afec2-5ed4-4036-b22d-ba067ec64f52(0)

你也可以将这些图转换为*dot*格式::

    >>> with open('graph.dot', 'w') as fh:
    ...     res.parent.parent.graph.to_dot(fh)


并创建图像:

.. code-block:: bash

    $ dot -Tpng graph.dot -o graph.png

.. image:: ../images/result_graph.png

.. _canvas-group:

Groups
------

.. versionadded:: 3.0

组可以用来并行的执行多个task。

:class:`~celery.group` 函数需要传入一个`signature`的列表::

    >>> from celery import group
    >>> from proj.tasks import add

    >>> group(add.s(2, 2), add.s(4, 4))
    (proj.tasks.add(2, 2), proj.tasks.add(4, 4))

如果你**直接调用**这个`group`，这个 `group` 中的 `task` 将会在当前进程中一个接着一个的被调用，
并且返回一个 :class:`~celery.result.GroupResult` 实例 —— 可以用来保持对结果集的跟踪 或
获取当前有多少个task已经就绪等等::

    >>> g = group(add.s(2, 2), add.s(4, 4))
    >>> res = g()
    >>> res.get()
    [4, 8]

`group`也支持使用迭代器作为参数::

    >>> group(add.s(i, i) for i in xrange(100))()

`group`也是一个`signature`，所以它能被用来和其它`signature`组合使用。

Group Results
~~~~~~~~~~~~~

`group` task返回一个特殊的结果，这个结果工作方式除了它将这个`group`视为一个整体外，
和普通的task结果一样::

    >>> from celery import group
    >>> from tasks import add

    >>> job = group([
    ...             add.s(2, 2),
    ...             add.s(4, 4),
    ...             add.s(8, 8),
    ...             add.s(16, 16),
    ...             add.s(32, 32),
    ... ])

    >>> result = job.apply_async()

    >>> result.ready()  # have all subtasks completed?
    True
    >>> result.successful() # were all subtasks successful?
    True
    >>> result.get()
    [4, 8, 16, 32, 64]

:class:`~celery.result.GroupResult` 携带一个 :class:`~celery.result.AsyncResult` 实例的列表并操作它们，
就像是一个单一的task一样。

它支持以下操作:

* :meth:`~celery.result.GroupResult.successful`

    如果所有subtasks执行成功(即，没有抛出异常)，则返回 :const:`True` .

* :meth:`~celery.result.GroupResult.failed`

    如果任何subtask执行失败，则返回 :const:`True`

* :meth:`~celery.result.GroupResult.waiting`

    如果有subtask还没有就绪，则返回 :const:`True`

* :meth:`~celery.result.GroupResult.ready`

    如果所有subtask都已经就绪，则返回 :const:`True`

* :meth:`~celery.result.GroupResult.completed_count`

    返回已经执行完成的subtasks的数量。

* :meth:`~celery.result.GroupResult.revoke`

    Revoke所有subtasks。

* :meth:`~celery.result.GroupResult.join`

    收集所有subtask的结果，并返回一个列表 —— 以调用时的顺序保存结果。

.. _canvas-chord:

Chords
------

.. versionadded:: 2.3

.. note::

    被用在chord的task，必须*没有*忽略它们的结果。 如果`chord`中的*任一*task（包括头部和主体中的）
    的result backend被禁用，你应该阅读":ref:`chord-important-notes`".


chord是一个task —— 仅当所有在`group`中的`task`执行完成后，才执行的。


让我们来计算这个表达式的和:math:`1+1+2+2+3+3+....+n+n`一直到100.

首先你需要两个task， :func:`add` and :func:`tsum` (:func:`sum` 时标准库函数。 译者注： 避免冲突):

.. code-block:: python

    @app.task
    def add(x, y):
        return x + y

    @app.task
    def tsum(numbers):
        return sum(numbers)


现在你可以使用`chord`去并行的计算每一个加法，然后获得这些结果的和::

    >>> from celery import chord
    >>> from tasks import add, tsum

    >>> chord(add.s(i, i)
    ...       for i in xrange(100))(tsum.s()).get()
    9900


这明显不是一个非常恰当的样例，message的开销以及同步的开销，使这比直接用python要慢得多::

    sum(i + i for i in xrange(100))

同步操作是昂贵的，所以你应该尽可能的避免使用`chord`。
尽管如此，`chord`仍然是一个你值得拥有的强大的原语as synchronization is a required step for many parallel algorithms.


我们来分解`chord`表达式:

.. code-block:: python

    >>> callback = tsum.s()
    >>> header = [add.s(i, i) for i in range(100)]
    >>> result = chord(header)(callback)
    >>> result.get()
    9900

记住，callback仅当所有位于`chord`的header中的`tasks`全部执行完成时被调用。header中的每一步都作为一个`task`被执行
—— 并行地、可能在不同的集群中的节点。 将使用位于header中的所有task的返回值来调用callback。
:meth:`chord`返回的task id时这个callback的task id，所以，你可以等待它完成以及获取最终返回值。
(但是，记住:ref:`never have a task waitfor other tasks <task-synchronous-subtasks>`)

.. _chord-errors:

Error handling
~~~~~~~~~~~~~~

那么如果其中一个task抛出了异常会发生什么呢？

在某些情况下是没有确切的文档的，并且在版本3.1以前，异常会被转发给chord的callback
This was not documented for some time and before version 3.1
the exception value will be forwarded to the chord callback.


版本3.1以后， 错误将扩散(propagate)给callback，所以callback将不会被执行，
而是callback改变为failure状态； 并且这个错误被设置为 :exc:`~@ChordError`异常:

.. code-block:: python

    >>> c = chord([add.s(4, 4), raising_task.s(), add.s(8, 8)])
    >>> result = c()
    >>> result.get()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "*/celery/result.py", line 120, in get
        interval=interval)
      File "*/celery/backends/amqp.py", line 150, in wait_for
        raise meta['result']
    celery.exceptions.ChordError: Dependency 97de6f3f-ea67-4517-a21c-d867c61fcb47
        raised ValueError('something something',)

如果你在使用3.0.14或之后的版本，你可以通过设置选项  :setting:`CELERY_CHORD_PROPAGATES` 启用这个新特性::

    CELERY_CHORD_PROPAGATES = True

然而，traceback信息可能是不一样的，具体依赖于使用的result backend；
你可以看见这个错误描述信息 —— 包括失败的`task`的ID、原始异常描述的字符串。
你可以在``result.traceback``中找到这个原始的traceback。

注意： 剩下的`task`将会继续执行，所以即使中间的task失败了，第三个task（``add.s(8,8)``）强仍然会被执行。
 :exc:`~@ChordError` 仅仅展示第一个失败的`task`（in time）：不会遵循chord头部中的group的task顺序。

.. _chord-important-notes:

Important Notes
~~~~~~~~~~~~~~~

`chord`使用的`task`必须*没有*忽略它们的结果。实际上，这意味着为了使用chords,你必须启用
:const:`CELERY_RESULT_BACKEND`。另外如果 :const:`CELERY_IGNORE_RESULT` 被设置为True，
请确保这些被`chord`使用的独立`task`被定义为 :const:`ignore_result=False`。
这适用于Task subclasses 和 decorated tasks。

Example Task subclass:

.. code-block:: python

    class MyTask(Task):
        abstract = True
        ignore_result = False


Example decorated task:

.. code-block:: python

    @app.task(ignore_result=False)
    def another_task(project):
        do_something()

默认情况下，同步步骤的实现方式是：有一个循环的task，每秒轮询一次这个`group`中的task的完成情况，
calling the signature when ready.

Example implementation:

.. code-block:: python

    from celery import maybe_signature

    @app.task(bind=True)
    def unlock_chord(self, group, callback, interval=1, max_retries=None):
        if group.ready():
            return maybe_signature(callback).delay(group.join())
        raise self.retry(countdown=interval, max_retries=max_retries)


这被除了`Redis`和`Memcached`以外的所有result backend使用，redis和memcached会在每个task完成之后增加一个计数，
然后在这个计数值达到这个`group`的header中的`task`数量的时候调用callback。
*注意*：`chords`和2.2版本以前的`Redis`工作的不是很好；你需要至少更新`Redis`到2.2。

The Redis and Memcached approach is a much better solution, but not easily
implemented in other backends (suggestions welcome!).


.. note::

    如果你正在结合`Redis`使用`chords`，并且也重载了:meth:`Task.after_return()`方法，
    你需要确保调用基类的这个方法，否者`chord`的回调函数不会被调用。
    (译者注： 基类中的after_return应该会处理一些计数问题)

    .. code-block:: python

        def after_return(self, *args, **kwargs):
            do_something()
            super(MyTask, self).after_return(*args, **kwargs)

.. _canvas-map:

Map & Starmap
-------------

:class:`~celery.map` and :class:`~celery.starmap` are built-in tasks
that calls the task for every element in a sequence.

和`group`的不同点在于：

- 只有*一个* `task`被发送出去

- 操作是串行化的

例如，使用 ``map``:

.. code-block:: python

    >>> from proj.tasks import add

    >>> ~xsum.map([range(10), range(100)])
    [45, 4950]

等效于:

.. code-block:: python

    @app.task
    def temp():
        return [xsum(range(10)), xsum(range(100))]

以及使用 ``starmap``::

    >>> ~add.starmap(zip(range(10), range(10)))
    [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

is the same as having a task doing:

.. code-block:: python

    @app.task
    def temp():
        return [add(i, i) for i in range(10)]


``map`` 和 ``starmap`` 都是`signature`, 所以它们能被作为`signature`使用，以及在`group`中组合等等。
例如：在10秒后调用这个starmap::

    >>> add.starmap(zip(range(10), range(10))).apply_async(countdown=10)

.. _canvas-chunks:

Chunks
------

`chunking` 让你将一个可迭代的工作切割为多个分片，因此：
如果你有100万的对象，你可以创建10个拥有10万对象的`task`。

有人可能会担心分割task可能会导致并行退化，但是对于一个繁忙的集群来说几乎是不可能的，
并且由于避免了(减少了)message的负载开销，还有可能带来性能的好处。

你可以使用:meth:`@Task.chunks`来创建一个`chunk` `signature`.

.. code-block:: python

    >>> add.chunks(zip(range(100), range(100)), 10)

如 :class:`~celery.group` 一样，当这个`chunk`在当前进程中被调用时，将会发送这个message
As with :class:`~celery.group` the act of sending the messages for
the chunks will happen in the current process when called:

.. code-block:: python

    >>> from proj.tasks import add

    >>> res = add.chunks(zip(range(100), range(100)), 10)()
    >>> res.get()
    [[0, 2, 4, 6, 8, 10, 12, 14, 16, 18],
     [20, 22, 24, 26, 28, 30, 32, 34, 36, 38],
     [40, 42, 44, 46, 48, 50, 52, 54, 56, 58],
     [60, 62, 64, 66, 68, 70, 72, 74, 76, 78],
     [80, 82, 84, 86, 88, 90, 92, 94, 96, 98],
     [100, 102, 104, 106, 108, 110, 112, 114, 116, 118],
     [120, 122, 124, 126, 128, 130, 132, 134, 136, 138],
     [140, 142, 144, 146, 148, 150, 152, 154, 156, 158],
     [160, 162, 164, 166, 168, 170, 172, 174, 176, 178],
     [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]

但是，调用``.apply_async``将创建一个专门的`task` —— 这个独立的`task`会在`worker`中被调用::

    >>> add.chunks(zip(range(100), range(100)), 10).apply_async()

你也可以转换`chunks`为`group`::

    >>> group = add.chunks(zip(range(100), range(100)), 10).group()

并为每个`task`插入(skew)一个倒计时::
and with the group skew the countdown of each task by increments
of one::

    >>> group.skew(start=1, stop=10)()

这意味着，第一个`task`为1秒的倒计时，第二个为2秒的倒计时，以此类推
