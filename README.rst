.. image:: https://github.com/agronholm/anyio/actions/workflows/test.yml/badge.svg
  :target: https://github.com/agronholm/anyio/actions/workflows/test.yml
  :alt: Build Status
.. image:: https://coveralls.io/repos/github/agronholm/anyio/badge.svg?branch=master
  :target: https://coveralls.io/github/agronholm/anyio?branch=master
  :alt: Code Coverage
.. image:: https://readthedocs.org/projects/anyio/badge/?version=latest
  :target: https://anyio.readthedocs.io/en/latest/?badge=latest
  :alt: Documentation
.. image:: https://badges.gitter.im/gitterHQ/gitter.svg
  :target: https://gitter.im/python-trio/AnyIO
  :alt: Gitter chat

AnyIO 是一个异步网络和并发库，基于 asyncio_ 或 trio_ 之上构建。它在 asyncio 上实现了类似 trio 的 `结构化并发 <structured concurrency>`_ (SC)，并与 trio 本身的原生 SC 和谐工作。

针对 AnyIO API 编写的应用程序和库可以在 asyncio_ 或 trio_ 上无修改地运行。AnyIO 还可以逐步地集成到库或应用程序中 —— 一点一点地进行，不需要完全重构。它能够与你选择的后端的原生库无缝融合。

文档
-------------

查看完整文档： https://hellowac.githu.io/anyio-zh-cn/

功能
--------

AnyIO 提供以下功能：

* 任务组（trio 术语中的 nurseries_）
* 高级网络功能（TCP、UDP 和 UNIX 套接字）

  * TCP 连接的 `Happy eyeballs`_ 算法（比 Python 3.8 中 asyncio 的更健壮）
  * 异步/等待风格的 UDP 套接字（与 asyncio 不同，后者仍需要使用传输层和协议）

* 用于字节流和对象流的多功能 API
* 任务间同步与通信（锁、条件、事件、信号量、对象流）
* 工作线程
* 子进程
* 异步文件 I/O（使用工作线程）
* 信号处理

AnyIO 还自带了 pytest_ 插件，支持异步固定装置（fixtures）。
它甚至与流行的 Hypothesis_ 库兼容。

.. _asyncio: https://docs.python.org/3/library/asyncio.html
.. _trio: https://github.com/python-trio/trio
.. _structured concurrency: https://en.wikipedia.org/wiki/Structured_concurrency
.. _nurseries: https://trio.readthedocs.io/en/stable/reference-core.html#nurseries-and-spawning
.. _Happy eyeballs: https://en.wikipedia.org/wiki/Happy_Eyeballs
.. _pytest: https://docs.pytest.org/en/latest/
.. _Hypothesis: https://hypothesis.works/
