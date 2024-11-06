常见问题
==========================

**Frequently Asked Questions**

为什么 Curio 不支持作为后端？
----------------------------------------

**Why is Curio not supported as a backend?**

.. tabs::

    .. tab:: 中文

        在 AnyIO 3.0 之前版本中，AnyIO 支持 Curio_。支持被移除的原因有以下两点：

        #. Curio_ 的接口仅允许协程函数访问 Curio_ 内核。这迫使 AnyIO 在其 API 设计中采取相同的策略，从而使依赖同步回调的现有应用程序难以适配 AnyIO。这也与 AnyIO 希望在相同功能的函数中匹配 Trio 的 API 的目标产生冲突（例如 ``Event.set()``）。
        #. Curio_ 的维护者明确要求将其支持从 AnyIO 中移除（`issue 185 <https://github.com/agronholm/anyio/issues/185>`_）。

    .. tab:: 英文

      Curio_ was supported in AnyIO before v3.0. Support for it was dropped for two reasons:

      #. Its interface allowed only coroutine functions to access the Curio_ kernel. This
         forced AnyIO to follow suit in its own API design, making it difficult to adapt
         existing applications that relied on synchronous callbacks to use AnyIO. It also
         interfered with the goal of matching Trio's API in functions with the same purpose
         (e.g. ``Event.set()``).
      #. The maintainer specifically requested Curio_ support to be removed from AnyIO
         (`issue 185 <https://github.com/agronholm/anyio/issues/185>`_).

.. _Curio: https://github.com/dabeaz/curio

为什么 Twisted 不支持作为后端？
------------------------------------------

**Why is Twisted not supported as a backend?**

.. tabs::

    .. tab:: 中文

        支持 Twisted_ 的最低要求是 sniffio_ 能够检测到正在运行的 Twisted 事件循环（并能够识别 Twisted_ 是否在其 asyncio reactor 上运行）。目前， sniffio_ 不支持此功能，因此 AnyIO 也无法支持 Twisted。

        如果您对 AnyIO 中的 Twisted 支持感兴趣，可以关注 Twisted 的相关 `问题 <https://github.com/twisted/twisted/pull/1263>`_ 。

    .. tab:: 英文

        The minimum requirement to support Twisted_ would be for sniffio_ to be able to detect a
        running Twisted event loop (and be able to tell when Twisted_ is being run on top of its
        asyncio reactor). This is not currently supported in sniffio_, so AnyIO cannot support
        Twisted either.

        There is a Twisted `issue <https://github.com/twisted/twisted/pull/1263>`_ that you can
        follow if you're interested in Twisted support in AnyIO.

.. _Twisted: https://twistedmatrix.com/trac/
.. _sniffio: https://github.com/python-trio/sniffio
