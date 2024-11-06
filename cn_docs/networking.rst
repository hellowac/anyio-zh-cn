使用套接字和流
=========================

**Using sockets and streams**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        网络功能可以说是任何异步库中最重要的部分。AnyIO 在其支持的每个后端提供的低级原语之上, 包含了自己实现的高层网络功能。

        目前, AnyIO 提供以下网络功能：

        * TCP 套接字(客户端 + 服务器)
        * UNIX 域套接字(客户端 + 服务器)
        * UDP 套接字
        * UNIX 数据报套接字

        目前尚不支持更为特殊的网络形式, 如原始套接字和 SCTP。

        .. warning:: 与标准 BSD 套接字接口和大多数其他网络库不同, AnyIO(从 2.0 版本起)通过引发 :exc:`~EndOfStream` 异常来标示任何流的结束, 而不是返回一个空的字节对象。

    .. tab:: 英文

        Networking capabilities are arguably the most important part of any asynchronous
        library. AnyIO contains its own high level implementation of networking on top of low
        level primitives offered by each of its supported backends.

        Currently AnyIO offers the following networking functionality:

        * TCP sockets (client + server)
        * UNIX domain sockets (client + server)
        * UDP sockets
        * UNIX datagram sockets

        More exotic forms of networking such as raw sockets and SCTP are currently not
        supported.

        .. warning:: Unlike the standard BSD sockets interface and most other networking
            libraries, AnyIO (from 2.0 onwards) signals the end of any stream by raising the
            :exc:`~EndOfStream` exception instead of returning an empty bytes object.

使用 TCP 套接字
------------------------

**Working with TCP sockets**

.. tabs::

    .. tab:: 中文

        TCP(传输控制协议)是互联网上最常用的协议。它允许连接到远程主机的端口, 并以可靠的方式发送和接收数据。

        要连接到某个地方的监听 TCP 套接字, 可以使用 :func:`~connect_tcp`::

            from anyio import connect_tcp, run


            async def main():
                async with await connect_tcp('hostname', 1234) as client:
                    await client.send(b'Client\n')
                    response = await client.receive()
                    print(response)

            run(main)

        为了方便, 您还可以使用 :func:`~connect_tcp` 在连接后与对端建立 TLS 会话, 方法是传递 ``tls=True``, 或为 ``ssl_context`` 或 ``tls_hostname`` 传递一个非空值。

        要接收传入的 TCP 连接, 首先使用 :func:`create_tcp_listener` 创建一个 TCP 监听器, 并在其上调用 :meth:`~.abc.Listener.serve`::

            from anyio import create_tcp_listener, run


            async def handle(client):
                async with client:
                    name = await client.receive(1024)
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                listener = await create_tcp_listener(local_port=1234)
                await listener.serve(handle)

            run(main)

        有关更多信息, 请参阅 :ref:`TLS` 部分。

    .. tab:: 英文

        TCP (Transmission Control Protocol) is the most commonly used protocol on the Internet.
        It allows one to connect to a port on a remote host and send and receive data in a
        reliable manner.

        To connect to a listening TCP socket somewhere, you can use :func:`~connect_tcp`::

            from anyio import connect_tcp, run


            async def main():
                async with await connect_tcp('hostname', 1234) as client:
                    await client.send(b'Client\n')
                    response = await client.receive()
                    print(response)

            run(main)

        As a convenience, you can also use :func:`~connect_tcp` to establish a TLS session with
        the peer after connection, by passing ``tls=True`` or by passing a nonempty value for
        either ``ssl_context`` or ``tls_hostname``.

        To receive incoming TCP connections, you first create a TCP listener with
        :func:`create_tcp_listener` and call :meth:`~.abc.Listener.serve` on it::

            from anyio import create_tcp_listener, run


            async def handle(client):
                async with client:
                    name = await client.receive(1024)
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                listener = await create_tcp_listener(local_port=1234)
                await listener.serve(handle)

            run(main)

        See the section on :ref:`TLS` for more information.

使用 UNIX 套接字
-------------------------

**Working with UNIX sockets**

.. tabs::

    .. tab:: 中文

        UNIX 域套接字是 UNIX 类操作系统上的一种进程间通信形式。它们不能用于连接到远程主机, 并且在 Windows 上无法使用。

        UNIX 域套接字的 API 与 TCP 套接字的 API 类似, 不同之处在于, 它使用的是文件系统路径, 而不是主机/端口组合。

        这是从 TCP 示例转换为使用 UNIX 套接字的客户端代码::

            from anyio import connect_unix, run


            async def main():
                async with await connect_unix('/tmp/mysock') as client:
                    await client.send(b'Client\n')
                    response = await client.receive(1024)
                    print(response)

            run(main)

        监听器代码如下::

            from anyio import create_unix_listener, run


            async def handle(client):
                async with client:
                    name = await client.receive(1024)
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                listener = await create_unix_listener('/tmp/mysock')
                await listener.serve(handle)

            run(main)

        .. note:: UNIX 套接字监听器不会删除它创建的套接字, 因此您可能需要手动删除它们。

    .. tab:: 英文

        UNIX domain sockets are a form of interprocess communication on UNIX-like operating
        systems. They cannot be used to connect to remote hosts and do not work on Windows.

        The API for UNIX domain sockets is much like the one for TCP sockets, except that
        instead of host/port combinations, you use file system paths.

        This is what the client from the TCP example looks like when converted to use UNIX
        sockets::

            from anyio import connect_unix, run


            async def main():
                async with await connect_unix('/tmp/mysock') as client:
                    await client.send(b'Client\n')
                    response = await client.receive(1024)
                    print(response)

            run(main)

        And the listener::

            from anyio import create_unix_listener, run


            async def handle(client):
                async with client:
                    name = await client.receive(1024)
                    await client.send(b'Hello, %s\n' % name)


            async def main():
                listener = await create_unix_listener('/tmp/mysock')
                await listener.serve(handle)

            run(main)

        .. note:: The UNIX socket listener does not remove the socket it creates, so you may
        need to delete them manually.

发送和接收文件描述符
++++++++++++++++++++++++++++++++++++++

**Sending and receiving file descriptors**

.. tabs::

    .. tab:: 中文

        UNIX 套接字可以用于将打开的文件描述符(套接字和文件)传递给另一个进程。接收端可以使用 `os.fdopen` 或 `socket.socket` 来分别获取可用的文件或套接字对象。

        以下是一个示例, 客户端连接到 UNIX 套接字服务器并接收服务器上打开的文件的描述符, 读取文件内容, 然后将其打印到标准输出。

        客户端代码::

            import os

            from anyio import connect_unix, run


            async def main():
                async with await connect_unix('/tmp/mysock') as client:
                    _, fds = await client.receive_fds(0, 1)
                    with os.fdopen(fds[0]) as file:
                        print(file.read())

            run(main)

        服务器代码::

            from pathlib import Path

            from anyio import create_unix_listener, run


            async def handle(client):
                async with client:
                    with path.open('r') as file:
                        await client.send_fds(b'this message is ignored', [file])


            async def main():
                listener = await create_unix_listener('/tmp/mysock')
                await listener.serve(handle)

            path = Path('/tmp/examplefile')
            path.write_text('Test file')
            run(main)

    .. tab:: 英文

        UNIX sockets can be used to pass open file descriptors (sockets and files) to another
        process. The receiving end can then use either :func:`os.fdopen` or
        :class:`socket.socket` to get a usable file or socket object, respectively.

        The following is an example where a client connects to a UNIX socket server and receives
        the descriptor of a file opened on the server, reads the contents of the file and then
        prints them on standard output.

        Client::

            import os

            from anyio import connect_unix, run


            async def main():
                async with await connect_unix('/tmp/mysock') as client:
                    _, fds = await client.receive_fds(0, 1)
                    with os.fdopen(fds[0]) as file:
                        print(file.read())

            run(main)

        Server::

            from pathlib import Path

            from anyio import create_unix_listener, run


            async def handle(client):
                async with client:
                    with path.open('r') as file:
                        await client.send_fds(b'this message is ignored', [file])


            async def main():
                listener = await create_unix_listener('/tmp/mysock')
                await listener.serve(handle)

            path = Path('/tmp/examplefile')
            path.write_text('Test file')
            run(main)

使用 UDP 套接字
------------------------

**Working with UDP sockets**

.. tabs::

    .. tab:: 中文

        UDP(用户数据报协议)是一种通过网络发送数据包的方式, 不具有连接、重试或错误纠正等特性。

        例如, 如果你想创建一个 UDP "hello" 服务, 该服务只读取一个数据包, 然后向发送方发送一个数据包, 内容前面加上 "Hello, ", 你可以这样做::

            import socket

            from anyio import create_udp_socket, run


            async def main():
                async with await create_udp_socket(
                    family=socket.AF_INET, local_port=1234
                ) as udp:
                    async for packet, (host, port) in udp:
                        await udp.sendto(b'Hello, ' + packet, host, port)

            run(main)

        .. note:: 如果你在本地机器上测试, 或者不知道使用哪个协议族, 可以考虑在上述示例中将 ``family=socket.AF_INET`` 替换为 ``local_host='localhost'`` 。

        如果你的用例涉及向单个目的地发送大量数据包, 你仍然可以将 UDP 套接字“连接”到特定的主机和端口, 以避免每次发送数据时都需要传递地址和端口::

            from anyio import create_connected_udp_socket, run


            async def main():
                async with await create_connected_udp_socket(
                        remote_host='hostname', remote_port=1234) as udp:
                    await udp.send(b'Hi there!\n')

            run(main)

    .. tab:: 英文

        UDP (User Datagram Protocol) is a way of sending packets over the network without
        features like connections, retries or error correction.

        For example, if you wanted to create a UDP "hello" service that just reads a packet and
        then sends a packet to the sender with the contents prepended with "Hello, ", you would
        do this::

            import socket

            from anyio import create_udp_socket, run


            async def main():
                async with await create_udp_socket(
                    family=socket.AF_INET, local_port=1234
                ) as udp:
                    async for packet, (host, port) in udp:
                        await udp.sendto(b'Hello, ' + packet, host, port)

            run(main)

        .. note:: If you are testing on your local machine or don't know which family socket to
        use, it is a good idea to replace ``family=socket.AF_INET`` by
        ``local_host='localhost'`` in the previous example.

        If your use case involves sending lots of packets to a single destination, you can still
        "connect" your UDP socket to a specific host and port to avoid having to pass the
        address and port every time you send data to the peer::

            from anyio import create_connected_udp_socket, run


            async def main():
                async with await create_connected_udp_socket(
                        remote_host='hostname', remote_port=1234) as udp:
                    await udp.send(b'Hi there!\n')

            run(main)

使用 UNIX 数据报套接字
----------------------------------

**Working with UNIX datagram sockets**

.. tabs::

    .. tab:: 中文

        UNIX 数据报套接字是 UNIX 域套接字的一个子集, 区别在于, 虽然 UNIX 套接字实现了可靠的连续字节流通信(类似于 TCP), 但 UNIX 数据报套接字实现了数据包的通信(类似于 UDP)。

        UNIX 数据报套接字的 API 模式类似于 UDP 套接字, 只是将主机/端口组合替换为文件系统路径——下面是使用 UNIX 数据报套接字编写的 UDP "hello" 服务示例::

            from anyio import create_unix_datagram_socket, run


            async def main():
                async with await create_unix_datagram_socket(
                    local_path='/tmp/mysock'
                ) as unix_dg:
                    async for packet, path in unix_dg:
                        await unix_dg.sendto(b'Hello, ' + packet, path)

            run(main)

        .. note:: 如果没有设置 ``local_path``, UNIX 数据报套接字将绑定到一个没有名称的地址, 通常无法从其他 UNIX 数据报套接字接收数据报。

        类似于 UDP 套接字, 如果你的用例涉及向单个目的地发送大量数据包, 你可以将 UNIX 数据报套接字“连接”到特定路径, 以避免每次发送数据时都需要传递路径::

            from anyio import create_connected_unix_datagram_socket, run


            async def main():
                async with await create_connected_unix_datagram_socket(
                    remote_path='/dev/log'
                ) as unix_dg:
                    await unix_dg.send(b'Hi there!\n')

            run(main)

    .. tab:: 英文

        UNIX datagram sockets are a subset of UNIX domain sockets, with the difference being
        that while UNIX sockets implement reliable communication of a continuous byte stream
        (similarly to TCP), UNIX datagram sockets implement communication of data packets
        (similarly to UDP).

        The API for UNIX datagram sockets is modeled after the one for UDP sockets, except that
        instead of host/port combinations, you use file system paths - here is the UDP "hello"
        service example written with UNIX datagram sockets::

            from anyio import create_unix_datagram_socket, run


            async def main():
                async with await create_unix_datagram_socket(
                    local_path='/tmp/mysock'
                ) as unix_dg:
                    async for packet, path in unix_dg:
                        await unix_dg.sendto(b'Hello, ' + packet, path)

            run(main)


        .. note:: If ``local_path`` is not set, the UNIX datagram socket will be bound on an
        unnamed address, and will generally not be able to receive datagrams from other UNIX
        datagram sockets.

        Similarly to UDP sockets, if your case involves sending lots of packets to a single
        destination, you can "connect" your UNIX datagram socket to a specific path to avoid
        having to pass the path every time you send data to the peer::

            from anyio import create_connected_unix_datagram_socket, run


            async def main():
                async with await create_connected_unix_datagram_socket(
                    remote_path='/dev/log'
                ) as unix_dg:
                    await unix_dg.send(b'Hi there!\n')

            run(main)
