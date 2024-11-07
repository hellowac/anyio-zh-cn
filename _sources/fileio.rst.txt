异步文件 I/O 支持
=============================

**Asynchronous file I/O support**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        AnyIO 提供了阻塞文件操作的异步包装器。这些包装器在工作线程中运行阻塞操作。

        示例::

            from anyio import open_file, run


            async def main():
                async with await open_file('/some/path/somewhere') as f:
                    contents = await f.read()
                    print(contents)

            run(main)

        这些包装器还支持按行异步迭代文件内容，就像标准文件对象支持同步迭代一样::

            from anyio import open_file, run


            async def main():
                async with await open_file('/some/path/somewhere') as f:
                    async for line in f:
                        print(line, end='')

            run(main)

        要将现有的打开文件对象包装为异步文件，可以使用 :func:`.wrap_file`::

            from anyio import wrap_file, run


            async def main():
                with open('/some/path/somewhere') as f:
                    async for line in wrap_file(f):
                        print(line, end='')

            run(main)

        .. note:: 关闭包装器时，也会关闭底层的同步文件对象。

        .. seealso:: :ref:`FileStreams`

    .. tab:: 英文

        AnyIO provides asynchronous wrappers for blocking file operations. These wrappers run
        blocking operations in worker threads.

        Example::

            from anyio import open_file, run


            async def main():
                async with await open_file('/some/path/somewhere') as f:
                    contents = await f.read()
                    print(contents)

            run(main)

        The wrappers also support asynchronous iteration of the file line by line, just as the
        standard file objects support synchronous iteration::

            from anyio import open_file, run


            async def main():
                async with await open_file('/some/path/somewhere') as f:
                    async for line in f:
                        print(line, end='')

            run(main)

        To wrap an existing open file object as an asynchronous file, you can use
        :func:`.wrap_file`::

            from anyio import wrap_file, run


            async def main():
                with open('/some/path/somewhere') as f:
                    async for line in wrap_file(f):
                        print(line, end='')

            run(main)

        .. note:: Closing the wrapper also closes the underlying synchronous file object.

        .. seealso:: :ref:`FileStreams`

异步路径操作
----------------------------

**Asynchronous path operations**

.. tabs::

    .. tab:: 中文

        AnyIO 提供了 :class:`pathlib.Path` 类的异步版本。它与原版有一些不同之处：

        * 执行磁盘 I/O 操作的方法（如 :meth:`~pathlib.Path.read_bytes`）会在工作线程中运行，因此需要使用 ``await``
        * 像 :meth:`~pathlib.Path.glob` 这样的函数返回一个异步迭代器，该迭代器生成异步的 :class:`~.Path` 对象
        * 正常返回 :class:`pathlib.Path` 对象的属性和方法，现在返回 :class:`~.Path` 对象
        * Python 3.10 API 中的方法和属性在所有版本中均可用
        * 不支持作为上下文管理器使用，因为在 pathlib 中已被弃用

        例如，创建一个包含二进制内容的文件可以这样做::

            from anyio import Path, run


            async def main():
                path = Path('/foo/bar')
                await path.write_bytes(b'hello, world')

            run(main)

        异步迭代目录内容可以按如下方式进行::

            from anyio import Path, run


            async def main():
                # 打印目录 /foo/bar 中每个文件的内容（假定是文本文件）
                dir_path = Path('/foo/bar')
                async for path in dir_path.iterdir():
                    if await path.is_file():
                        print(await path.read_text())
                        print('---------------------')

            run(main)

    .. tab:: 英文

        AnyIO provides an asynchronous version of the :class:`pathlib.Path` class. It differs
        with the original in a number of ways:

        * Operations that perform disk I/O (like :meth:`~pathlib.Path.read_bytes`) are run in a
        worker thread and thus require an ``await``
        * Methods like :meth:`~pathlib.Path.glob` return an asynchronous iterator that yields
        asynchronous :class:`~.Path` objects
        * Properties and methods that normally return :class:`pathlib.Path` objects return
        :class:`~.Path` objects instead
        * Methods and properties from the Python 3.10 API are available on all versions
        * Use as a context manager is not supported, as it is deprecated in pathlib

        For example, to create a file with binary content::

            from anyio import Path, run


            async def main():
                path = Path('/foo/bar')
                await path.write_bytes(b'hello, world')

            run(main)

        Asynchronously iterating a directory contents can be done as follows::

            from anyio import Path, run


            async def main():
                # Print the contents of every file (assumed to be text) in the directory /foo/bar
                dir_path = Path('/foo/bar')
                async for path in dir_path.iterdir():
                    if await path.is_file():
                        print(await path.read_text())
                        print('---------------------')

            run(main)
