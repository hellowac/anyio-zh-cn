使用线程
====================

**Working with threads**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        实际的异步应用程序有时需要执行网络、文件或计算开销大的操作。这些操作通常会阻塞异步事件循环，导致性能问题。解决方案是将此类代码放在 *工作线程(worker threads)* 中运行。使用工作线程可以让事件循环继续运行其他任务，同时工作线程运行阻塞调用。

    .. tab:: 英文

        Practical asynchronous applications occasionally need to run network, file or
        computationally expensive operations. Such operations would normally block the
        asynchronous event loop, leading to performance issues. The solution is to run such code
        in *worker threads*. Using worker threads lets the event loop continue running other
        tasks while the worker thread runs the blocking call.

在工作线程中运行函数
-------------------------------------

**Running a function in a worker thread**

.. tabs::

    .. tab:: 中文

        在工作线程中运行一个（同步的）可调用对象::

            import time

            from anyio import to_thread, run


            async def main():
                await to_thread.run_sync(time.sleep, 5)

            run(main)

        默认情况下，当任务在等待工作线程完成时，它们会被保护免受取消。您可以传递 ``cancellable=True`` 参数来允许这些任务被取消。然而，请注意，线程仍然会继续运行——只是它的结果将被忽略。

    .. tab:: 英文

        To run a (synchronous) callable in a worker thread::

            import time

            from anyio import to_thread, run


            async def main():
                await to_thread.run_sync(time.sleep, 5)

            run(main)

        By default, tasks are shielded from cancellation while they are waiting for a worker
        thread to finish. You can pass the ``cancellable=True`` parameter to allow such tasks to
        be cancelled. Note, however, that the thread will still continue running – only its
        outcome will be ignored.

.. seealso:: :ref:`RunInProcess`

从工作线程调用异步代码
----------------------------------------------

**Calling asynchronous code from a worker thread**

.. tabs::

    .. tab:: 中文

        如果您需要从工作线程调用一个协程函数，可以这样做::

            from anyio import from_thread, sleep, to_thread, run


            def blocking_function():
                from_thread.run(sleep, 5)


            async def main():
                await to_thread.run_sync(blocking_function)

            run(main)

        .. note:: 必须使用 :func:`~to_thread.run_sync` 启动工作线程，才能使此功能正常工作。

    .. tab:: 英文

        If you need to call a coroutine function from a worker thread, you can do this::

            from anyio import from_thread, sleep, to_thread, run


            def blocking_function():
                from_thread.run(sleep, 5)


            async def main():
                await to_thread.run_sync(blocking_function)

            run(main)

        .. note:: The worker thread must have been spawned using :func:`~to_thread.run_sync` for
        this to work.

从工作线程调用同步代码
---------------------------------------------

**Calling synchronous code from a worker thread**

.. tabs::

    .. tab:: 中文

        有时您可能需要从工作线程在事件循环线程中调用同步代码。常见的情况包括设置异步事件或将数据发送到内存对象流。因为这些方法不是线程安全的，所以您需要安排它们在事件循环线程中调用，使用 :func:`~from_thread.run_sync`::

            import time

            from anyio import Event, from_thread, to_thread, run

            def worker(event):
                time.sleep(1)
                from_thread.run_sync(event.set)

            async def main():
                event = Event()
                await to_thread.run_sync(worker, event)
                await event.wait()

            run(main)

    .. tab:: 英文

        Occasionally you may need to call synchronous code in the event loop thread from a
        worker thread. Common cases include setting asynchronous events or sending data to a
        memory object stream. Because these methods aren't thread safe, you need to arrange them
        to be called inside the event loop thread using :func:`~from_thread.run_sync`::

            import time

            from anyio import Event, from_thread, to_thread, run

            def worker(event):
                time.sleep(1)
                from_thread.run_sync(event.set)

            async def main():
                event = Event()
                await to_thread.run_sync(worker, event)
                await event.wait()

            run(main)

从外部线程调用异步代码
-------------------------------------------------

**Calling asynchronous code from an external thread**

.. tabs::

    .. tab:: 中文

        如果您需要从一个不是由事件循环生成的工作线程中运行异步代码，您需要一个 *阻塞门户(blocking portal)* 。这个门户需要在事件循环线程中获取。

        一种方法是使用 :class:`~from_thread.start_blocking_portal` 启动一个新事件循环并获取门户（它的参数与 :func:`~run` 基本相同）::

            from anyio.from_thread import start_blocking_portal


            with start_blocking_portal(backend='trio') as portal:
                portal.call(...)

        如果您已经有一个正在运行的事件循环，并希望允许外部线程访问，您可以直接创建一个 :class:`~.BlockingPortal`::

            from anyio import run
            from anyio.from_thread import BlockingPortal


            async def main():
                async with BlockingPortal() as portal:
                    # ...将门户交给外部线程...
                    await portal.sleep_until_stopped()

            run(main)

    .. tab:: 英文

        If you need to run async code from a thread that is not a worker thread spawned by the
        event loop, you need a *blocking portal*. This needs to be obtained from within the
        event loop thread.

        One way to do this is to start a new event loop with a portal, using
        :class:`~from_thread.start_blocking_portal` (which takes mostly the same arguments as
        :func:`~run`::

            from anyio.from_thread import start_blocking_portal


            with start_blocking_portal(backend='trio') as portal:
                portal.call(...)

        If you already have an event loop running and wish to grant access to external threads,
        you can create a :class:`~.BlockingPortal` directly::

            from anyio import run
            from anyio.from_thread import BlockingPortal


            async def main():
                async with BlockingPortal() as portal:
                    # ...hand off the portal to external threads...
                    await portal.sleep_until_stopped()

            run(main)

从工作线程生成任务
----------------------------------

**Spawning tasks from worker threads**

.. tabs::

    .. tab:: 中文

        当您需要生成一个在后台运行的任务时，可以使用 :meth:`~.BlockingPortal.start_task_soon` 来实现::

            from concurrent.futures import as_completed

            from anyio import sleep
            from anyio.from_thread import start_blocking_portal


            async def long_running_task(index):
                await sleep(1)
                print(f'Task {index} running...')
                await sleep(index)
                return f'Task {index} return value'


            with start_blocking_portal() as portal:
                futures = [portal.start_task_soon(long_running_task, i) for i in range(1, 5)]
                for future in as_completed(futures):
                    print(future.result())

        通过取消返回的 :class:`~concurrent.futures.Future` 来取消这种方式生成的任务。

        阻塞门户还提供了一个与 :meth:`TaskGroup.start() <.abc.TaskGroup.start>` 类似的方法 :meth:`~.BlockingPortal.start_task`，它与其对等方法一样，通过调用 ``task_status.started()`` 等待可调用对象信号准备就绪::

            from anyio import sleep, TASK_STATUS_IGNORED
            from anyio.from_thread import start_blocking_portal


            async def service_task(*, task_status=TASK_STATUS_IGNORED):
                task_status.started('STARTED')
                await sleep(1)
                return 'DONE'


            with start_blocking_portal() as portal:
                future, start_value = portal.start_task(service_task)
                print('Task has started with value', start_value)

                return_value = future.result()
                print('Task has finished with return value', return_value)

    .. tab:: 英文

        When you need to spawn a task to be run in the background, you can do so using
        :meth:`~.BlockingPortal.start_task_soon`::

            from concurrent.futures import as_completed

            from anyio import sleep
            from anyio.from_thread import start_blocking_portal


            async def long_running_task(index):
                await sleep(1)
                print(f'Task {index} running...')
                await sleep(index)
                return f'Task {index} return value'


            with start_blocking_portal() as portal:
                futures = [portal.start_task_soon(long_running_task, i) for i in range(1, 5)]
                for future in as_completed(futures):
                    print(future.result())

        Cancelling tasks spawned this way can be done by cancelling the returned
        :class:`~concurrent.futures.Future`.

        Blocking portals also have a method similar to
        :meth:`TaskGroup.start() <.abc.TaskGroup.start>`:
        :meth:`~.BlockingPortal.start_task` which, like its counterpart, waits for the callable
        to signal readiness by calling ``task_status.started()``::

            from anyio import sleep, TASK_STATUS_IGNORED
            from anyio.from_thread import start_blocking_portal


            async def service_task(*, task_status=TASK_STATUS_IGNORED):
                task_status.started('STARTED')
                await sleep(1)
                return 'DONE'


            with start_blocking_portal() as portal:
                future, start_value = portal.start_task(service_task)
                print('Task has started with value', start_value)

                return_value = future.result()
                print('Task has finished with return value', return_value)


从工作线程使用异步上下文管理器
-------------------------------------------------------

**Using asynchronous context managers from worker threads**

.. tabs::

    .. tab:: 中文

        您可以使用 :meth:`~.BlockingPortal.wrap_async_context_manager` 将异步上下文管理器包装为同步上下文管理器::

            from anyio.from_thread import start_blocking_portal


            class AsyncContextManager:
                async def __aenter__(self):
                    print('entering')

                async def __aexit__(self, exc_type, exc_val, exc_tb):
                    print('exiting with', exc_type)


            async_cm = AsyncContextManager()
            with start_blocking_portal() as portal, portal.wrap_async_context_manager(async_cm):
                print('inside the context manager block')

        .. note:: 不能在事件循环线程中的同步回调中使用包装的异步上下文管理器。

    .. tab:: 英文

        You can use :meth:`~.BlockingPortal.wrap_async_context_manager` to wrap an asynchronous
        context managers as a synchronous one::

            from anyio.from_thread import start_blocking_portal


            class AsyncContextManager:
                async def __aenter__(self):
                    print('entering')

                async def __aexit__(self, exc_type, exc_val, exc_tb):
                    print('exiting with', exc_type)


            async_cm = AsyncContextManager()
            with start_blocking_portal() as portal, portal.wrap_async_context_manager(async_cm):
                print('inside the context manager block')

        .. note:: You cannot use wrapped async context managers in synchronous callbacks inside
        the event loop thread.

上下文传播
-------------------

**Context propagation**

.. tabs::

    .. tab:: 中文

        在工作线程中运行函数时，当前上下文会被复制到工作线程。因此，任务中可用的任何上下文变量也将在线程中运行的代码中可用。与上下文变量一样，对它们所做的任何更改将不会传播回调用的异步任务。

        从工作线程调用异步代码时，上下文再次被复制到事件循环线程中调用目标函数的任务。

    .. tab:: 英文

        When running functions in worker threads, the current context is copied to the worker
        thread. Therefore any context variables available on the task will also be available to
        the code running on the thread. As always with context variables, any changes made to
        them will not propagate back to the calling asynchronous task.

        When calling asynchronous code from worker threads, context is again copied to the task
        that calls the target function in the event loop thread.

调整默认最大工作线程数
-------------------------------------------------

**Adjusting the default maximum worker thread count**

.. tabs::

    .. tab:: 中文

        默认的 AnyIO 工作线程限制器值为 **40**，这意味着任何没有显式 ``limiter`` 参数的 :func:`.to_thread.run_sync` 调用将最多会启动 40 个线程。您可以像这样调整此限制：

            from anyio import to_thread

            async def foo():
                # 设置最大工作线程数为 60
                to_thread.current_default_thread_limiter().total_tokens = 60

        .. note:: AnyIO 的默认线程池限制器不会影响 :mod:`asyncio` 中的默认线程池执行器。

    .. tab:: 英文

        The default AnyIO worker thread limiter has a value of **40**, meaning that any calls
        to :func:`.to_thread.run_sync` without an explicit ``limiter`` argument will cause a
        maximum of 40 threads to be spawned. You can adjust this limit like this::

            from anyio import to_thread

            async def foo():
                # Set the maximum number of worker threads to 60
                to_thread.current_default_thread_limiter().total_tokens = 60

        .. note:: AnyIO's default thread pool limiter does not affect the default thread pool
            executor on :mod:`asyncio`.

对工作线程中的取消做出反应
------------------------------------------

**Reacting to cancellation in worker threads**

.. tabs::

    .. tab:: 中文

        虽然 Python 没有机制来取消在线程中运行的代码，但 AnyIO 提供了一种机制，允许用户代码自愿检查宿主任务的作用域是否已被取消，如果已取消，则引发取消异常。这可以通过简单地调用 :func:`from_thread.check_cancelled` 来实现：

            from anyio import to_thread, from_thread

            def sync_function():
                while True:
                    from_thread.check_cancelled()
                    print("还没有取消")
                    sleep(1)

            async def foo():
                with move_on_after(3):
                    await to_thread.run_sync(sync_function)

    .. tab:: 英文

        While there is no mechanism in Python to cancel code running in a thread, AnyIO provides a
        mechanism that allows user code to voluntarily check if the host task's scope has been cancelled,
        and if it has, raise a cancellation exception. This can be done by simply calling
        :func:`from_thread.check_cancelled`::

            from anyio import to_thread, from_thread

            def sync_function():
                while True:
                    from_thread.check_cancelled()
                    print("Not cancelled yet")
                    sleep(1)

            async def foo():
                with move_on_after(3):
                    await to_thread.run_sync(sync_function)


按需共享阻塞门户
-----------------------------------

**Sharing a blocking portal on demand**

.. tabs::

    .. tab:: 中文

        如果您正在构建一个需要按需启动阻塞门户的同步 API，您可能需要比每次调用都启动一个阻塞门户更高效的解决方案。为此，您可以使用 :class:`BlockingPortalProvider`::

            from anyio.from_thread import BlockingPortalProvider

            class MyAPI:
                def __init__(self, async_obj) -> None:
                    self._async_obj = async_obj
                    self._portal_provider = BlockingPortalProvider()

                def do_stuff(self) -> None:
                    with self._portal_provider as portal:
                        portal.call(async_obj.do_async_stuff)

        现在，无论有多少线程同时调用 ``MyAPI`` 实例上的 ``do_stuff()`` 方法，都会使用相同的阻塞门户来处理内部的异步调用。很容易看出，这比每个调用都启动自己的阻塞门户要高效得多。

    .. tab:: 英文

        If you're building a synchronous API that needs to start a blocking portal on demand,
        you might need a more efficient solution than just starting a blocking portal for each
        call. To that end, you can use :class:`BlockingPortalProvider`::

            from anyio.from_thread import BlockingPortalProvider

            class MyAPI:
                def __init__(self, async_obj) -> None:
                    self._async_obj = async_obj
                    self._portal_provider = BlockingPortalProvider()

                def do_stuff(self) -> None:
                    with self._portal_provider as portal:
                        portal.call(async_obj.do_async_stuff)

        Now, no matter how many threads call the ``do_stuff()`` method on a ``MyAPI`` instance
        at the same time, the same blocking portal will be used to handle the async calls
        inside. It's easy to see that this is much more efficient than having each call spawn
        its own blocking portal.
