.. _broker-rabbitmq:

================
使用 RabbitMQ
================

.. contents::
    :local:

.. _installation-and-configuration

安装与配置
============================

RabbitMQ 是默认的中间人，所以除了需要你要使用的中间人实例的 URL 位置，
它并不需要任何额外的依赖或起始配置::

    >>> BROKER_URL = 'amqp://guest:guest@localhost:5672//'

Celery 中间人 URL 的描述和完整的中间人可用配置选项列表见
:ref:`conf-broker-settings` 。

.. _installing-rabbitmq:

安装 RabbitMQ 服务器
==============================

见 RabbitMQ 网站上的 `安装 RabbitMQ <Installing RabbitMQ>`_ 。Mac OS X 用户
，则请见 `在 OS X 上安装 RabbitMQ <rabbitmq-osx-installation>`_ 。

.. _`Installing RabbitMQ`: http://www.rabbitmq.com/install.html

.. note::

    如果你在安装和使用 :program:`rabbitmqctl` 后遇到了 `nodedown` 错
    误，那么这篇文章可以帮助你找到错误的来源:

        http://somic.org/2009/02/19/on-rabbitmqctl-and-badrpcnodedown/

.. _rabbitmq-configuration:

设置 RabbitMQ
-------------------

要使用 Celery，我们需要创建一个 RabbitMQ 用户、一个虚拟主机，并且
允许这个用户访问这个虚拟主机：

.. code-block:: bash

    $ sudo rabbitmqctl add_user myuser mypassword

.. code-block:: bash

    $ sudo rabbitmqctl add_vhost myvhost

.. code-block:: bash

    $ sudo rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"

关于  `访问控制 <access control>`_ 的更多信息见 RabbitMQ 的
`管理员指南 <Admin Guide>`_ 。

.. _`Admin Guide`: http://www.rabbitmq.com/admin-guide.html

.. _`access control`: http://www.rabbitmq.com/admin-guide.html#access-control

.. _rabbitmq-osx-installation:

在 OS X 上安装 RabbitMQ
---------------------------

在 Snow Leopard 上安装 RabbitMQ 最简单的方式就是 `Homebrew`_ ——OS X 上的一款
新颖别致，光彩动人的包管理系统。

在本例中，我们将把 Homebrew 安装到 :file:`/lol` ，但你可以选择任意位置，
如果你想，甚至可以是你的用户根目录，Homebrew 的强大之处之一就是可以重定址。


Homebrew 实际上是一个 `git`_ 仓库，所以要安装 Homebrew，你首先需要安装 git。
从 http://code.google.com/p/git-osx-installer/downloads/list?can=3 下载并安装
磁盘映像。

当安装好 git 后，你终于可以克隆这个仓库到 :file:`/lol` ：

.. code-block:: bash

    $ git clone git://github.com/mxcl/homebrew /lol

Brew 包含一个简单的工具称为 :program:`brew` ，用于安装、移除和查询包。为了使
用它，你需要先把它添加到 :envvar:`PATH` 中。可以把下面的这行添加到你的
:file:`~/.profile` 末尾来实现：

.. code-block:: bash

    export PATH="/lol/bin:/lol/sbin:$PATH"

保存并重新加载：

.. code-block:: bash

    $ source ~/.profile

你终于可以用 :program:`brew` 安装 RabbitMQ 了：

.. code-block:: bash

    $ brew install rabbitmq

.. _`Homebrew`: http://github.com/mxcl/homebrew/
.. _`git`: http://git-scm.org

.. _rabbitmq-osx-system-hostname:

配置系统的主机名
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你使用了 DHCP 服务器，它会给你分配一个随机的主机名，那么你需要设置一个永久
的主机名。这是因为 RabbitMQ 通过主机名来与节点通信。

使用 :program:`scutil` 命令来永久修改主机名：

.. code-block:: bash

    $ sudo scutil --set HostName myhost.local

然后把主机名加入到 :file:`/etc/hosts` 中，这样才能解析到 IP 地址::

    127.0.0.1       localhost myhost myhost.local

如果你启用了 RabbitMQ 服务器，你的 Rabbit 节点现在应被 :program:`rabbitmqctl`
识别为 `rabbit@myhost` 。

.. code-block:: bash

    $ sudo rabbitmqctl status
    Status of node rabbit@myhost ...
    [{running_applications,[{rabbit,"RabbitMQ","1.7.1"},
                        {mnesia,"MNESIA  CXC 138 12","4.4.12"},
                        {os_mon,"CPO  CXC 138 46","2.2.4"},
                        {sasl,"SASL  CXC 138 11","2.1.8"},
                        {stdlib,"ERTS  CXC 138 10","1.16.4"},
                        {kernel,"ERTS  CXC 138 10","2.13.4"}]},
    {nodes,[rabbit@myhost]},
    {running_nodes,[rabbit@myhost]}]
    ...done.

如果你的 DHCP 分配的主机名以 IP 地址开头这就尤其重要（例如
`23.10.112.31.comcast.net` ），因为 RabbitMQ 会试图访问 `rabbit@23` ，
而这是一个非法的主机名。

.. _rabbitmq-osx-start-stop:

启动/停止 RabbitMQ 服务器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

启动服务器：

.. code-block:: bash

    $ sudo rabbitmq-server

你也可以添加 :option:`-detached` 属性来让它在后台运行（注意：只有一
个破折号）：


.. code-block:: bash

    $ sudo rabbitmq-server -detached

永远不要用 :program:`kill` 停止 RabbitMQ 服务器，而是应该用
:program:`rabbitmqctl` 命令：

.. code-block:: bash

    $ sudo rabbitmqctl stop

当服务器正常运行后，你可以继续阅读
When the server is running, you can continue reading :ref:`rabbitmq-configuration` 。