创建和管理任务
===========================

**Creating and managing tasks**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        *任务* 是一个执行单元，使您能够同时处理多个需要等待的操作。尽管可以创建任意数量的任务，但异步事件循环在任一时间只能运行其中的一个。当任务遇到一个需要任务暂停等待的 ``await`` 语句时，事件循环便可以自由地处理其他任务。当第一个任务等待的事件完成后，事件循环将在获得机会时恢复该任务的执行。

        AnyIO 中的任务处理大致遵循 Trio_ 模型。可以使用 *任务组* 来创建（*生成*）任务。任务组是一个异步上下文管理器，确保在退出上下文块后，其所有子任务都以某种方式完成。如果子任务或上下文块中的代码引发了异常，则所有子任务都会被取消。否则，上下文管理器会等待所有子任务退出后再继续执行。

        示例代码如下::

            from anyio import sleep, create_task_group, run


            async def sometask(num: int) -> None:
                print('Task', num, 'running')
                await sleep(1)
                print('Task', num, 'finished')


            async def main() -> None:
                async with create_task_group() as tg:
                    for num in range(5):
                        tg.start_soon(sometask, num)

                print('All tasks finished!')

            run(main)

    .. tab:: 英文

        A *task* is a unit of execution that lets you do many things concurrently that need waiting on. This works so that while you can have any number of tasks, the asynchronous event loop can only run one of them at a time. When the task encounters an ``await`` statement that requires the task to sleep until something happens, the event loop is then free to work on another task. When the thing the first task was waiting is complete, the event loop will resume the execution of that task on the first opportunity it gets.

        Task handling in AnyIO loosely follows the Trio_ model. Tasks can be created (*spawned*) using *task groups*. A task group is an asynchronous context manager that makes sure that all its child tasks are finished one way or another after the context block is exited. If a child task, or the code in the enclosed context block raises an exception, all child tasks are cancelled. Otherwise the context manager just waits until all child tasks have exited before proceeding.

        Here's a demonstration::

            from anyio import sleep, create_task_group, run


            async def sometask(num: int) -> None:
                print('Task', num, 'running')
                await sleep(1)
                print('Task', num, 'finished')


            async def main() -> None:
                async with create_task_group() as tg:
                    for num in range(5):
                        tg.start_soon(sometask, num)

                print('All tasks finished!')

            run(main)

.. _Trio: https://trio.readthedocs.io/en/latest/reference-core.html#tasks-let-you-do-multiple-things-at-once

启动和初始化任务
-------------------------------

**Starting and initializing tasks**

.. tabs::

    .. tab:: 中文
        
        有时能够等待任务成功初始化自身是非常有用的。例如，在启动网络服务时，您可以让任务启动监听器，然后向调用方发出初始化完成的信号。这样，调用方可以启动依赖于该服务已启动和运行的其他任务。此外，如果套接字绑定失败或在初始化期间出现其他问题，异常会传播给调用方，从而可以捕获并处理异常。

        这可以通过 :meth:`TaskGroup.start() <.abc.TaskGroup.start>` 来实现::

            from anyio import (
                TASK_STATUS_IGNORED,
                create_task_group,
                connect_tcp,
                create_tcp_listener,
                run,
            )
            from anyio.abc import TaskStatus


            async def handler(stream):
                ...


            async def start_some_service(
                port: int, *, task_status: TaskStatus[None] = TASK_STATUS_IGNORED
            ):
                async with await create_tcp_listener(
                    local_host="127.0.0.1", local_port=port
                ) as listener:
                    task_status.started()
                    await listener.serve(handler)


            async def main():
                async with create_task_group() as tg:
                    await tg.start(start_some_service, 5000)
                    async with await connect_tcp("127.0.0.1", 5000) as stream:
                        ...


            run(main)

        目标协程函数 **必须** 调用 ``task_status.started()``，因为调用 :meth:`TaskGroup.start() <.abc.TaskGroup.start>` 的任务将会阻塞，直到此方法被调用。如果生成的任务未调用它，那么 :meth:`TaskGroup.start() <.abc.TaskGroup.start>` 调用将引发 ``RuntimeError``。

        .. 注意:: 与 :meth:`~.abc.TaskGroup.start_soon` 不同，:meth:`~.abc.TaskGroup.start` 需要使用 ``await``。

    .. tab:: 英文

        Sometimes it is very useful to be able to wait until a task has successfully initialized itself. For example, when starting network services, you can have your task start the listener and then signal the caller that initialization is done. That way, the caller can now start another task that depends on that service being up and running. Also, if the socket bind fails or something else goes wrong during initialization, the exception will be propagated to the caller which can then catch and handle it.

        This can be done with :meth:`TaskGroup.start() <.abc.TaskGroup.start>`::

            from anyio import (
                TASK_STATUS_IGNORED,
                create_task_group,
                connect_tcp,
                create_tcp_listener,
                run,
            )
            from anyio.abc import TaskStatus


            async def handler(stream):
                ...


            async def start_some_service(
                port: int, *, task_status: TaskStatus[None] = TASK_STATUS_IGNORED
            ):
                async with await create_tcp_listener(
                    local_host="127.0.0.1", local_port=port
                ) as listener:
                    task_status.started()
                    await listener.serve(handler)


            async def main():
                async with create_task_group() as tg:
                    await tg.start(start_some_service, 5000)
                    async with await connect_tcp("127.0.0.1", 5000) as stream:
                        ...


            run(main)

        The target coroutine function **must** call ``task_status.started()`` because the task that is calling with :meth:`TaskGroup.start() <.abc.TaskGroup.start>` will be blocked until then. If the spawned task never calls it, then the :meth:`TaskGroup.start() <.abc.TaskGroup.start>` call will raise a ``RuntimeError``.

        .. note:: Unlike :meth:`~.abc.TaskGroup.start_soon`, :meth:`~.abc.TaskGroup.start` needs an ``await``.

处理任务组中的多个错误
----------------------------------------

**Handling multiple errors in a task group**

.. tabs::

    .. tab:: 中文
        
        在任务组中，多个任务可能会引发异常。当任务响应取消操作时，可能进入异常处理块或 ``finally:`` 块，并在此期间引发异常。这就引出了一个问题：哪个异常会从任务组上下文管理器中传播出来？答案是“两个”。实际上，这意味着会引发一个特殊的异常 :exc:`ExceptionGroup` （或 :exc:`BaseExceptionGroup` ），其中包含了两个异常对象。

        要捕获可能嵌套在组中的此类异常，需要采取特殊措施。在 Python 3.11 及更高版本中，可以使用 ``except*`` 语法来捕获多个异常::

            from anyio import create_task_group

            try:
                async with create_task_group() as tg:
                    tg.start_soon(some_task)
                    tg.start_soon(another_task)
            except* ValueError as excgroup:
                for exc in excgroup.exceptions:
                    ...  # 处理每个 ValueError
            except* KeyError as excgroup:
                for exc in excgroup.exceptions:
                    ...  # 处理每个 KeyError
        
        如果需要兼容旧版本的 Python，可以使用 exceptiongroup_ 包中的 ``catch()`` 函数::

            from anyio import create_task_group
            from exceptiongroup import catch

            def handle_valueerror(excgroup: ExceptionGroup) -> None:
                for exc in excgroup.exceptions:
                    ...  # 处理每个 ValueError

            def handle_keyerror(excgroup: ExceptionGroup) -> None:
                for exc in excgroup.exceptions:
                    ...  # 处理每个 KeyError

            with catch({
                ValueError: handle_valueerror,
                KeyError: handle_keyerror
            }):
                async with create_task_group() as tg:
                    tg.start_soon(some_task)
                    tg.start_soon(another_task)
        
        如果需要在处理器中设置局部变量，可以将其声明为 ``nonlocal``::

            def handle_valueerror(exc):
                nonlocal somevariable
                somevariable = 'whatever'

    .. tab:: 英文

        It is possible for more than one task to raise an exception in a task group. This can happen when a task reacts to cancellation by entering either an exception handler block or a ``finally:`` block and raises an exception there. This raises the question: which exception is propagated from the task group context manager? The answer is "both". In practice this means that a special exception, :exc:`ExceptionGroup` (or :exc:`BaseExceptionGroup`) is raised which contains both exception objects.

        To catch such exceptions potentially nested in groups, special measures are required. On Python 3.11 and later, you can use the ``except*`` syntax to catch multiple exceptions::

            from anyio import create_task_group

            try:
                async with create_task_group() as tg:
                    tg.start_soon(some_task)
                    tg.start_soon(another_task)
            except* ValueError as excgroup:
                for exc in excgroup.exceptions:
                    ...  # handle each ValueError
            except* KeyError as excgroup:
                for exc in excgroup.exceptions:
                    ...  # handle each KeyError

        If compatibility with older Python versions is required, you can use the ``catch()`` function from the exceptiongroup_ package::

            from anyio import create_task_group
            from exceptiongroup import catch

            def handle_valueerror(excgroup: ExceptionGroup) -> None:
                for exc in excgroup.exceptions:
                    ...  # handle each ValueError

            def handle_keyerror(excgroup: ExceptionGroup) -> None:
                for exc in excgroup.exceptions:
                    ...  # handle each KeyError

            with catch({
                ValueError: handle_valueerror,
                KeyError: handle_keyerror
            }):
                async with create_task_group() as tg:
                    tg.start_soon(some_task)
                    tg.start_soon(another_task)

        If you need to set local variables in the handlers, declare them as ``nonlocal``::

            def handle_valueerror(exc):
                nonlocal somevariable
                somevariable = 'whatever'

.. _exceptiongroup: https://pypi.org/project/exceptiongroup/

上下文传播
-------------------

**Context propagation**

.. tabs::

    .. tab:: 中文
        
        每当生成一个新任务时，`context`_ 将被复制到该新任务中。需要特别注意*哪个*上下文会被复制到新生成的任务中。被复制的不是任务组的宿主任务的上下文，而是调用 :meth:`TaskGroup.start() <.abc.TaskGroup.start>` 或 :meth:`TaskGroup.start_soon() <.abc.TaskGroup.start_soon>` 的任务的上下文。

    .. tab:: 英文

        Whenever a new task is spawned, `context`_ will be copied to the new task. It is
        important to note *which* context will be copied to the newly spawned task. It is not
        the context of the task group's host task that will be copied, but the context of the
        task that calls :meth:`TaskGroup.start() <.abc.TaskGroup.start>` or
        :meth:`TaskGroup.start_soon() <.abc.TaskGroup.start_soon>`.

.. _context: https://docs.python.org/3/library/contextvars.html

与 asyncio.TaskGroup 的区别
----------------------------------

**Differences with asyncio.TaskGroup**

.. tabs::

    .. tab:: 中文
        
        :class:`asyncio.TaskGroup` 类是在 Python 3.11 中新增的，其设计与 AnyIO 的 :class:`~.abc.TaskGroup` 类非常相似。然而，asyncio 的对应类在语义上有一些重要的区别：

        * 任务组本身是直接实例化的，而不是通过工厂函数创建
        * 任务仅通过 :meth:`~asyncio.TaskGroup.create_task` 生成；没有 ``start()`` 或 ``start_soon()`` 方法
        * :meth:`~asyncio.TaskGroup.create_task` 方法返回一个任务对象，可以进行 `await` 操作（或取消）
        * 通过 :meth:`~asyncio.TaskGroup.create_task` 生成的任务只能单独取消（任务组中没有 ``cancel()`` 方法或类似的方法）
        * 当通过 :meth:`~asyncio.TaskGroup.create_task` 生成的任务在其协程开始运行之前被取消时，它将无法处理取消异常
        * :class:`asyncio.TaskGroup` 不允许在某个任务发生异常并触发任务组关闭后再启动新任务     

    .. tab:: 英文

        The :class:`asyncio.TaskGroup` class, added in Python 3.11, is very similar in design to
        the AnyIO :class:`~.abc.TaskGroup` class. The asyncio counterpart has some important
        differences in its semantics, however:

        * The task group itself is instantiated directly, rather than using a factory function
        * Tasks are spawned solely through :meth:`~asyncio.TaskGroup.create_task`; there is no
        ``start()`` or ``start_soon()`` method
        * The :meth:`~asyncio.TaskGroup.create_task` method returns a task object which can be
        awaited on (or cancelled)
        * Tasks spawned via :meth:`~asyncio.TaskGroup.create_task` can only be cancelled
        individually (there is no ``cancel()`` method or similar in the task group)
        * When a task spawned via :meth:`~asyncio.TaskGroup.create_task` is cancelled before its
        coroutine has started running, it will not get a chance to handle the cancellation
        exception
        * :class:`asyncio.TaskGroup` does not allow starting new tasks after an exception in
        one of the tasks has triggered a shutdown of the task group
