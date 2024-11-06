取消和超时
=========================

Cancellation and timeouts

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        取消任务的能力是异步编程模型的主要优势。而线程则无法被强制终止，关闭它们需要线程内代码的完全配合。

        在 AnyIO 中，任务的取消遵循 Trio_ 框架建立的模型。这意味着任务的取消是通过所谓的 *取消作用域* 实现的。取消作用域被用作上下文管理器，并且可以嵌套。取消一个取消作用域会取消其内部所有嵌套的取消作用域。如果任务正在等待某个操作，则会立即被取消。如果任务刚刚开始运行，它将继续运行，直到第一次尝试执行需要等待的操作（如 :func:`~sleep` ）时才会被取消。

        任务组包含其自己的取消作用域。通过取消该作用域可以取消整个任务组。
        

    .. tab:: 英文

        The ability to cancel tasks is the foremost advantage of the asynchronous programming
        model. Threads, on the other hand, cannot be forcibly killed and shutting them down will
        require perfect cooperation from the code running in them.

        Cancellation in AnyIO follows the model established by the Trio_ framework. This means
        that cancellation of tasks is done via so called *cancel scopes*. Cancel scopes are used
        as context managers and can be nested. Cancelling a cancel scope cancels all cancel
        scopes nested within it. If a task is waiting on something, it is cancelled immediately.
        If the task is just starting, it will run until it first tries to run an operation
        requiring waiting, such as :func:`~sleep`.

        A task group contains its own cancel scope. The entire task group can be cancelled by
        cancelling this scope.

.. _Trio: https://trio.readthedocs.io/en/latest/reference-core.html
   #cancellation-and-timeouts

超时
--------

**Timeouts**

.. tabs::

    .. tab:: 中文
        
        网络操作通常可能需要很长时间，因此你通常希望设置某种超时机制，以确保你的应用不会永远卡住。主要有两种方法来实现这一点：:func:`~move_on_after` 和 :func:`~fail_after`。这两者都作为同步上下文管理器使用。它们之间的区别在于，前者在超时后简单地提前退出上下文块，而后者则会引发 :exc:`TimeoutError`。

        这两种方法都会创建一个新的取消作用域，你可以通过访问 :attr:`~.CancelScope.deadline` 属性来检查截止时间。然而，请注意，外部取消作用域的截止时间可能早于当前取消作用域的截止时间。要检查实际的截止时间，你可以使用 :func:`~current_effective_deadline` 函数。

        下面是你通常如何使用超时的示例::
        
            from anyio import create_task_group, move_on_after, sleep, run


            async def main():
                async with create_task_group() as tg:
                    with move_on_after(1) as scope:
                        print('Starting sleep')
                        await sleep(2)
                        print('This should never be printed')

                    # 如果超时达到，cancelled_caught 属性将为 True
                    print('Exited cancel scope, cancelled =', scope.cancelled_caught)

            run(main)

        .. note:: 不建议直接从 :func:`~fail_after` 取消作用域，因为如果退出作用域的延迟足够长，导致超过截止时间，这可能会错误地引发 :exc:`TimeoutError`。    

    .. tab:: 英文

        Networked operations can often take a long time, and you usually want to set up some
        kind of a timeout to ensure that your application doesn't stall forever. There are two
        principal ways to do this: :func:`~move_on_after` and :func:`~fail_after`. Both are used
        as synchronous context managers. The difference between these two is that the former
        simply exits the context block prematurely on a timeout, while the other raises a
        :exc:`TimeoutError`.

        Both methods create a new cancel scope, and you can check the deadline by accessing the
        :attr:`~.CancelScope.deadline` attribute. Note, however, that an outer cancel scope
        may have an earlier deadline than your current cancel scope. To check the actual
        deadline, you can use the :func:`~current_effective_deadline` function.

        Here's how you typically use timeouts::

            from anyio import create_task_group, move_on_after, sleep, run


            async def main():
                async with create_task_group() as tg:
                    with move_on_after(1) as scope:
                        print('Starting sleep')
                        await sleep(2)
                        print('This should never be printed')

                    # The cancelled_caught property will be True if timeout was reached
                    print('Exited cancel scope, cancelled =', scope.cancelled_caught)

            run(main)

        .. note:: It's recommended not to directly cancel a scope from :func:`~fail_after`, as
            that may currently result in :exc:`TimeoutError` being erroneously raised if exiting
            the scope is delayed long enough for the deadline to be exceeded.

屏蔽
---------

**Shielding**

.. tabs::

    .. tab:: 中文
        
        在某些情况下，你可能希望暂时保护任务免于被取消。最重要的应用场景是对异步资源执行关闭操作。

        为此，可以使用 ``shield=True`` 参数打开一个新的取消作用域::

            from anyio import CancelScope, create_task_group, sleep, run


            async def external_task():
                print('Started sleeping in the external task')
                await sleep(1)
                print('This line should never be seen')


            async def main():
                async with create_task_group() as tg:
                    with CancelScope(shield=True) as scope:
                        tg.start_soon(external_task)
                        tg.cancel_scope.cancel()
                        print('Started sleeping in the host task')
                        await sleep(1)
                        print('Finished sleeping in the host task')

            run(main)

        被保护的代码块将免于取消，除非该保护代码块本身正在被取消。保护取消作用域通常最好与 :func:`~move_on_after` 或 :func:`~fail_after` 结合使用，这两者也接受 ``shield=True`` 参数。 

    .. tab:: 英文

        There are cases where you want to shield your task from cancellation, at least
        temporarily. The most important such use case is performing shutdown procedures on
        asynchronous resources.

        To accomplish this, open a new cancel scope with the ``shield=True`` argument::

            from anyio import CancelScope, create_task_group, sleep, run


            async def external_task():
                print('Started sleeping in the external task')
                await sleep(1)
                print('This line should never be seen')


            async def main():
                async with create_task_group() as tg:
                    with CancelScope(shield=True) as scope:
                        tg.start_soon(external_task)
                        tg.cancel_scope.cancel()
                        print('Started sleeping in the host task')
                        await sleep(1)
                        print('Finished sleeping in the host task')

            run(main)

        The shielded block will be exempt from cancellation except when the shielded block
        itself is being cancelled. Shielding a cancel scope is often best combined with
        :func:`~move_on_after` or :func:`~fail_after`, both of which also accept
        ``shield=True``.

完成
------------

**Finalization**

.. tabs::

    .. tab:: 中文
        
        有时你可能希望在操作失败时执行清理操作::

            async def do_something():
                try:
                    await run_async_stuff()
                except BaseException:
                    # (执行清理操作)
                    raise

        在某些特定情况下，你可能只想捕获取消异常。这比较棘手，因为每个异步框架都有自己的异常类，而 AnyIO 无法控制任务在取消时抛出的异常。为了解决这个问题，AnyIO 提供了一种方法来获取当前运行的异步框架特定的异常类，使用 :func:`~get_cancelled_exc_class`::

            from anyio import get_cancelled_exc_class


            async def do_something():
                try:
                    await run_async_stuff()
                except get_cancelled_exc_class():
                    # (执行清理操作)
                    raise

        .. warning:: 如果捕获了取消异常，务必重新抛出它。未能重新抛出可能会导致应用程序出现未定义的行为。

        如果在清理过程中需要使用 ``await``，你需要将其包含在一个受保护的取消作用域中，否则操作会立即被取消，因为它已经处于一个被取消的作用域中::

            async def do_something():
                try:
                    await run_async_stuff()
                except get_cancelled_exc_class():
                    with CancelScope(shield=True):
                        await some_cleanup_function()

                    raise

    .. tab:: 英文

        Sometimes you may want to perform cleanup operations in response to the failure of the
        operation::

            async def do_something():
                try:
                    await run_async_stuff()
                except BaseException:
                    # (perform cleanup)
                    raise

        In some specific cases, you might only want to catch the cancellation exception. This is
        tricky because each async framework has its own exception class for that and AnyIO
        cannot control which exception is raised in the task when it's cancelled. To work around
        that, AnyIO provides a way to retrieve the exception class specific to the currently
        running async framework, using:func:`~get_cancelled_exc_class`::

            from anyio import get_cancelled_exc_class


            async def do_something():
                try:
                    await run_async_stuff()
                except get_cancelled_exc_class():
                    # (perform cleanup)
                    raise

        .. warning:: Always reraise the cancellation exception if you catch it. Failing to do so
            may cause undefined behavior in your application.

        If you need to use ``await`` during finalization, you need to enclose it in a shielded
        cancel scope, or the operation will be cancelled immediately since it's in an already
        cancelled scope::

            async def do_something():
                try:
                    await run_async_stuff()
                except get_cancelled_exc_class():
                    with CancelScope(shield=True):
                        await some_cleanup_function()

                    raise

避免取消范围堆栈损坏
--------------------------------------

**Avoiding cancel scope stack corruption**

.. tabs::

    .. tab:: 中文
        
        在使用取消作用域时，重要的是它们在每个任务内应按照 LIFO（后进先出）顺序被进入和退出。通常这不是问题，因为取消作用域通常作为上下文管理器使用。然而，在某些情况下，取消作用域堆栈可能仍然会发生损坏：

        * 手动调用 ``CancelScope.__enter__()`` 和 ``CancelScope.__exit__()``，通常是在另一个上下文管理器类中，以错误的顺序调用
        * 使用 ``[Async]ExitStack`` 的取消作用域，方式是通过嵌套上下文管理器无法实现的方式
        * 使用低级协程协议在不同的取消作用域中执行协程函数的部分内容
        * 在异步生成器中使用 ``yield``，同时该生成器被包裹在取消作用域中

        记住，任务组包含它们自己的取消作用域，因此相同的风险情况也适用于它们。

        例如，以下代码是非常可疑的::

            # 错误！
            async def some_generator():
                async with create_task_group() as tg:
                    tg.start_soon(foo)
                    yield

        这段代码的问题在于它违反了结构性并发：如果生成的任务引发异常会发生什么？宿主任务将因此被取消，但在这种情况下，宿主任务可能早已结束。即使没有结束，生成器中的任何封闭 ``try...except`` 也不会被触发。不幸的是，AnyIO 目前无法自动检测这种情况，因此实际上你可能会因为运行类似代码而在应用程序中遇到一些奇怪的行为。

        然而，根据它们的使用方式，这种模式 *通常* 是安全的，只要你确保同一个宿主任务在整个封闭代码块中一直在运行::

            # 在大多数情况下是安全的！
            @async_context_manager
            async def some_context_manager():
                async with create_task_group() as tg:
                    tg.start_soon(foo)
                    yield

        在 AnyIO 3.6 之前，这种用法模式在 pytest 的异步生成器固定装置中也是无效的。然而，从 3.6 开始，每个异步生成器固定装置都会在同一个任务中从头到尾运行，这使得任务组或取消作用域可以安全地跨越 ``yield``。

        当你手动实现异步上下文管理器协议，并且你的异步上下文管理器需要使用其他上下文管理器时，你可能会发现有必要直接调用它们的 ``__aenter__()`` 和 ``__aexit__()``。在这种情况下，确保它们的 ``__aexit__()`` 方法按 ``__aenter__()`` 调用的逆序被调用是至关重要的。为此，你可能会发现 :class:`~contextlib.AsyncExitStack` 类非常有用::

            from contextlib import AsyncExitStack

            from anyio import create_task_group


            class MyAsyncContextManager:
                async def __aenter__(self):
                    self._exitstack = AsyncExitStack()
                    await self._exitstack.__aenter__()
                    self._task_group = await self._exitstack.enter_async_context(
                        create_task_group()
                    )

                async def __aexit__(self, exc_type, exc_val, exc_tb):
                    return await self._exitstack.__aexit__(exc_type, exc_val, exc_tb)

    .. tab:: 英文

        When using cancel scopes, it is important that they are entered and exited in LIFO (last
        in, first out) order within each task. This is usually not an issue since cancel scopes
        are normally used as context managers. However, in certain situations, cancel scope
        stack corruption might still occur:

        * Manually calling ``CancelScope.__enter__()`` and ``CancelScope.__exit__()``, usually
        from another context manager class, in the wrong order
        * Using cancel scopes with ``[Async]ExitStack`` in a manner that couldn't be achieved by
        nesting them as context managers
        * Using the low level coroutine protocol to execute parts of the coroutine function in
        different cancel scopes
        * Yielding in an async generator while enclosed in a cancel scope

        Remember that task groups contain their own cancel scopes so the same list of risky
        situations applies to them too.

        As an example, the following code is highly dubious::

            # Bad!
            async def some_generator():
                async with create_task_group() as tg:
                    tg.start_soon(foo)
                    yield

        The problem with this code is that it violates structural concurrency: what happens if
        the spawned task raises an exception? The host task would be cancelled as a result, but
        the host task might be long gone by the time that happens. Even if it weren't, any
        enclosing ``try...except`` in the generator would not be triggered. Unfortunately there
        is currently no way to automatically detect this condition in AnyIO, so in practice you
        may simply experience some weird behavior in your application as a consequence of
        running code like above.

        Depending on how they are used, this pattern is, however, *usually* safe to use in
        asynchronous context managers, so long as you make sure that the same host task keeps
        running throughout the entire enclosed code block::

            # Okay in most cases!
            @async_context_manager
            async def some_context_manager():
                async with create_task_group() as tg:
                    tg.start_soon(foo)
                    yield

        Prior to AnyIO 3.6, this usage pattern was also invalid in pytest's asynchronous
        generator fixtures. Starting from 3.6, however, each async generator fixture is run from
        start to end in the same task, making it possible to have task groups or cancel scopes
        safely straddle the ``yield``.

        When you're implementing the async context manager protocol manually and your async
        context manager needs to use other context managers, you may find it necessary to call
        their ``__aenter__()`` and ``__aexit__()`` directly. In such cases, it is absolutely
        vital to ensure that their ``__aexit__()`` methods are called in the exact reverse order
        of the ``__aenter__()`` calls. To this end, you may find the
        :class:`~contextlib.AsyncExitStack` class very useful::

            from contextlib import AsyncExitStack

            from anyio import create_task_group


            class MyAsyncContextManager:
                async def __aenter__(self):
                    self._exitstack = AsyncExitStack()
                    await self._exitstack.__aenter__()
                    self._task_group = await self._exitstack.enter_async_context(
                        create_task_group()
                    )

                async def __aexit__(self, exc_type, exc_val, exc_tb):
                    return await self._exitstack.__aexit__(exc_type, exc_val, exc_tb)
