.. _concurrency-eventlet:

===========================
 Concurrency with Eventlet
===========================

.. _eventlet-introduction:

Introduction
============

`Evenlet`_ 的主页有详细的描述.
Evenlet是一个改变你代码如何运行，而不是改变你如何写代码的 Python并发网络库。

    * 它使用`epool(4)`_ 或 `libevent`_ 作为高伸缩性的非阻塞网络IO(`highly scalable non-blocking I/O`_)。
    * `Coroutines`_ 使开发者使用类似于多线程的阻塞模型编程，但是却享有非阻塞I/O的优势。
    * 事件派发是隐式完成的，这意味着你可以轻松的在Python解释器中使用`Eventlet`，
      亦或作为大型应用程序的一部分。

Celery支持`Evenlet`作为一个可选的执行池（excution pool）实现。
它相比`Prefork`会有一些优势，但是你必须保证你的`task`不会做任何阻塞的调用，
因为阻塞调用将halt这个worker内的所有其它操作，直到这次阻塞调用完成。

`Prefork pool`可以利用多进程，然而多进程的数量限制在每个CPU少量的进程。
而使用`Evenlet`你可以高效的生成成百上千的`green thread`。
一个非正式的测试（feed hub system）表明：`Evenlet pool`每秒可以取出并处理数百个feeds，
但是`prefork pool`需要花费14秒去处理100个feeds。注意，这仅仅是采用了事件驱动I/O
的应用程序的优势之一（之一指：异步HTTP请求）。你可能想要结合`Evenlet`和`prefork`，
并且根据兼容性或能工作的最好去路由`task`。
You may want a mix of both Eventlet and
prefork workers, and route tasks according to compatibility or
what works best.

Enabling Eventlet
=================

你可以为:program:`celery worker` 使用``-P``选项，来启用 `Evenlet pool`:

.. code-block:: bash

    $ celery -A proj worker -P eventlet -c 1000

.. _eventlet-examples:

Examples
========

查看Celery发布中心的`Eventlet examples`_ 目录，了解更多使用Eventlet的样例。

.. _`Eventlet`: http://eventlet.net
.. _`epoll(4)`: http://linux.die.net/man/4/epoll
.. _`libevent`: http://monkey.org/~provos/libevent/
.. _`highly scalable non-blocking I/O`:
    http://en.wikipedia.org/wiki/Asynchronous_I/O#Select.28.2Fpoll.29_loops
.. _`Coroutines`: http://en.wikipedia.org/wiki/Coroutine
.. _`Eventlet examples`:
    https://github.com/celery/celery/tree/master/examples/eventlet

