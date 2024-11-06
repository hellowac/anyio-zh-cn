流
=======

**Streams**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        在 AnyIO 中, “流(stream)” 是一个简单的接口, 用于将信息从一个地方传输到另一个地方。它可以表示进程间通信或通过网络发送数据。AnyIO 将流分为两类：字节流(byte streams)和对象流(object streams)。

        字节流(在 Trio 术语中为“Streams”)是接收和/或发送字节块的对象。它们是基于流套接字的限制来建模的, 这意味着边界不会被严格遵守。实际上, 这意味着例如你调用 ``.send(b'hello ')`` 然后 ``.send(b'world')``, 另一端接收到的数据将被任意拆分, 如( ``b'hello'``  和 ``b' world'`` )、 ``b'hello world'`` 或( ``b'hel'``、 ``b'lo wo'``、 ``b'rld'``)。

        另一方面, 对象流(在 Trio 术语中为“Channels”)处理 Python 对象。这些流最常见的实现是内存对象流。对象流的具体语义因实现方式而异。

        许多流实现会包装其他流。其中一些可以包装任何字节导向的流, 即 ``ObjectStream[bytes]`` 和 ``ByteStream``。这使得许多有趣的用例成为可能。

    .. tab:: 英文

        A "stream" in AnyIO is a simple interface for transporting information from one place to
        another. It can mean either in-process communication or sending data over a network.
        AnyIO divides streams into two categories: byte streams and object streams.

        Byte streams ("Streams" in Trio lingo) are objects that receive and/or send chunks of
        bytes. They are modelled after the limitations of the stream sockets, meaning the
        boundaries are not respected. In practice this means that if, for example, you call
        ``.send(b'hello ')`` and then ``.send(b'world')``, the other end will receive the data
        chunked in any arbitrary way, like (``b'hello'`` and ``b' world'``), ``b'hello world'``
        or (``b'hel'``, ``b'lo wo'``, ``b'rld'``).

        Object streams ("Channels" in Trio lingo), on the other hand, deal with Python objects.
        The most commonly used implementation of these is the memory object stream. The exact
        semantics of object streams vary a lot by implementation.

        Many stream implementations wrap other streams. Of these, some can wrap any
        bytes-oriented streams, meaning ``ObjectStream[bytes]`` and ``ByteStream``. This enables
        many interesting use cases.

.. _memory object streams:

内存对象流
---------------------

**Memory object streams**

.. tabs::

    .. tab:: 中文

        内存对象流用于实现带有多个任务的生产者-消费者模式。通过使用 :func:`~create_memory_object_stream`, 你将获得一对对象流：一个用于发送, 一个用于接收。它们本质上像队列, 但支持关闭和异步迭代。

        默认情况下, 内存对象流创建时的缓冲区大小为 0。这意味着 :meth:`~.streams.memory.MemoryObjectSendStream.send` 将会阻塞, 直到有另一个任务调用 :meth:`~.streams.memory.MemoryObjectReceiveStream.receive`。你可以在创建流时设置自定义的缓冲区大小。也可以通过传递 :data:`math.inf` 作为缓冲区大小来创建一个无限缓冲区, 但这并不推荐使用。

        内存对象流可以通过调用 ``clone()`` 方法进行克隆。每个克隆可以单独关闭, 但只有当所有克隆都关闭时, 流的每一端才会被认为是关闭的。例如, 如果你有两个接收流的克隆, 发送流只有在两个接收流都关闭后才会开始引发 :exc:`~BrokenResourceError`。

        多个任务可以在同一个内存对象流(或其克隆)上进行发送和接收, 但每个发送的项只会交付给一个接收者。

        内存对象流的接收端可以使用异步迭代协议进行迭代。当所有发送流的克隆都关闭时, 循环会退出。

        示例::

            from anyio import create_task_group, create_memory_object_stream, run
            from anyio.streams.memory import MemoryObjectReceiveStream


            async def process_items(receive_stream: MemoryObjectReceiveStream[str]) -> None:
                async with receive_stream:
                    async for item in receive_stream:
                        print('received', item)


            async def main():
                # [str] 指定了通过内存对象流传递的对象类型。这是一个技巧, 因为 create_memory_object_stream
                # 实际上是一个伪装成函数的类。
                send_stream, receive_stream = create_memory_object_stream[str]()
                async with create_task_group() as tg:
                    tg.start_soon(process_items, receive_stream)
                    async with send_stream:
                        for num in range(10):
                            await send_stream.send(f'number {num}')

            run(main)

        与其他 AnyIO 流不同(但与 Trio 的 Channels 一致), 内存对象流可以同步关闭, 使用 ``close()`` 方法或将流作为上下文管理器来关闭::

            from anyio.streams.memory import MemoryObjectSendStream


            def synchronous_callback(send_stream: MemoryObjectSendStream[str]) -> None:
                with send_stream:
                    send_stream.send_nowait('hello')

    .. tab:: 英文

        Memory object streams are intended for implementing a producer-consumer pattern with
        multiple tasks. Using :func:`~create_memory_object_stream`, you get a pair of object
        streams: one for sending, one for receiving. They essentially work like queues, but with
        support for closing and asynchronous iteration.

        By default, memory object streams are created with a buffer size of 0. This means that
        :meth:`~.streams.memory.MemoryObjectSendStream.send` will block until there's another
        task that calls :meth:`~.streams.memory.MemoryObjectReceiveStream.receive`. You can set
        the buffer size to a value of your choosing when creating the stream. It is also
        possible to have an unbounded buffer by passing :data:`math.inf` as the buffer size but
        this is not recommended.

        Memory object streams can be cloned by calling the ``clone()`` method. Each clone can be
        closed separately, but each end of the stream is only considered closed once all of its
        clones have been closed. For example, if you have two clones of the receive stream, the
        send stream will start raising :exc:`~BrokenResourceError` only when both receive
        streams have been closed.

        Multiple tasks can send and receive on the same memory object stream (or its clones) but
        each sent item is only ever delivered to a single recipient.

        The receive ends of memory object streams can be iterated using the async iteration
        protocol. The loop exits when all clones of the send stream have been closed.

        Example::

            from anyio import create_task_group, create_memory_object_stream, run
            from anyio.streams.memory import MemoryObjectReceiveStream


            async def process_items(receive_stream: MemoryObjectReceiveStream[str]) -> None:
                async with receive_stream:
                    async for item in receive_stream:
                        print('received', item)


            async def main():
                # The [str] specifies the type of the objects being passed through the
                # memory object stream. This is a bit of trick, as create_memory_object_stream
                # is actually a class masquerading as a function.
                send_stream, receive_stream = create_memory_object_stream[str]()
                async with create_task_group() as tg:
                    tg.start_soon(process_items, receive_stream)
                    async with send_stream:
                        for num in range(10):
                            await send_stream.send(f'number {num}')

            run(main)

        In contrast to other AnyIO streams (but in line with Trio's Channels), memory object
        streams can be closed synchronously, using either the ``close()`` method or by using the
        stream as a context manager::

            from anyio.streams.memory import MemoryObjectSendStream


            def synchronous_callback(send_stream: MemoryObjectSendStream[str]) -> None:
                with send_stream:
                    send_stream.send_nowait('hello')

装订流
---------------

**Stapled streams**

.. tabs::

    .. tab:: 中文

        一个拼接流将任何互相兼容的接收流和发送流结合在一起, 形成一个单一的双向流。

        它有两种变体：

        * :class:`~.streams.stapled.StapledByteStream`(将 :class:`~.abc.ByteReceiveStream` 与 :class:`~.abc.ByteSendStream` 结合)
        * :class:`~.streams.stapled.StapledObjectStream`(将 :class:`~.abc.ObjectReceiveStream` 与兼容的 :class:`~.abc.ObjectSendStream` 结合)

    .. tab:: 英文

        A stapled stream combines any mutually compatible receive and send stream together,
        forming a single bidirectional stream.

        It comes in two variants:

        * :class:`~.streams.stapled.StapledByteStream` (combines a
        :class:`~.abc.ByteReceiveStream` with a :class:`~.abc.ByteSendStream`)
        * :class:`~.streams.stapled.StapledObjectStream` (combines an
        :class:`~.abc.ObjectReceiveStream` with a compatible :class:`~.abc.ObjectSendStream`)

缓冲字节流
---------------------

**Buffered byte streams**

.. tabs::

    .. tab:: 中文

        缓冲字节流包装了一个现有的字节导向接收流, 并提供了一些需要缓冲的功能, 例如接收指定数量的字节, 或接收直到找到给定的分隔符为止。

        示例::

            from anyio import run, create_memory_object_stream
            from anyio.streams.buffered import BufferedByteReceiveStream


            async def main():
                send, receive = create_memory_object_stream 
                buffered = BufferedByteReceiveStream(receive)
                for part in b'hel', b'lo, ', b'wo', b'rld!':
                    await send.send(part)

                result = await buffered.receive_exactly(8)
                print(repr(result))

                result = await buffered.receive_until(b'!', 10)
                print(repr(result))

            run(main)

        上述脚本会输出以下内容::

            b'hello, w'
            b'orld'

    .. tab:: 英文

        A buffered byte stream wraps an existing bytes-oriented receive stream and provides
        certain amenities that require buffering, such as receiving an exact number of bytes, or
        receiving until the given delimiter is found.

        Example::

            from anyio import run, create_memory_object_stream
            from anyio.streams.buffered import BufferedByteReceiveStream


            async def main():
                send, receive = create_memory_object_stream[bytes](4)
                buffered = BufferedByteReceiveStream(receive)
                for part in b'hel', b'lo, ', b'wo', b'rld!':
                    await send.send(part)

                result = await buffered.receive_exactly(8)
                print(repr(result))

                result = await buffered.receive_until(b'!', 10)
                print(repr(result))

            run(main)

        The above script gives the following output::

            b'hello, w'
            b'orld'

文本流
------------

**Text streams**

.. tabs::

    .. tab:: 中文

        文本流包装了现有的接收/发送流, 并将字符串编码/解码为字节, 反之亦然。

        示例::

            from anyio import run, create_memory_object_stream
            from anyio.streams.text import TextReceiveStream, TextSendStream


            async def main():
                bytes_send, bytes_receive = create_memory_object_stream 
                text_send = TextSendStream(bytes_send)
                await text_send.send('åäö')
                result = await bytes_receive.receive()
                print(repr(result))

                text_receive = TextReceiveStream(bytes_receive)
                await bytes_send.send(result)
                result = await text_receive.receive()
                print(repr(result))

            run(main)

        上述脚本会输出以下内容::

            b'\xc3\xa5\xc3\xa4\xc3\xb6'
            'åäö'

    .. tab:: 英文

        Text streams wrap existing receive/send streams and encode/decode strings to bytes and
        vice versa.

        Example::

            from anyio import run, create_memory_object_stream
            from anyio.streams.text import TextReceiveStream, TextSendStream


            async def main():
                bytes_send, bytes_receive = create_memory_object_stream[bytes](1)
                text_send = TextSendStream(bytes_send)
                await text_send.send('åäö')
                result = await bytes_receive.receive()
                print(repr(result))

                text_receive = TextReceiveStream(bytes_receive)
                await bytes_send.send(result)
                result = await text_receive.receive()
                print(repr(result))

            run(main)

        The above script gives the following output::

            b'\xc3\xa5\xc3\xa4\xc3\xb6'
            'åäö'

.. _FileStreams:

文件流
------------

**File streams**

.. tabs::

    .. tab:: 中文

        文件流用于从文件系统中读取或写入文件。它们对于将文件替代为其他数据源, 或将输出写入文件以用于日志记录或调试目的非常有用。

        示例::

            from anyio import run
            from anyio.streams.file import FileReadStream, FileWriteStream


            async def main():
                path = '/tmp/testfile'
                async with await FileWriteStream.from_path(path) as stream:
                    await stream.send(b'Hello, World!')

                async with await FileReadStream.from_path(path) as stream:
                    async for chunk in stream:
                        print(chunk.decode(), end='')

                print()

            run(main)

    .. tab:: 英文

        File streams read from or write to files on the file system. They can be useful for
        substituting a file for another source of data, or writing output to a file for logging
        or debugging purposes.

        Example::

            from anyio import run
            from anyio.streams.file import FileReadStream, FileWriteStream


            async def main():
                path = '/tmp/testfile'
                async with await FileWriteStream.from_path(path) as stream:
                    await stream.send(b'Hello, World!')

                async with await FileReadStream.from_path(path) as stream:
                    async for chunk in stream:
                        print(chunk.decode(), end='')

                print()

            run(main)

.. versionadded:: 3.0


.. _TLS:

TLS 流
-----------

**TLS streams**

.. tabs::

    .. tab:: 中文

        TLS(传输层安全性), 是SSL(安全套接字层)的继任者, 是在AnyIO中为TCP流提供身份验证和保密性的支持方式。

        TLS通常在连接建立后立即进行。握手过程包括以下步骤：

        * 将证书发送给对端(通常仅由服务器进行)
        * 将对端的证书与受信任的CA证书进行验证
        * 检查对端主机名是否与证书匹配

    .. tab:: 英文

        TLS (Transport Layer Security), the successor to SSL (Secure Sockets Layer), is the
        supported way of providing authenticity and confidentiality for TCP streams in AnyIO.

        TLS is typically established right after the connection has been made. The handshake
        involves the following steps:

        * Sending the certificate to the peer (usually just by the server)
        * Checking the peer certificate(s) against trusted CA certificates
        * Checking that the peer host name matches the certificate

获取服务器证书
******************************

**Obtaining a server certificate**

.. tabs::

    .. tab:: 中文

        获取X.509证书用于服务器的主要方式有三种：

        #. 创建一个自签名证书
        #. 使用 certbot_ 或类似软件从 `Let's Encrypt`_ 自动获取证书
        #. 从证书供应商处购买一个证书

        第一种选项可能是最简单的, 但这要求任何连接到您的服务器的客户端将自签名证书添加到其受信任证书列表中。显然, 这在本地开发之外是不可行的, 并且在生产环境中强烈不推荐使用。

        第二种选项如今是推荐的方法, 只要您有一个可以运行certbot_或类似软件的环境, 并且能够在必要时自动替换证书为更新版本, 同时不需要额外的功能, 比如类2验证。

        第三种选项可能是您唯一有效的选择, 特别是在您对证书有特殊要求, 只有证书供应商可以满足这些要求, 或者在您的环境中自动更新证书不可行或不切实际时。

    .. tab:: 英文

        There are three principal ways you can get an X.509 certificate for your server:

        #. Create a self signed certificate
        #. Use certbot_ or a similar software to automatically obtain certificates from `Let's Encrypt`_
        #. Buy one from a certificate vendor

        The first option is probably the easiest, but this requires that any client
        connecting to your server adds the self signed certificate to their list of trusted
        certificates. This is of course impractical outside of local development and is strongly
        discouraged in production use.

        The second option is nowadays the recommended method, as long as you have an environment
        where running certbot_ or similar software can automatically replace the certificate
        with a newer one when necessary, and that you don't need any extra features like class 2
        validation.

        The third option may be your only valid choice when you have special requirements for
        the certificate that only a certificate vendor can fulfill, or that automatically
        renewing the certificates is not possible or practical in your environment.

.. _certbot: https://certbot.eff.org/
.. _Let's Encrypt: https://letsencrypt.org/

使用自签名证书
******************************

**Using self signed certificates**

.. tabs::

    .. tab:: 中文

        要为 ``localhost`` 创建一个自签名证书, 可以使用openssl_命令行工具：

        .. code-block:: bash

            openssl req -x509 -newkey rsa:2048 -subj '/CN=localhost' -keyout key.pem -out cert.pem -nodes -days 365

        这将创建一个(2048位)私有RSA密钥(``key.pem``)和一个证书(``cert.pem``), 其主机名为“localhost”。在这些设置下, 证书的有效期为一年。

        要使用这个密钥-证书对设置服务器, 可以参考以下示例::

            import ssl

            from anyio import create_tcp_listener, run
            from anyio.streams.tls import TLSListener


            async def handle(client):
                async with client:
                    name = await client.receive()
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                # 创建一个用于客户端身份验证的上下文
                context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)

                # 加载服务器证书和私钥
                context.load_cert_chain(certfile='cert.pem', keyfile='key.pem')

                # 创建监听器并开始提供连接服务
                listener = TLSListener(await create_tcp_listener(local_port=1234), context)
                await listener.serve(handle)

            run(main)

        然后, 可以使用以下方式连接到此服务器::

            import ssl

            from anyio import connect_tcp, run


            async def main():
                # 如果证书没有被您计算机上安装的CA证书信任, 则需要以下两个步骤；
                # 如果您使用Let's Encrypt或商业证书供应商, 则可以跳过这部分
                context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
                context.load_verify_locations(cafile='cert.pem')

                async with await connect_tcp('localhost', 1234, ssl_context=context) as client:
                    await client.send(b'Client\n')
                    response = await client.receive()
                    print(response)

            run(main)

    .. tab:: 英文

        To create a self signed certificate for ``localhost``, you can use the openssl_ command
        line tool:

        .. code-block:: bash

            openssl req -x509 -newkey rsa:2048 -subj '/CN=localhost' -keyout key.pem -out cert.pem -nodes -days 365

        This creates a (2048 bit) private RSA key (``key.pem``) and a certificate (``cert.pem``)
        matching the host name "localhost". The certificate will be valid for one year with
        these settings.

        To set up a server using this key-certificate pair::

            import ssl

            from anyio import create_tcp_listener, run
            from anyio.streams.tls import TLSListener


            async def handle(client):
                async with client:
                    name = await client.receive()
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                # Create a context for the purpose of authenticating clients
                context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)

                # Load the server certificate and private key
                context.load_cert_chain(certfile='cert.pem', keyfile='key.pem')

                # Create the listener and start serving connections
                listener = TLSListener(await create_tcp_listener(local_port=1234), context)
                await listener.serve(handle)

            run(main)

        Connecting to this server can then be done as follows::

            import ssl

            from anyio import connect_tcp, run


            async def main():
                # These two steps are only required for certificates that are not trusted by the
                # installed CA certificates on your machine, so you can skip this part if you
                # use Let's Encrypt or a commercial certificate vendor
                context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
                context.load_verify_locations(cafile='cert.pem')

                async with await connect_tcp('localhost', 1234, ssl_context=context) as client:
                    await client.send(b'Client\n')
                    response = await client.receive()
                    print(response)

            run(main)

.. _openssl: https://www.openssl.org/

动态创建自签名证书
********************************************

**Creating self-signed certificates on the fly**

.. tabs::

    .. tab:: 中文

        在测试启用了TLS的服务时, 动态生成证书会更加方便。为此, 您可以使用 trustme_ 库::

            import ssl

            import pytest
            import trustme


            @pytest.fixture(scope='session')
            def ca():
                return trustme.CA()


            @pytest.fixture(scope='session')
            def server_context(ca):
                server_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
                ca.issue_cert('localhost').configure_cert(server_context)
                return server_context


            @pytest.fixture(scope='session')
            def client_context(ca):
                client_context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
                ca.configure_trust(client_context)
                return client_context

        然后, 您可以将上述fixture中的服务器和客户端上下文传递给 :class:`~.streams.tls.TLSListener`、:meth:`~.streams.tls.TLSStream.wrap` 或您在任一端使用的任何方法。

    .. tab:: 英文

        When testing your TLS enabled service, it would be convenient to generate the
        certificates on the fly. To this end, you can use the trustme_ library::

            import ssl

            import pytest
            import trustme


            @pytest.fixture(scope='session')
            def ca():
                return trustme.CA()


            @pytest.fixture(scope='session')
            def server_context(ca):
                server_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
                ca.issue_cert('localhost').configure_cert(server_context)
                return server_context


            @pytest.fixture(scope='session')
            def client_context(ca):
                client_context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
                ca.configure_trust(client_context)
                return client_context

        You can then pass the server and client contexts from the above fixtures to
        :class:`~.streams.tls.TLSListener`, :meth:`~.streams.tls.TLSStream.wrap` or whatever you
        use on either side.

.. _trustme: https://pypi.org/project/trustme/

处理不规则的 EOF
************************

**Dealing with ragged EOFs**

.. tabs::

    .. tab:: 中文

        根据 `TLS标准 <TLS standard>`_ , 加密连接应该以关闭握手结束。此做法可以防止所谓的 `截断攻击 <truncation attacks>`_ 。然而, 广泛使用的协议实现(如HTTP)通常忽略这一要求, 因为协议级别的关闭信号会使关闭握手变得多余。

        AnyIO 默认遵循此标准(不同于Python标准库的 :mod:`ssl` 模块)。这一实践的实际含义是, 如果您正在实现一个预期跳过TLS关闭握手的协议, 您需要在 :meth:`~.streams.tls.TLSStream.wrap` 或 :class:`~.streams.tls.TLSListener` 中传递 ``standard_compatible=False`` 选项。

    .. tab:: 英文

        According to the `TLS standard`_, encrypted connections should end with a closing
        handshake. This practice prevents so-called `truncation attacks`_. However, broadly
        available implementations for protocols such as HTTP, widely ignore this requirement
        because the protocol level closing signal would make the shutdown handshake redundant.

        AnyIO follows the standard by default (unlike the Python standard library's :mod:`ssl`
        module). The practical implication of this is that if you're implementing a protocol
        that is expected to skip the TLS closing handshake, you need to pass the
        ``standard_compatible=False`` option to :meth:`~.streams.tls.TLSStream.wrap` or
        :class:`~.streams.tls.TLSListener`.

.. _TLS standard: https://tools.ietf.org/html/draft-ietf-tls-tls13-28
.. _truncation attacks: https://en.wikipedia.org/wiki/Transport_Layer_Security
   #Attacks_against_TLS/SSL
