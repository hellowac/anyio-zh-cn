从 AnyIO 3 迁移到 AnyIO 4
=================================

.. py:currentmodule:: anyio

**Migrating from AnyIO 3 to AnyIO 4**

非标准异常组类已被删除
--------------------------------------------------

**The non-standard exception group class was removed**

.. tabs::

    .. tab:: 中文

        AnyIO 3 之前有一个自定义的 ``ExceptionGroup`` 类，它在 :pep:`654` 异常组类出现之前就已存在。这个类现在已经被移除，改为使用内置的 :exc:`BaseExceptionGroup` 和 :exc:`ExceptionGroup` 类。如果您的代码曾经抛出旧的 ``ExceptionGroup`` 异常或捕获它，您需要切换到这些标准类。否则，您可以忽略这一部分。

        如果您正在针对 Python 3.11 之前的版本，您需要使用 `exceptiongroup_` 回溯，并从 ``exceptiongroup`` 导入这些类之一。 :exc:`BaseExceptionGroup` 和 :exc:`ExceptionGroup` 之间的唯一区别是，后者只能包含从 :exc:`Exception` 派生的异常，因此也可以通过 ``except Exception:`` 来捕获。

    .. tab:: 英文

        AnyIO 3 had its own ``ExceptionGroup`` class which predated the :pep:`654` exception
        group classes. This class has now been removed in favor of the built-in
        :exc:`BaseExceptionGroup` and :exc:`ExceptionGroup` classes. If your code was either
        raising the old ``ExceptionGroup`` exception or catching it, you need to make the switch
        to these standard classes. Otherwise you can ignore this part.

        If you're targeting Python releases older than 3.11, you need to use the exceptiongroup_
        backport and import one of those classes from ``exceptiongroup``. The only difference
        between :exc:`BaseExceptionGroup` and :exc:`ExceptionGroup` is that the latter can
        only contain exceptions derived from :exc:`Exception`, and likewise can be caught with
        ``except Exception:``.

任务组现在将单个异常包装在组中
------------------------------------------------

**Task groups now wrap single exceptions in groups**

.. tabs::

    .. tab:: 中文

        AnyIO 4 中最显著的不兼容变更是，任务组现在总是会在主任务或任何子任务抛出异常时（除了取消异常）抛出异常组。以前，只有当任务组需要抛出多个异常时，才会抛出异常组。实际上，这意味着如果您的代码以前期望捕获从任务组中抛出的特定类型异常，您现在需要切换到 ``except*`` 语法（如果您恰好使用的是 Python 3.11 或更高版本），或者使用 `exceptiongroup_` 回溯中的 ``catch()`` 上下文管理器。

        因此，如果您的代码像这样::

            try:
                await function_using_a_taskgroup()
            except ValueError as exc:
                ...

        那么 Python 3.11+ 版本的等效代码几乎是一样的::

            try:
                await function_using_a_taskgroup()
            except* ValueError as excgrp:
                # 注意：excgrp 现在是一个 ExceptionGroup！
                ...

        如果您需要保持与较旧 Python 版本的兼容性，您需要使用回溯版本::

            from exceptiongroup import ExceptionGroup, catch

            def handle_value_errors(excgrp: ExceptionGroup) -> None:
                ...

            with catch({ValueError: handle_value_errors}):
                await function_using_a_taskgroup()

        这个差异通常也会出现在测试套件中。例如，如果您以前在基于 pytest 的测试套件中有如下代码::

            with pytest.raises(ValueError):
                await function_using_a_taskgroup()

        现在需要改成::

            from exceptiongroup import ExceptionGroup

            with pytest.raises(ExceptionGroup) as exc:
                await function_using_a_taskgroup()

            assert len(exc.value.exceptions) == 1
            assert isinstance(exc.value.exceptions[0], ValueError)

        如果您需要保持与 AnyIO 3 和 4 的兼容性，可以使用以下兼容性代码，通过解包来“折叠”单一异常组::

            import sys
            from contextlib import contextmanager
            from typing import Generator

            has_exceptiongroups = True
            if sys.version_info < (3, 11):
                try:
                    from exceptiongroup import BaseExceptionGroup
                except ImportError:
                    has_exceptiongroups = False


            @contextmanager
            def collapse_excgroups() -> Generator[None, None, None]:
                try:
                    yield
                except BaseException as exc:
                    if has_exceptiongroups:
                        while isinstance(exc, BaseExceptionGroup) and len(exc.exceptions) == 1:
                            exc = exc.exceptions[0]

                    raise exc

    .. tab:: 英文

        The most prominent backwards incompatible change in AnyIO 4 was that task groups now
        always raise exception groups when either the host task or any child tasks raise an
        exception (other than a cancellation exception). Previously, an exception group was only
        raised when more than one exception needed to be raised from the task group. The
        practical consequence is that if your code previously expected to catch a specific kind
        of exception falling out of a task group, you now need to either switch to the
        ``except*`` syntax (if you're fortunate enough to work solely with Python 3.11 or
        later), or use the ``catch()`` context manager from the exceptiongroup_ backport.

        So, if you had code like this::

            try:
                await function_using_a_taskgroup()
            except ValueError as exc:
                ...

        The Python 3.11+ equivalent would look almost the same::

            try:
                await function_using_a_taskgroup()
            except* ValueError as excgrp:
                # Note: excgrp is an ExceptionGroup now!
                ...

        If you need to stay compatible with older Python releases, you need to use the
        backport::

            from exceptiongroup import ExceptionGroup, catch

            def handle_value_errors(excgrp: ExceptionGroup) -> None:
                ...

            with catch({ValueError: handle_value_errors}):
                await function_using_a_taskgroup()

        This difference often comes up in test suites too. For example, if you had this before
        in a pytest-based test suite::

            with pytest.raises(ValueError):
                await function_using_a_taskgroup()

        You now need to change it to::

            from exceptiongroup import ExceptionGroup

            with pytest.raises(ExceptionGroup) as exc:
                await function_using_a_taskgroup()

            assert len(exc.value.exceptions) == 1
            assert isinstance(exc.value.exceptions[0], ValueError)

        If you need to stay compatible with both AnyIO 3 and 4, you can use the following
        compatibility code to "collapse" single-exception groups by unwrapping them::

            import sys
            from contextlib import contextmanager
            from typing import Generator

            has_exceptiongroups = True
            if sys.version_info < (3, 11):
                try:
                    from exceptiongroup import BaseExceptionGroup
                except ImportError:
                    has_exceptiongroups = False


            @contextmanager
            def collapse_excgroups() -> Generator[None, None, None]:
                try:
                    yield
                except BaseException as exc:
                    if has_exceptiongroups:
                        while isinstance(exc, BaseExceptionGroup) and len(exc.exceptions) == 1:
                            exc = exc.exceptions[0]

                    raise exc

类型注释内存对象流的语法已更改changed
-----------------------------------------------------------

**Syntax for type annotated memory object streams has **

.. tabs::

    .. tab:: 中文

        以前，创建类型注解的内存对象流是通过将所需的类型作为第二个参数传递来实现的::

            send, receive = create_memory_object_stream(100, int)

        在 4.0 中，:class:`create_memory_object_stream() <create_memory_object_stream>` 是一个伪装成函数的类，因此你需要为它提供类型参数化::

            send, receive = create_memory_object_stream 

        如果您以前没有为您的内存对象流进行类型参数化，那么在这方面不需要进行任何更改。

    .. tab:: 英文

        Where previously, creating type annotated memory object streams worked by passing the
        desired type as the second argument::

            send, receive = create_memory_object_stream(100, int)

        In 4.0, :class:`create_memory_object_stream() <create_memory_object_stream>` is a class
        masquerading as a function, so you need to parametrize it::

            send, receive = create_memory_object_stream[int](100)

        If you didn't parametrize your memory object streams before, then you don't need to make
        any changes in this regard.

事件循环工厂而不是事件循环策略
----------------------------------------------------

**Event loop factories instead of event loop policies**

.. tabs::

    .. tab:: 中文

        如果你正在使用自定义的 asyncio 事件循环策略与 :func:`run` 一起，你需要改为传递一个 *事件循环工厂(event loop factory)*，即一个返回新事件循环的可调用对象。

        以 uvloop_ 为例，像以下这样的代码::

            anyio.run(main, backend_options={"event_loop_policy": uvloop.EventLoopPolicy()})

        应该转换为::

            anyio.run(main, backend_options={"loop_factory": uvloop.new_event_loop})

        确保不要实际调用工厂函数！

    .. tab:: 英文

        If you're using a custom asyncio event loop policy with :func:`run`, you need to switch
        to passing an *event loop factory*, that is, a callable that returns a new event loop.

        Using uvloop_ as an example, code like the following::

            anyio.run(main, backend_options={"event_loop_policy": uvloop.EventLoopPolicy()})

        should be converted into::

            anyio.run(main, backend_options={"loop_factory": uvloop.new_event_loop})

        Make sure not to actually call the factory function!

.. _exceptiongroup: https://pypi.org/project/exceptiongroup/
.. _uvloop: https://github.com/MagicStack/uvloop

从 AnyIO 2 迁移到 AnyIO 3
=================================

**Migrating from AnyIO 2 to AnyIO 3**

.. tabs::

    .. tab:: 中文

        AnyIO 3 更改了一些函数和方法，这需要你在代码中进行相应的调整。所有已弃用的函数和方法将在 AnyIO 4 中被移除。

    .. tab:: 英文

        AnyIO 3 changed some functions and methods in a way that needs some adaptation in your
        code. All deprecated functions and methods will be removed in AnyIO 4.

异步函数转换为同步
-----------------------------------------------

**Asynchronous functions converted to synchronous**

.. tabs::

    .. tab:: 中文

        AnyIO 3 将几个先前的异步函数和方法更改为常规函数，原因有以下两点：

        #. 更好地满足第三方库使用同步回调的用例需求。
        #. 更好地匹配 Trio_ 的 API。

        以下函数和方法已被更改：

        * :func:`current_time`
        * :func:`current_effective_deadline`
        * :meth:`CancelScope.cancel() <.CancelScope.cancel>`
        * :meth:`CapacityLimiter.acquire_nowait`
        * :meth:`CapacityLimiter.acquire_on_behalf_of_nowait`
        * :meth:`Condition.release`
        * :meth:`Event.set`
        * :func:`get_current_task`
        * :func:`get_running_tasks`
        * :meth:`Lock.release`
        * :meth:`MemoryObjectReceiveStream.receive_nowait() <.streams.memory.MemoryObjectReceiveStream.receive_nowait>`
        * :meth:`MemoryObjectSendStream.send_nowait() <.streams.memory.MemoryObjectSendStream.send_nowait>`
        * :func:`open_signal_receiver`
        * :meth:`Semaphore.release`

        在迁移到 AnyIO 3 时，只需去掉每个调用中的 ``await``。

        .. note:: 出于向后兼容性原因，:func:`current_time`、:func:`current_effective_deadline` 和 :func:`get_running_tasks` 返回的对象是其原始类型的可等待版本（分别为 :class:`float` 和 :class:`list`）。这些可等待版本是原始类型的子类，因此它们应与原始版本表现一致。但如果你绝对需要原始类型，可以根据需要对返回值使用 ``maybe_async`` 或 ``float()`` / ``list()``。

        以下异步上下文管理器改为常规上下文管理器：

        * :func:`fail_after`
        * :func:`move_on_after`
        * ``open_cancel_scope()``（现为 ``CancelScope()``）

        在迁移时，只需将 ``async with`` 更改为普通的 ``with``。

        除了
        :meth:`MemoryObjectReceiveStream.receive_nowait()
        <.streams.memory.MemoryObjectReceiveStream.receive_nowait>`
        以外，它们都可以继续按原样使用——不过在 AnyIO 3 上以这种方式使用时将引发 :exc:`DeprecationWarning`。

        如果你在编写需要兼容两个主要版本的库，则需要使用 AnyIO 2.2 中添加的兼容性函数：``maybe_async()`` 和 ``maybe_async_cm()``。它们分别允许你安全地使用函数/方法和上下文管理器，而不论当前安装的是哪个主要版本。

        示例 1 —— 设置事件::

            from anyio.abc import Event
            from anyio import maybe_async


            async def foo(event: Event):
                await maybe_async(event.set())
                ...

        示例 2 —— 打开取消作用域::

            from anyio import CancelScope, maybe_async_cm

            async def foo():
                async with maybe_async_cm(CancelScope()) as scope:
                    ...

    .. tab:: 英文

        AnyIO 3 changed several previously asynchronous functions and methods into regular ones
        for two reasons:

        #. to better serve use cases where synchronous callbacks are used by third party
        libraries
        #. to better match the API of Trio_

        The following functions and methods were changed:

        * :func:`current_time`
        * :func:`current_effective_deadline`
        * :meth:`CancelScope.cancel() <.CancelScope.cancel>`
        * :meth:`CapacityLimiter.acquire_nowait`
        * :meth:`CapacityLimiter.acquire_on_behalf_of_nowait`
        * :meth:`Condition.release`
        * :meth:`Event.set`
        * :func:`get_current_task`
        * :func:`get_running_tasks`
        * :meth:`Lock.release`
        * :meth:`MemoryObjectReceiveStream.receive_nowait()
        <.streams.memory.MemoryObjectReceiveStream.receive_nowait>`
        * :meth:`MemoryObjectSendStream.send_nowait()
        <.streams.memory.MemoryObjectSendStream.send_nowait>`
        * :func:`open_signal_receiver`
        * :meth:`Semaphore.release`

        When migrating to AnyIO 3, simply remove the ``await`` from each call to these.

        .. note:: For backwards compatibility reasons, :func:`current_time`,
        :func:`current_effective_deadline` and :func:`get_running_tasks` return objects which
        are awaitable versions of their original types (:class:`float` and :class:`list`,
        respectively). These awaitable versions are subclasses of the original types so they
        should behave as their originals, but if you absolutely need the pristine original
        types, you can either use ``maybe_async`` or ``float()`` / ``list()`` on the returned
        value as appropriate.

        The following async context managers changed to regular context managers:

        * :func:`fail_after`
        * :func:`move_on_after`
        * ``open_cancel_scope()`` (now just ``CancelScope()``)

        When migrating, just change ``async with`` into a plain ``with``.

        With the exception of
        :meth:`MemoryObjectReceiveStream.receive_nowait()
        <.streams.memory.MemoryObjectReceiveStream.receive_nowait>`,
        all of them can still be used like before – they will raise :exc:`DeprecationWarning`
        when used this way on AnyIO 3, however.

        If you're writing a library that needs to be compatible with both major releases, you
        will need to use the compatibility functions added in AnyIO 2.2: ``maybe_async()`` and
        ``maybe_async_cm()``. These will let you safely use functions/methods and context
        managers (respectively) regardless of which major release is currently installed.

        Example 1 – setting an event::

            from anyio.abc import Event
            from anyio import maybe_async


            async def foo(event: Event):
                await maybe_async(event.set())
                ...

        Example 2 – opening a cancel scope::

            from anyio import CancelScope, maybe_async_cm

            async def foo():
                async with maybe_async_cm(CancelScope()) as scope:
                    ...

.. _Trio: https://github.com/python-trio/trio

启动任务
--------------

**Starting tasks**

.. tabs::

    .. tab:: 中文

        ``TaskGroup.spawn()`` 协程方法已被弃用，推荐使用同步方法 :meth:`.TaskGroup.start_soon` （该方法与 Trio 的 “nurseries” 中的 ``start_soon()`` 方法相对应）。如果你完全迁移到 AnyIO 3，只需切换到调用新方法（并去掉 ``await``）。

        如果代码需要兼容 AnyIO 2 和 3，你可以继续使用 ``TaskGroup.spawn()``（直到 AnyIO 4）并抑制弃用警告::

            import warnings

            async def foo():
                async with create_task_group() as tg:
                    with warnings.catch_warnings():
                        await tg.spawn(otherfunc)

    .. tab:: 英文

        The ``TaskGroup.spawn()`` coroutine method has been deprecated in favor of the
        synchronous method :meth:`.TaskGroup.start_soon` (which mirrors ``start_soon()`` in
        Trio's nurseries). If you're fully migrating to AnyIO 3, simply switch to calling the
        new method (and remove the ``await``).

        If your code needs to work with both AnyIO 2 and 3, you can keep using
        ``TaskGroup.spawn()`` (until AnyIO 4) and suppress the deprecation warning::

            import warnings

            async def foo():
                async with create_task_group() as tg:
                    with warnings.catch_warnings():
                        await tg.spawn(otherfunc)

阻止门户更改
-----------------------

**Blocking portal changes**

.. tabs::

    .. tab:: 中文

        AnyIO 现在**要求** :func:`.from_thread.start_blocking_portal` 作为上下文管理器使用::

            from anyio import sleep
            from anyio.from_thread import start_blocking_portal

            with start_blocking_portal() as portal:
                portal.call(sleep, 1)

        与 ``TaskGroup.spawn()`` 类似， ``BlockingPortal.spawn_task()`` 方法也已重命名为 :meth:`~from_thread.BlockingPortal.start_task_soon`，以便与任务组一致。

        ``create_blocking_portal()`` 工厂函数也被弃用，建议直接实例化 :class:`~from_thread.BlockingPortal`。

        对于需要跨版本兼容的代码，可以通过捕获弃用警告（如上所示）来处理。

    .. tab:: 英文

        AnyIO now **requires** :func:`.from_thread.start_blocking_portal` to be used as a
        context manager::

            from anyio import sleep
            from anyio.from_thread import start_blocking_portal

            with start_blocking_portal() as portal:
                portal.call(sleep, 1)

        As with ``TaskGroup.spawn()``, the ``BlockingPortal.spawn_task()`` method has also been
        renamed to :meth:`~from_thread.BlockingPortal.start_task_soon`, so as to be consistent
        with task groups.

        The ``create_blocking_portal()`` factory function was also deprecated in favor of
        instantiating :class:`~from_thread.BlockingPortal` directly.

        For code requiring cross compatibility, catching the deprecation warning (as above)
        should work.

同步原语
--------------------------

**Synchronization primitives**

.. tabs::

    .. tab:: 中文

        同步原语工厂函数（如 ``create_event()`` 等）已被弃用，建议直接实例化类。因此，将如下代码::

            from anyio import create_event

            async def main():
                event = create_event()

        转换为::

            from anyio import Event

            async def main():
                event = Event()

        或者，如果需要兼容 AnyIO 2 和 3，则可以这样处理::

            try:
                from anyio import Event
                create_event = Event
            except ImportError:
                from anyio import create_event
                from anyio.abc import Event

            async def foo() -> Event:
                return create_event()

    .. tab:: 英文

        Synchronization primitive factories (``create_event()`` etc.) were deprecated in favor
        of instantiating the classes directly. So convert code like this::

            from anyio import create_event

            async def main():
                event = create_event()

        into this::

            from anyio import Event

            async def main():
                event = Event()

        or, if you need to work with both AnyIO 2 and 3::

            try:
                from anyio import Event
                create_event = Event
            except ImportError:
                from anyio import create_event
                from anyio.abc import Event

            async def foo() -> Event:
                return create_event()

线程函数已移动
-------------------------

**Threading functions moved**

.. tabs::

    .. tab:: 中文

        线程函数已被重构至子模块，参考了 Trio 的模块化方式：

        * ``current_default_worker_thread_limiter`` → :func:`.to_thread.current_default_thread_limiter` （注意：函数名称也被重命名！）
        * ``run_sync_in_worker_thread()`` → :func:`.to_thread.run_sync`
        * ``run_async_from_thread()`` → :func:`.from_thread.run`
        * ``run_sync_from_thread()`` → :func:`.from_thread.run_sync`

        旧版本函数仍可使用，但调用时会发出弃用警告。

    .. tab:: 英文

        Threading functions were restructured to submodules, following the example of Trio:

        * ``current_default_worker_thread_limiter`` → :func:`.to_thread.current_default_thread_limiter` (NOTE: the function was renamed too!)
        * ``run_sync_in_worker_thread()`` → :func:`.to_thread.run_sync`
        * ``run_async_from_thread()`` → :func:`.from_thread.run`
        * ``run_sync_from_thread()`` → :func:`.from_thread.run_sync`

        The old versions are still in place but emit deprecation warnings when called.
