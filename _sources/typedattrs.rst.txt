使用类型化属性 
======================

**Using typed attributes**

.. py:currentmodule:: anyio

.. tabs::

    .. tab:: 中文

        在 AnyIO 中, 流和监听器可以相互叠加以提供额外的功能。但是, 当你想从下层的某个层次查找信息时, 可能需要遍历整个链条才能找到你需要的内容, 这非常不方便。为了解决这个问题, AnyIO 提供了一个 *类型化属性* 系统, 你可以通过唯一的键查找特定的属性。如果一个流或监听器包装器没有你要找的属性, 它将会在被包装的实例中查找, 那个包装器可以继续在它所包装的实例中查找, 直到找到该属性或者链条的末端。这也使得包装器可以在必要时覆盖被包装对象的属性。

        一个常见的使用场景是查找TCP连接远程端的IP地址, 这时流可能是 :class:`~.abc.SocketStream` 或 :class:`~.streams.tls.TLSStream`::

            from anyio import connect_tcp
            from anyio.abc import SocketAttribute


            async def connect(host, port, tls: bool):
                stream = await connect_tcp(host, port, tls=tls)
                print('Connected to', stream.extra(SocketAttribute.remote_address))

        每个类型化属性提供者类应该自行文档化它提供的属性集合。

    .. tab:: 英文

        On AnyIO, streams and listeners can be layered on top of each other to provide extra
        functionality. But when you want to look up information from one of the layers down
        below, you might have to traverse the entire chain to find what you're looking for,
        which is highly inconvenient. To address this, AnyIO has a system of *typed attributes*
        where you can look for a specific attribute by its unique key. If a stream or listener
        wrapper does not have the attribute you're looking for, it will look it up in the
        wrapped instance, and that wrapper can look in its wrapped instance and so on, until the
        attribute is either found or the end of the chain is reached. This also lets wrappers
        override attributes from the wrapped objects when necessary.

        A common use case is finding the IP address of the remote side of a TCP connection when
        the stream may be either :class:`~.abc.SocketStream` or
        :class:`~.streams.tls.TLSStream`::

            from anyio import connect_tcp
            from anyio.abc import SocketAttribute


            async def connect(host, port, tls: bool):
                stream = await connect_tcp(host, port, tls=tls)
                print('Connected to', stream.extra(SocketAttribute.remote_address))

        Each typed attribute provider class should document the set of attributes it provides on
        its own.

定义您自己的类型化属性 
----------------------------------

**Defining your own typed attributes**

.. tabs::

    .. tab:: 中文

        根据约定，类型化属性会与同一类别的其他属性一起存储在一个容器类中::

            from anyio import TypedAttributeSet, typed_attribute


            class MyTypedAttribute(TypedAttributeSet):
                string_valued_attribute: str = typed_attribute()
                some_float_attribute: float = typed_attribute()

        要为这些属性提供值，请在你的类中实现 :meth:`~.TypedAttributeProvider.extra_attributes` 属性::

            from collections.abc import Callable, Mapping

            from anyio import TypedAttributeProvider


            class MyAttributeProvider(TypedAttributeProvider):
                @property
                def extra_attributes() -> Mapping[Any, Callable[[], Any]]:
                    return {
                        MyTypedAttribute.string_valued_attribute: lambda: 'my attribute value',
                        MyTypedAttribute.some_float_attribute: lambda: 6.492
                    }

        如果你的类继承自另一个类型化属性提供者，请确保将其属性包含在返回值中::

            class AnotherAttributeProvider(MyAttributeProvider):
                @property
                def extra_attributes() -> Mapping[Any, Callable[[], Any]]:
                    return {
                        **super().extra_attributes,
                        MyTypedAttribute.string_valued_attribute: lambda: 'overridden attribute value'
                    }

    .. tab:: 英文

        By convention, typed attributes are stored together in a container class with other
        attributes of the same category::

            from anyio import TypedAttributeSet, typed_attribute


            class MyTypedAttribute(TypedAttributeSet):
                string_valued_attribute: str = typed_attribute()
                some_float_attribute: float = typed_attribute()

        To provide values for these attributes, implement the
        :meth:`~.TypedAttributeProvider.extra_attributes` property in your class::

            from collections.abc import Callable, Mapping

            from anyio import TypedAttributeProvider


            class MyAttributeProvider(TypedAttributeProvider):
                @property
                def extra_attributes() -> Mapping[Any, Callable[[], Any]]:
                    return {
                        MyTypedAttribute.string_valued_attribute: lambda: 'my attribute value',
                        MyTypedAttribute.some_float_attribute: lambda: 6.492
                    }

        If your class inherits from another typed attribute provider, make sure you include its
        attributes in the return value::

            class AnotherAttributeProvider(MyAttributeProvider):
                @property
                def extra_attributes() -> Mapping[Any, Callable[[], Any]]:
                    return {
                        **super().extra_attributes,
                        MyTypedAttribute.string_valued_attribute: lambda: 'overridden attribute value'
                    }
