使用同步原语
================================

**Using synchronization primitives**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        同步原语是任务用来相互通信和协调的对象。它们对于分配工作负载、通知其他任务以及保护对共享资源的访问等操作非常有用。

        .. note:: AnyIO 原语不是线程安全的，因此不应直接从工作线程中使用。对于这种情况，请使用 :func:`~from_thread.run_sync` 。

    .. tab:: 英文

        Synchronization primitives are objects that are used by tasks to communicate and
        coordinate with each other. They are useful for things like distributing workload,
        notifying other tasks and guarding access to shared resources.

        .. note:: AnyIO primitives are not thread-safe, therefore they should not be used
        directly from worker threads.  Use :func:`~from_thread.run_sync` for that.

事件
------

**Events**

.. tabs::

    .. tab:: 中文

        事件用于通知任务它们一直在等待的事情已经发生。一个事件对象可以有多个监听者，当事件被触发时，所有监听者都会被通知。

        示例::

            from anyio import Event, create_task_group, run


            async def notify(event):
                event.set()


            async def main():
                event = Event()
                async with create_task_group() as tg:
                    tg.start_soon(notify, event)
                    await event.wait()
                    print('Received notification!')

            run(main)

        .. note:: 与标准库的事件不同，AnyIO 事件不能被重用，必须被替换。这种做法防止了一类竞争条件，并且与 Trio 库的语义一致。

    .. tab:: 英文

        Events are used to notify tasks that something they've been waiting to happen has
        happened. An event object can have multiple listeners and they are all notified when the
        event is triggered.

        Example::

            from anyio import Event, create_task_group, run


            async def notify(event):
                event.set()


            async def main():
                event = Event()
                async with create_task_group() as tg:
                    tg.start_soon(notify, event)
                    await event.wait()
                    print('Received notification!')

            run(main)

        .. note:: Unlike standard library Events, AnyIO events cannot be reused, and must be
            replaced instead. This practice prevents a class of race conditions, and matches the
            semantics of the Trio library.

信号量
----------

**Semaphores**

.. tabs::

    .. tab:: 中文

        信号量用于限制对共享资源的访问。信号量从一个最大值开始，每次任务获取信号量时，该值会递减，而释放时会递增。如果值降到零，任何获取信号量的尝试都会被阻塞，直到另一个任务释放它。

        示例::

            from anyio import Semaphore, create_task_group, sleep, run

            async def use_resource(tasknum, semaphore):
                async with semaphore:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                semaphore = Semaphore(2)
                async with create_task_group() as tg:
                    for num in range(10):
                        tg.start_soon(use_resource, num, semaphore)

            run(main)

        .. tip:: 如果信号量的性能对你来说很重要，你可以将 ``fast_acquire=True`` 传递给 :class:`Semaphore` 。这样做会跳过 :meth:`Semaphore.acquire` 中的 :func:`~.lowlevel.cancel_shielded_checkpoint` 调用，如果没有争用（即获取信号量立即成功）。在某些情况下，这可能会导致任务在使用信号量的循环中永远不会将控制权交还给事件循环，特别是在该循环没有其他 yield 点的情况下。

    .. tab:: 英文

        Semaphores are used for limiting access to a shared resource. A semaphore starts with a
        maximum value, which is decremented each time the semaphore is acquired by a task and
        incremented when it is released. If the value drops to zero, any attempt to acquire the
        semaphore will block until another task frees it.

        Example::

            from anyio import Semaphore, create_task_group, sleep, run


            async def use_resource(tasknum, semaphore):
                async with semaphore:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                semaphore = Semaphore(2)
                async with create_task_group() as tg:
                    for num in range(10):
                        tg.start_soon(use_resource, num, semaphore)

            run(main)

        .. tip:: If the performance of semaphores is critical for you, you could pass
        ``fast_acquire=True`` to :class:`Semaphore`. This has the effect of skipping the
        :func:`~.lowlevel.cancel_shielded_checkpoint` call in :meth:`Semaphore.acquire` if
        there is no contention (acquisition succeeds immediately). This could, in some cases,
        lead to the task never yielding control back to to the event loop if you use the
        semaphore in a loop that does not have other yield points.

锁
-----

**Locks**

.. tabs::

    .. tab:: 中文

        锁用于保护共享资源，以确保每次只有一个任务能够独占访问。它们的功能类似于最大值为 1 的信号量，唯一的区别是，只有获取锁的任务才被允许释放它。

        示例::

            from anyio import Lock, create_task_group, sleep, run


            async def use_resource(tasknum, lock):
                async with lock:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                lock = Lock()
                async with create_task_group() as tg:
                    for num in range(4):
                        tg.start_soon(use_resource, num, lock)

            run(main)

        .. tip:: 如果锁的性能对你来说很重要，你可以将 ``fast_acquire=True`` 传递给 :class:`Lock`。这样做会跳过 :meth:`Lock.acquire` 中的 :func:`~.lowlevel.cancel_shielded_checkpoint` 调用，如果没有争用（即获取锁立即成功）。在某些情况下，这可能会导致任务在使用锁的循环中永远不会将控制权交还给事件循环，特别是在该循环没有其他 yield 点的情况下。 

    .. tab:: 英文

        Locks are used to guard shared resources to ensure sole access to a single task at once.
        They function much like semaphores with a maximum value of 1, except that only the task
        that acquired the lock is allowed to release it.

        Example::

            from anyio import Lock, create_task_group, sleep, run


            async def use_resource(tasknum, lock):
                async with lock:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                lock = Lock()
                async with create_task_group() as tg:
                    for num in range(4):
                        tg.start_soon(use_resource, num, lock)

            run(main)

        .. tip:: If the performance of locks is critical for you, you could pass
        ``fast_acquire=True`` to :class:`Lock`. This has the effect of skipping the
        :func:`~.lowlevel.cancel_shielded_checkpoint` call in :meth:`Lock.acquire` if there
        is no contention (acquisition succeeds immediately). This could, in some cases, lead
        to the task never yielding control back to to the event loop if use the lock in a
        loop that does not have other yield points.

条件
----------

**Conditions**

.. tabs::

    .. tab:: 中文

        条件实际上是事件和锁的结合体。它首先获取一个锁，然后等待来自事件的通知。一旦条件接收到通知，它将释放锁。通知任务还可以选择一次唤醒多个监听者，甚至是所有监听者。

        与 :class:`Lock` 类似，:class:`Condition` 也要求获取锁的任务必须是释放锁的任务。

        示例::

            from anyio import Condition, create_task_group, sleep, run


            async def listen(tasknum, condition):
                async with condition:
                    await condition.wait()
                    print('Woke up task number', tasknum)


            async def main():
                condition = Condition()
                async with create_task_group() as tg:
                    for tasknum in range(6):
                        tg.start_soon(listen, tasknum, condition)

                    await sleep(1)
                    async with condition:
                        condition.notify(1)

                    await sleep(1)
                    async with condition:
                        condition.notify(2)

                    await sleep(1)
                    async with condition:
                        condition.notify_all()

            run(main)

    .. tab:: 英文

        A condition is basically a combination of an event and a lock. It first acquires a lock
        and then waits for a notification from the event. Once the condition receives a
        notification, it releases the lock. The notifying task can also choose to wake up more
        than one listener at once, or even all of them.

        Like :class:`Lock`, :class:`Condition` also requires that the task which locked it also
        the one to release it.

        Example::

            from anyio import Condition, create_task_group, sleep, run


            async def listen(tasknum, condition):
                async with condition:
                    await condition.wait()
                    print('Woke up task number', tasknum)


            async def main():
                condition = Condition()
                async with create_task_group() as tg:
                    for tasknum in range(6):
                        tg.start_soon(listen, tasknum, condition)

                    await sleep(1)
                    async with condition:
                        condition.notify(1)

                    await sleep(1)
                    async with condition:
                        condition.notify(2)

                    await sleep(1)
                    async with condition:
                        condition.notify_all()

            run(main)

容量限制器
-----------------

**Capacity limiters**

.. tabs::

    .. tab:: 中文

        容量限制器类似于信号量，不同之处在于单个借用者（默认是当前任务）一次只能持有一个令牌。也可以代表任何可哈希对象借用令牌。

        示例::

            from anyio import CapacityLimiter, create_task_group, sleep, run


            async def use_resource(tasknum, limiter):
                async with limiter:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                limiter = CapacityLimiter(2)
                async with create_task_group() as tg:
                    for num in range(10):
                        tg.start_soon(use_resource, num, limiter)

            run(main)

        你可以通过设置限制器的 ``total_tokens`` 属性来调整令牌的总数。

    .. tab:: 英文

        Capacity limiters are like semaphores except that a single borrower (the current task by
        default) can only hold a single token at a time. It is also possible to borrow a token
        on behalf of any arbitrary object, so long as that object is hashable.

        Example::

            from anyio import CapacityLimiter, create_task_group, sleep, run


            async def use_resource(tasknum, limiter):
                async with limiter:
                    print('Task number', tasknum, 'is now working with the shared resource')
                    await sleep(1)


            async def main():
                limiter = CapacityLimiter(2)
                async with create_task_group() as tg:
                    for num in range(10):
                        tg.start_soon(use_resource, num, limiter)

            run(main)

        You can adjust the total number of tokens by setting a different value on the limiter's
        ``total_tokens`` property.

资源保护器
---------------

**Resource guards**

.. tabs::

    .. tab:: 中文

        某些资源，如套接字，对于并发使用非常敏感，甚至不应允许尝试并发使用。在这种情况下，:class:`ResourceGuard` 是合适的解决方案::

            class Resource:
                def __init__(self):
                    self._guard = ResourceGuard()

                async def do_something() -> None:
                    with self._guard:
                        ...

        现在，如果另一个任务在第一个调用尚未完成之前尝试在同一个 ``Resource`` 实例上调用 ``do_something()`` 方法，这将引发 :exc:`BusyResourceError`。

    .. tab:: 英文

        Some resources, such as sockets, are very sensitive about concurrent use and should not
        allow even attempts to be used concurrently. For such cases, :class:`ResourceGuard` is
        the appropriate solution::

            class Resource:
                def __init__(self):
                    self._guard = ResourceGuard()

                async def do_something() -> None:
                    with self._guard:
                        ...

        Now, if another task tries calling the ``do_something()`` method on the same
        ``Resource`` instance before the first call has finished, that will raise a
        :exc:`BusyResourceError`.

队列
------

**Queues**

.. tabs::

    .. tab:: 中文

        作为队列的替代，AnyIO 提供了一个更强大的构造：:ref:`内存对象流 <memory object streams>`。

    .. tab:: 英文

        In place of queues, AnyIO offers a more powerful construct:
        :ref:`memory object streams <memory object streams>`.
