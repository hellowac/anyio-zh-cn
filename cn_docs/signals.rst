接收操作系统信号
==================================

**Receiving operating system signals**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        你可能会偶尔发现以有意义的方式接收发送到应用程序的信号是有用的。例如，当接收到 :py:data:`signal.SIGTERM` 信号时，应用程序应该优雅地关闭。同样， :py:data:`~signal.SIGHUP` 常用于请求应用程序重新加载其配置。

        AnyIO 提供了一个简单的机制来接收你感兴趣的信号::

            import signal

            from anyio import open_signal_receiver, run


            async def main():
                with open_signal_receiver(signal.SIGTERM, signal.SIGHUP) as signals:
                    async for signum in signals:
                        if signum == signal.SIGTERM:
                            return
                        elif signum == signal.SIGHUP:
                            print('Reloading configuration')

            run(main)

        .. note:: 信号处理程序只能在主线程中安装，因此，当事件循环通过 :class:`~.from_thread.BlockingPortal` 运行时，它们将不起作用。

        .. note:: Windows 原生不支持信号，因此在跨平台应用程序中不要依赖此功能。

    .. tab:: 英文

        You may occasionally find it useful to receive signals sent to your application in a
        meaningful way. For example, when you receive a ``signal.SIGTERM`` signal, your
        application is expected to shut down gracefully. Likewise, ``SIGHUP`` is often used as a
        means to ask the application to reload its configuration.

        AnyIO provides a simple mechanism for you to receive the signals you're interested in::

            import signal

            from anyio import open_signal_receiver, run


            async def main():
                with open_signal_receiver(signal.SIGTERM, signal.SIGHUP) as signals:
                    async for signum in signals:
                        if signum == signal.SIGTERM:
                            return
                        elif signum == signal.SIGHUP:
                            print('Reloading configuration')

            run(main)

        .. note:: Signal handlers can only be installed in the main thread, so they will not
        work when the event loop is being run through :class:`~.from_thread.BlockingPortal`,
        for instance.

        .. note:: Windows does not natively support signals so do not rely on this in a cross
        platform application.

处理键盘中断和系统退出
-----------------------------------------

**Handling KeyboardInterrupt and SystemExit**

.. tabs::

    .. tab:: 中文

        默认情况下，不同的后端对 Ctrl+C (或在 Windows 上的 Ctrl+Break)键组合和外部终止( 分别是： ``KeyboardInterrupt`` 和 ``SystemExit`` )的处理方式不同: Trio 在应用程序内部引发相关异常，而 asyncio 则关闭所有任务并退出。如果你需要在这些情况下执行清理工作，你需要安装一个信号处理程序::

            import signal

            from anyio import open_signal_receiver, create_task_group, run
            from anyio.abc import CancelScope


            async def signal_handler(scope: CancelScope):
                with open_signal_receiver(signal.SIGINT, signal.SIGTERM) as signals:
                    async for signum in signals:
                        if signum == signal.SIGINT:
                            print('Ctrl+C pressed!')
                        else:
                            print('Terminated!')

                        scope.cancel()
                        return


            async def main():
                async with create_task_group() as tg:
                    tg.start_soon(signal_handler, tg.cancel_scope)
                    ...  # 继续启动实际的应用程序逻辑

            run(main)

        .. note:: Windows 不支持 :data:`~signal.SIGTERM` 信号，因此，如果你需要在 Windows 上实现优雅的关闭机制，你需要找到其他方法。

    .. tab:: 英文

        By default, different backends handle the Ctrl+C (or Ctrl+Break on Windows) key
        combination and external termination (:exc:`KeyboardInterrupt` and :exc:`SystemExit`,
        respectively) differently: Trio raises the relevant exception inside the application
        while asyncio shuts down all the tasks and exits. If you need to do your own cleanup in
        these situations, you will need to install a signal handler::

            import signal

            from anyio import open_signal_receiver, create_task_group, run
            from anyio.abc import CancelScope


            async def signal_handler(scope: CancelScope):
                with open_signal_receiver(signal.SIGINT, signal.SIGTERM) as signals:
                    async for signum in signals:
                        if signum == signal.SIGINT:
                            print('Ctrl+C pressed!')
                        else:
                            print('Terminated!')

                        scope.cancel()
                        return


            async def main():
                async with create_task_group() as tg:
                    tg.start_soon(signal_handler, tg.cancel_scope)
                    ...  # proceed with starting the actual application logic

            run(main)

        .. note:: Windows does not support the :data:`~signal.SIGTERM` signal so if you need a
        mechanism for graceful shutdown on Windows, you will have to find another way.
