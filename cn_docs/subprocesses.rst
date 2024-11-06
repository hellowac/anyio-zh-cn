使用子进程
==================

**Using subprocesses**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        AnyIO 允许您在子进程中运行任意可执行文件，既可以作为一次性调用，也可以为您打开一个进程句柄，以便对子进程进行更多控制。

        您可以将命令作为字符串提供，在这种情况下，它会传递给默认的 shell(等同于 :func:`subprocess.run` 中的 ``shell=True``)，也可以将命令作为字符串序列提供(``shell=False``)，在这种情况下，可执行文件是序列中的第一个项目，其余的则是传递给它的参数。

    .. tab:: 英文

        AnyIO allows you to run arbitrary executables in subprocesses, either as a one-shot call
        or by opening a process handle for you that gives you more control over the subprocess.

        You can either give the command as a string, in which case it is passed to your default
        shell (equivalent to ``shell=True`` in :func:`subprocess.run`), or as a sequence of
        strings (``shell=False``) in which case the executable is the first item in the sequence
        and the rest are arguments passed to it.

运行一次性命令
-------------------------

**Running one-shot commands**

.. tabs::

    .. tab:: 中文

        要通过一次调用运行外部命令，使用 :func:`~run_process`::

            from anyio import run_process, run


            async def main():
                result = await run_process('ps')
                print(result.stdout.decode())

            run(main)

        上面的代码在 shell 中运行 ``ps`` 命令。要直接运行它::

            from anyio import run_process, run


            async def main():
                result = await run_process(['ps'])
                print(result.stdout.decode())

            run(main)

    .. tab:: 英文

        To run an external command with one call, use :func:`~run_process`::

            from anyio import run_process, run


            async def main():
                result = await run_process('ps')
                print(result.stdout.decode())

            run(main)

        The snippet above runs the ``ps`` command within a shell. To run it directly::

            from anyio import run_process, run


            async def main():
                result = await run_process(['ps'])
                print(result.stdout.decode())

            run(main)

使用进程
----------------------

**Working with processes**

.. tabs::

    .. tab:: 中文

        当你对与子进程的交互有更复杂的需求时，可以使用 :func:`~open_process` 启动一个子进程::

            from anyio import open_process, run
            from anyio.streams.text import TextReceiveStream


            async def main():
                async with await open_process(['ps']) as process:
                    async for text in TextReceiveStream(process.stdout):
                        print(text)

            run(main)

        有关更多信息，请参阅 :class:`~.abc.Process` 的 API 文档。

    .. tab:: 英文

        When you have more complex requirements for your interaction with subprocesses, you can
        launch one with :func:`~open_process`::

            from anyio import open_process, run
            from anyio.streams.text import TextReceiveStream


            async def main():
                async with await open_process(['ps']) as process:
                    async for text in TextReceiveStream(process.stdout):
                        print(text)

            run(main)

        See the API documentation of :class:`~.abc.Process` for more information.

.. _RunInProcess:

在工作进程中运行函数
-------------------------------------

**Running functions in worker processes**

.. tabs::

    .. tab:: 中文

        当你需要运行 CPU 密集型代码时，使用工作进程比线程更合适，因为当前的 Python 实现无法同时在多个线程中运行 Python 代码。

        对此规则的例外情况包括：

        #. 阻塞的 I/O 操作
        #. 显式释放全局解释器锁(Global Interpreter Lock)的 C 扩展代码

        如果你希望运行的代码不属于此类别，最好使用工作进程，以便充分利用多个 CPU 核心。
        这可以通过使用 :func:`.to_process.run_sync` 来实现::

            import time

            from anyio import run, to_process


            def cpu_intensive_function(arg1, arg2):
                time.sleep(1)
                return arg1 + arg2

            async def main():
                result = await to_process.run_sync(cpu_intensive_function, 'Hello, ', 'world!')
                print(result)

            # 当应用程序使用 run_sync_in_process() 时，这个检查非常重要
            if __name__ == '__main__':
                run(main)

    .. tab:: 英文

        When you need to run CPU intensive code, worker processes are better than threads
        because current implementations of Python cannot run Python code in multiple threads at
        once.

        Exceptions to this rule are:

        #. Blocking I/O operations
        #. C extension code that explicitly releases the Global Interpreter Lock

        If the code you wish to run does not belong in this category, it's best to use worker
        processes instead in order to take advantage of multiple CPU cores.
        This is done by using :func:`.to_process.run_sync`::

            import time

            from anyio import run, to_process


            def cpu_intensive_function(arg1, arg2):
                time.sleep(1)
                return arg1 + arg2

            async def main():
                result = await to_process.run_sync(cpu_intensive_function, 'Hello, ', 'world!')
                print(result)

            # This check is important when the application uses run_sync_in_process()
            if __name__ == '__main__':
                run(main)

技术细节
*****************

**Technical details**

.. tabs::

    .. tab:: 中文

        关于传递的参数和返回值，有一些限制：

        * 参数必须是可 pickle 的(使用最高可用协议)
        * 返回值必须是可 pickle 的(使用最高可用协议)
        * 目标可调用对象必须是可导入的(lambda 表达式和内部函数不可用)

        其他注意事项：

        * 即使 ``cancellable=False`` 的调用也可以在请求发送到工作进程之前被取消
        * 如果在工作进程上执行的可取消调用被取消，工作进程将被终止
        * 工作进程会导入父进程的 ``__main__`` 模块，因此必须使用 ``if __name__ == '__main__':`` 来防止无限递归
        * ``sys.stdin``、 ``sys.stdout`` 和 ``sys.stderr`` 会被重定向到 ``/dev/null``，因此 :func:`print` 和 :func:`input` 将无法使用
        * 工作进程在 5 分钟无活动后终止，或者当事件循环结束时终止

        * 在 asyncio 中，必须使用 :func:`asyncio.run` 或 :func:`anyio.run` 进行适当的清理
        * 目前不支持多进程风格的同步原语

    .. tab:: 英文

        There are some limitations regarding the arguments and return values passed:

        * the arguments must be pickleable (using the highest available protocol)
        * the return value must be pickleable (using the highest available protocol)
        * the target callable must be importable (lambdas and inner functions won't work)

        Other considerations:

        * Even ``cancellable=False`` runs can be cancelled before the request has been sent to
        the worker process
        * If a cancellable call is cancelled during execution on the worker process, the worker
        process will be killed
        * The worker process imports the parent's ``__main__`` module, so guarding for any
        import time side effects using ``if __name__ == '__main__':`` is required to avoid
        infinite recursion
        * ``sys.stdin`` and ``sys.stdout``, ``sys.stderr`` are redirected to ``/dev/null`` so
        :func:`print` and :func:`input` won't work
        * Worker processes terminate after 5 minutes of inactivity, or when the event loop is
        finished

        * On asyncio, either :func:`asyncio.run` or :func:`anyio.run` must be used for proper
            cleanup to happen
        * Multiprocessing-style synchronization primitives are currently not available
