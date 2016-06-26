.. _guide-webhooks:

================================
 HTTP Callback Tasks (Webhooks)
================================

.. module:: celery.task.http

.. contents::
    :local:

.. _webhook-basics:

Basics
======

如果你需要调用另一个语言、框架或类似的东西，你可以使用HTTP callback tasks。


HTTP callback tasks使用GET/POST数据去传递参数，并一JSON作为响应数据。
以这种方式调用task如下::

    GET http://example.com/mytask/?arg1=a&arg2=b&arg3=c

或使用POST请求::

    POST http://example.com/mytask

.. note::

    POST的数据需要使用form encoded。

使用GET还是POST，取决与你以及你的需求。

如果执行成功，web页面应该以如下格式返回一个响应::

    {'status': 'success', 'retval': …}

如果执行失败，则::

    {'status': 'failure', 'reason': 'Invalid moon alignment.'}

Enabling the HTTP task
----------------------

为了启用HTTP 派发任务，你必须增加:mod:`celery.task.http`到配置项
:setting:`CELERY_IMPORTS`中，或者在启动worker的时候使用参数 ``-I celery.task.http``。


.. _webhook-django-example:

Django webhook example
======================

利用以上信息，你可以在Django中定义一个简单的`task`:

.. code-block:: python

    from django.http import HttpResponse
    from anyjson import serialize


    def multiply(request):
        x = int(request.GET['x'])
        y = int(request.GET['y'])
        result = x * y
        response = {'status': 'success', 'retval': result}
        return HttpResponse(serialize(response), mimetype='application/json')

.. _webhook-rails-example:

Ruby on Rails webhook example
=============================

or in Ruby on Rails:

.. code-block:: ruby

    def multiply
        @x = params[:x].to_i
        @y = params[:y].to_i

        @status = {:status => 'success', :retval => @x * @y}

        render :json => @status
    end

你可以轻松的移植这个方案到任何语言/框架；欢迎更多的样例和库示例。

.. _webhook-calling:

Calling webhook tasks
=====================

使用 :class:`~celery.task.http.URL`类，调用`task`：

    >>> from celery.task.http import URL
    >>> res = URL('http://example.com/multiply').get_async(x=10, y=10)


:class:`~celery.task.http.URL`是使用 :class:`HttpDispatchTask` 的快捷方式。
你可以子类化它去扩展功能。

    >>> from celery.task.http import HttpDispatchTask
    >>> res = HttpDispatchTask.delay(
    ...     url='http://example.com/multiply',
    ...     method='GET', x=10, y=10)
    >>> res.get()
    100

:program:`celery worker` 的输出（或者日志文件，如果启用了）,应该显示这个被执行了的`task`::

    [INFO/MainProcess] Task celery.task.http.HttpDispatchTask
            [f2cc8efc-2a14-40cd-85ad-f1c77c94beeb] processed: 100

由于可以通过HTTP使用:func:`djcelery.views.apply` 来调用`task`，所以从其他语言调用`task`时非常容易的。
你应该在Celery的发布页面(http://github.com/celery/celery/tree/master/examples/celery_http_gateway/)
中阅读`examples/celery_http_gateway` 来获取一个使用HTTP的task示例。
