使用 AnyIO 进行测试
==================

**Testing with AnyIO**

.. tabs::

    .. tab:: 中文

        AnyIO 提供了内置支持，用于通过 pytest_ 插件测试你的库或应用程序。

    .. tab:: 英文

        AnyIO provides built-in support for testing your library or application in the form of a pytest_ plugin.

.. _pytest: https://docs.pytest.org/en/latest/

创建异步测试
---------------------------

**Creating asynchronous tests**

.. tabs::

    .. tab:: 中文

        Pytest 本身不支持运行异步测试函数，因此需要标记这些函数，以便 AnyIO pytest 插件能够识别它们。可以通过以下两种方式之一来实现：

        #. 使用 ``pytest.mark.anyio`` 标记
        #. 使用 ``anyio_backend`` 固定器，直接使用或通过其他固定器

        最简单的方法是如下所示::

            import pytest

            # 这与在模块中的所有测试函数上使用 @pytest.mark.anyio 相同
            pytestmark = pytest.mark.anyio


            async def test_something():
                ...

        使用此标记标记模块、类或函数，与在它们上应用 ``pytest.mark.usefixtures('anyio_backend')`` 的效果相同。

        因此，你也可以在测试和固定器中直接要求使用该固定器::

            import pytest


            async def test_something(anyio_backend):
                ...

    .. tab:: 英文

        Pytest does not natively support running asynchronous test functions, so they have to be
        marked for the AnyIO pytest plugin to pick them up. This can be done in one of two ways:

        #. Using the ``pytest.mark.anyio`` marker
        #. Using the ``anyio_backend`` fixture, either directly or via another fixture

        The simplest way is thus the following::

            import pytest

            # This is the same as using the @pytest.mark.anyio on all test functions in the module
            pytestmark = pytest.mark.anyio


            async def test_something():
                ...

        Marking modules, classes or functions with this marker has the same effect as applying
        the ``pytest.mark.usefixtures('anyio_backend')`` on them.

        Thus, you can also require the fixture directly in your tests and fixtures::

            import pytest


            async def test_something(anyio_backend):
                ...

指定要运行的后端
---------------------------------

**Specifying the backends to run on**

.. tabs::

    .. tab:: 中文

        ``anyio_backend`` 固定器决定了测试和固定器运行时使用的后端及其选项。AnyIO pytest 插件提供了一个函数作用域的固定器，该固定器的名称为 ``anyio_backend``, 它会在所有支持的后端上运行所有内容。

        如果你想为整个项目更改后端/选项，可以在顶级的 ``conftest.py`` 文件中放置如下内容::

            @pytest.fixture
            def anyio_backend():
                return 'asyncio'

        如果你想为选定的后端指定不同的选项，可以通过传递一个包含（后端名称，选项字典）的元组来实现::

            @pytest.fixture(params=[
                pytest.param(('asyncio', {'use_uvloop': True}), id='asyncio+uvloop'),
                pytest.param(('asyncio', {'use_uvloop': False}), id='asyncio'),
                pytest.param(('trio', {'restrict_keyboard_interrupt_to_checkpoints': True}), id='trio')
            ])
            def anyio_backend(request):
                return request.param

        如果你需要在特定后端上运行单个测试，可以使用 ``@pytest.mark.parametrize``（记得将 ``anyio_backend`` 参数添加到实际的测试函数中，否则 pytest 会报错）::

            @pytest.mark.parametrize('anyio_backend', ['asyncio'])
            async def test_on_asyncio_only(anyio_backend):
                ...

        由于 ``anyio_backend`` 固定器可以返回字符串或元组，因此为了方便，你还可以使用以下两个额外的函数作用域固定器（它们本身依赖于 ``anyio_backend`` 固定器）：

        * ``anyio_backend_name``：后端的名称（例如 ``asyncio``）
        * ``anyio_backend_options``：用于运行该后端的选项字典

    .. tab:: 英文

        The ``anyio_backend`` fixture determines the backends and their options that tests and
        fixtures are run with. The AnyIO pytest plugin comes with a function scoped fixture with
        this name which runs everything on all supported backends.

        If you change the backends/options for the entire project, then put something like this
        in your top level ``conftest.py``::

            @pytest.fixture
            def anyio_backend():
                return 'asyncio'

        If you want to specify different options for the selected backend, you can do so by
        passing a tuple of (backend name, options dict)::

            @pytest.fixture(params=[
                pytest.param(('asyncio', {'use_uvloop': True}), id='asyncio+uvloop'),
                pytest.param(('asyncio', {'use_uvloop': False}), id='asyncio'),
                pytest.param(('trio', {'restrict_keyboard_interrupt_to_checkpoints': True}), id='trio')
            ])
            def anyio_backend(request):
                return request.param

        If you need to run a single test on a specific backend, you can use
        ``@pytest.mark.parametrize`` (remember to add the ``anyio_backend`` parameter to the
        actual test function, or pytest will complain)::

            @pytest.mark.parametrize('anyio_backend', ['asyncio'])
            async def test_on_asyncio_only(anyio_backend):
                ...

        Because the ``anyio_backend`` fixture can return either a string or a tuple, there are
        two additional function-scoped fixtures (which themselves depend on the
        ``anyio_backend`` fixture) provided for your convenience:

        * ``anyio_backend_name``: the name of the backend (e.g. ``asyncio``)
        * ``anyio_backend_options``: the dictionary of option keywords used to run the backend

异步装置
---------------------

**Asynchronous fixtures**

.. tabs::

    .. tab:: 中文

        该插件还支持协程函数作为固定器，用于设置和销毁测试中使用的异步服务。

        有两种方法可以让 AnyIO pytest 插件运行你的异步固定器：

        #. 在启用 AnyIO 的测试中使用它们（参见第一部分）
        #. 在固定器本身中使用 ``anyio_backend`` 固定器（或任何使用它的其他固定器）

        最简单的方法是使用第一种选项::

            import pytest

            pytestmark = pytest.mark.anyio


            @pytest.fixture
            async def server():
                server = await setup_server()
                yield server
                await server.shutdown()


            async def test_server(server):
                result = await server.do_something()
                assert result == 'foo'


        对于 ``autouse=True`` 的固定器，你可能需要使用另一种方法::

            @pytest.fixture(autouse=True)
            async def server(anyio_backend):
                server = await setup_server()
                yield
                await server.shutdown()


            async def test_server():
                result = await client.do_something_on_the_server()
                assert result == 'foo'

    .. tab:: 英文

        The plugin also supports coroutine functions as fixtures, for the purpose of setting up
        and tearing down asynchronous services used for tests.

        There are two ways to get the AnyIO pytest plugin to run your asynchronous fixtures:

        #. Use them in AnyIO enabled tests (see the first section)
        #. Use the ``anyio_backend`` fixture (or any other fixture using it) in the fixture
        itself

        The simplest way is using the first option::

            import pytest

            pytestmark = pytest.mark.anyio


            @pytest.fixture
            async def server():
                server = await setup_server()
                yield server
                await server.shutdown()


            async def test_server(server):
                result = await server.do_something()
                assert result == 'foo'


        For ``autouse=True`` fixtures, you may need to use the other approach::

            @pytest.fixture(autouse=True)
            async def server(anyio_backend):
                server = await setup_server()
                yield
                await server.shutdown()


            async def test_server():
                result = await client.do_something_on_the_server()
                assert result == 'foo'


使用具有更高范围的异步装置
---------------------------------------

**Using async fixtures with higher scopes**

.. tabs::

    .. tab:: 中文

        对于作用域不是 ``function`` 的异步固定器，你需要定义你自己的 ``anyio_backend`` 固定器，因为默认的 ``anyio_backend`` 固定器是函数作用域的::

            @pytest.fixture(scope='module')
            def anyio_backend():
                return 'asyncio'


            @pytest.fixture(scope='module')
            async def server(anyio_backend):
                server = await setup_server()
                yield
                await server.shutdown()

    .. tab:: 英文

        For async fixtures with scopes other than ``function``, you will need to define your own
        ``anyio_backend`` fixture because the default ``anyio_backend`` fixture is function
        scoped::

            @pytest.fixture(scope='module')
            def anyio_backend():
                return 'asyncio'


            @pytest.fixture(scope='module')
            async def server(anyio_backend):
                server = await setup_server()
                yield
                await server.shutdown()

技术细节
-----------------

**Technical details**

.. tabs::

    .. tab:: 中文

        固定器和测试是由一个“测试运行器”执行的，该运行器为每个后端单独实现。测试运行器在请求期间保持事件循环打开，使得固定器中的代码可以与测试中的代码（以及彼此之间）进行通信。

        当第一个匹配的异步测试或固定器即将运行时，测试运行器被创建；当该固定器被拆卸或测试完成时，测试运行器被关闭。因此，如果没有使用更高层次的（作用域为 ``class`` 或更高）异步固定器，则为每个匹配的测试创建一个单独的测试运行器。相反，如果有至少一个作用域高于 ``function`` 的异步固定器在所有测试之间共享，则在测试会话期间只会创建一个测试运行器。

    .. tab:: 英文

        The fixtures and tests are run by a "test runner", implemented separately for each
        backend. The test runner keeps an event loop open during the request, making it possible
        for code in fixtures to communicate with the code in the tests (and each other).

        The test runner is created when the first matching async test or fixture is about to be
        run, and shut down when that same fixture is being torn down or the test has finished
        running. As such, if no higher-order (scoped ``class`` or higher) async fixtures are
        used, a separate test runner is created for each matching test. Conversely, if even one
        async fixture, scoped higher than ``function``, is shared across all tests, only one
        test runner will be created during the test session.

上下文变量传播
++++++++++++++++++++++++++++

**Context variable propagation**

.. tabs::

    .. tab:: 中文

        异步测试运行器在同一个任务中运行所有异步固定器和测试，因此在异步测试运行器中设置的上下文变量会影响同一运行器中的其他异步固定器和测试。然而，这些上下文变量**不会**被传递到同步测试和固定器，或者其他异步测试运行器。

    .. tab:: 英文

        The asynchronous test runner runs all async fixtures and tests in the same task, so
        context variables set in async fixtures or tests, within an async test runner, will
        affect other async fixtures and tests within the same runner. However, these context
        variables are **not** carried over to synchronous tests and fixtures, or to other async
        test runners.

与其他异步测试运行器的比较
++++++++++++++++++++++++++++++++++++++++

**Comparison with other async test runners**

.. tabs::

    .. tab:: 中文

        ``pytest-asyncio`` 库仅适用于 asyncio 代码。与 AnyIO pytest 插件类似，它可以通过指定更高层次的 ``event_loop`` 固定器来支持更高层次的固定器。然而，它在每个操作的异步固定器的设置和拆卸阶段都会在一个新的异步任务中运行，这使得上下文变量的传播变得不可能，并且阻止了任务组和取消作用域的正常功能。

        ``pytest-trio`` 库用于测试 Trio 项目，仅适用于 Trio 代码。此外，它仅支持函数作用域的异步固定器。与 AnyIO pytest 插件的另一个显著区别是，当异步固定器的依赖图允许时，它会尝试并发地运行异步固定器的设置和拆卸过程。

    .. tab:: 英文

        The ``pytest-asyncio`` library only works with asyncio code. Like the AnyIO pytest
        plugin, it can be made to support higher order fixtures (by specifying a higher order
        ``event_loop`` fixture). However, it runs the setup and teardown phases of each async
        fixture in a new async task per operation, making context variable propagation
        impossible and preventing task groups and cancel scopes from functioning properly.

        The ``pytest-trio`` library, made for testing Trio projects, works only with Trio code.
        Additionally, it only supports function scoped async fixtures. Another significant
        difference with the AnyIO pytest plugin is that attempts to run the setup and teardown
        for async fixtures concurrently when their dependency graphs allow that.
