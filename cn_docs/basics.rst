基础知识
==========

**The basics**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文
    
        AnyIO 需要 Python 3.8 或更高版本才能运行。建议在开发或使用 AnyIO 时设置 virtualenv_ 。

    .. tab:: 英文

        AnyIO requires Python 3.8 or later to run. It is recommended that you set up a virtualenv_ when developing or playing around with AnyIO.

安装
------------

**Installation**

.. tabs::

    .. tab:: 中文
    
        要安装 AnyIO, 请运行:

        .. code-block:: bash

            pip install anyio

        要安装支持 Trio_ 的版本，可以像这样将其作为附加组件安装：

        .. code-block:: bash

            pip install anyio[trio]

    .. tab:: 英文

        To install AnyIO, run:

        .. code-block:: bash

            pip install anyio

        To install a supported version of Trio_, you can install it as an extra like this:

        .. code-block:: bash

            pip install anyio[trio]

运行异步程序
----------------------

**Running async programs**

.. tabs::

    .. tab:: 中文

        最简单的 AnyIO 程序如下所示::

            from anyio import run


            async def main():
                print('Hello, world!')

            run(main)

        这将在默认后端（asyncio）上运行上述程序。要在另一个支持的后端（如 Trio_ ）上运行，可以使用 ``backend`` 参数，如下所示::

            run(main, backend='trio')

        但 AnyIO 代码并不要求通过 :func:`run` 运行。你也可以直接使用后端库的原生 ``run()`` 函数::

            import sniffio
            import trio
            from anyio import sleep


            async def main():
                print('Hello')
                await sleep(1)
                print("I'm running on", sniffio.current_async_library())

            trio.run(main)

        .. versionchanged:: 4.0.0
            在 ``asyncio`` 后端，对于 Python 3.11 之前的版本， ``anyio.run()`` 现在使用了回溯移植版的
            :class:`asyncio.Runner`。

    .. tab:: 英文

        The simplest possible AnyIO program looks like this::

            from anyio import run


            async def main():
                print('Hello, world!')

            run(main)

        This will run the program above on the default backend (asyncio). To run it on another
        supported backend, say Trio_, you can use the ``backend`` argument, like so::

            run(main, backend='trio')

        But AnyIO code is not required to be run via :func:`run`. You can just as well use the
        native ``run()`` function of the backend library::

            import sniffio
            import trio
            from anyio import sleep


            async def main():
                print('Hello')
                await sleep(1)
                print("I'm running on", sniffio.current_async_library())

            trio.run(main)

        .. versionchanged:: 4.0.0
            On the ``asyncio`` backend, ``anyio.run()`` now uses a back-ported version of
            :class:`asyncio.Runner` on Pythons older than 3.11.

.. _backend options:

后端特定选项
------------------------

**Backend specific options**

.. tabs::

    .. tab:: 中文

        **Asyncio**:

        * 选项见 :class:`asyncio.Runner` 的文档
        * ``use_uvloop`` （``bool``, 默认为 False）：如果可用，使用更快的 uvloop_ 事件循环实现（这是传递 ``loop_factory=uvloop.new_event_loop`` 的简写，如果 ``loop_factory`` 传递了非 ``None`` 的值， 则该选项会被忽略）

        **Trio**：选项请参考
        `官方文档
        <https://trio.readthedocs.io/en/stable/reference-core.html#trio.run>`_

        .. versionchanged:: 3.2.0
            ``use_uvloop`` 的默认值已更改为 ``False``。
        .. versionchanged:: 4.0.0
            ``policy`` 选项已被 ``loop_factory`` 替代。

    .. tab:: 英文

        **Asyncio**:

        * options covered in the documentation of :class:`asyncio.Runner`
        * ``use_uvloop`` (``bool``, default=False): Use the faster uvloop_ event loop implementation, if available (this is a shorthand for passing ``loop_factory=uvloop.new_event_loop``, and is ignored if ``loop_factory`` is passed a value other than ``None``)

        **Trio**: options covered in the
        `official documentation
        <https://trio.readthedocs.io/en/stable/reference-core.html#trio.run>`_

        .. versionchanged:: 3.2.0
            The default value of ``use_uvloop`` was changed to ``False``.
        .. versionchanged:: 4.0.0
            The ``policy`` option was replaced with ``loop_factory``.

.. _uvloop: https://pypi.org/project/uvloop/

使用原生异步库
----------------------------

**Using native async libraries**

.. tabs::

    .. tab:: 中文

        AnyIO 允许你将为 AnyIO 编写的代码与为你选择的异步框架编写的代码混合使用。不过，有一些规则需要记住：

        * 你只能使用与你运行的后端相对应的“原生”库，因此，例如，不能将为 Trio_ 编写的库与为 asyncio 编写的库一起使用。
        * 在 Trio_ 以外的后端上，由这些“原生”库生成的任务不受 AnyIO 强制执行的取消规则的约束。
        * 在 AnyIO 之外生成的线程不能使用 :func:`.from_thread.run` 来调用异步代码。

    .. tab:: 英文

        AnyIO lets you mix and match code written for AnyIO and code written for the
        asynchronous framework of your choice. There are a few rules to keep in mind however:

        * You can only use "native" libraries for the backend you're running, so you cannot, for example, use a library written for Trio_ together with a library written for asyncio.
        * Tasks spawned by these "native" libraries on backends other than Trio_ are not subject to the cancellation rules enforced by AnyIO
        * Threads spawned outside of AnyIO cannot use :func:`.from_thread.run` to call asynchronous code

.. _virtualenv: https://docs.python-guide.org/dev/virtualenvs/
.. _Trio: https://github.com/python-trio/trio
