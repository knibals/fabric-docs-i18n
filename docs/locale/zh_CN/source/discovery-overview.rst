服务发现
=================

为什么我们需要服务发现?
---------------------------------

为了在 Peer 节点上执行链码，将交易提交给排序节点，并更新交易状态，应用程序需要连接 SDK 中开放的 API。

然而，为了让应用程序连接到相关的网络节点，SDK 需要大量信息。除了通道上排序节点和 Peer 节点的 CA 证书和 TLS 证书及其 IP 地址和端口号之外，它还必须知道相关的背书策略以及安装了链码的 Peer 节点（这样应用程序才能知道将链码提案发送给哪个 Peer 节点）。

在 v1.2 之前，这些信息是静态编码的。然而，这种实现不能动态响应网络的更改（例如添加已安装相关链码的 Peer 节点或 Peer 节点临时离线）。静态配置也不能让应用程序对背书策略本身的更改做出反应（比如通道中加入了新的组织）。

此外，客户端应用程序无法知道哪些 Peer 节点已更新账本，哪些未更新。因此该应用程序可能向网络中尚未同步账本的 Peer 节点提交提案，从而导致事务在提交时失效并浪费资源。

**服务发现** 通过让 Peer 节点动态计算所需信息，并以可消耗的方式将其提供给 SDK，从而改善了此过程。

Fabric 中的服务发现是如何工作的
---------------------------------------------------

应用程序启动时会通过设置得知应用程序开发人员或者管理员所信任的一组 Peer 节点，以便为发现查询提供可信的响应。客户端应用程序所使用的候选节点也在该组织中。请注意，为了被服务发现所识别，Peer 节点必须定义一个 ``EXTERNAL_ENDPOINT`` 。想了解如何执行此操作，请查看我们的文档 :doc:`discovery-cli` 

应用程序向发现服务发出配置查询并获取它所拥有的所有静态配置，否则就需要与网络的其余节点通信。继续向 Peer 节点的发现服务发送查询，就可以随时刷新此信息。

该服务在 Peer 节点（而不是应用程序）上运行，并使用 gossip 通信层维护的网络元数据信息来找出哪些 Peer 节点在线。它还从 Peer 节点的状态数据库中获取信息，例如相关的签名策略等。

通过服务发现，应用程序不再需要指定为他们背书的 Peer 节点。给定通道和链码 ID，SDK 就可以通过发现服务查询其所对应的 Peer 节点。然后，发现服务会计算出由两个对象组成的描述符：

1. **布局（Layouts）**: Peer 组的列表，以及每个组中应当选取的 Peer 节点数。
2. **组到 Peer 的映射**: 布局中的组和通道的 Peer 节点的对应。实际上，一个组很可能是代表一个组织的 Peer 节点，但是由于服务 API 是通用的并且与组织无关，因此这只是一个“组”。

下面是一个``AND(Org1, Org2)``策略的描述符示例，其中每个组织中有两个 Peer 节点。

The following is an example of a descriptor from the evaluation of a policy of
``AND(Org1, Org2)`` where there are two peers in each of the organizations.

.. code-block:: none

   Layouts: [
        QuantitiesByGroup: {
          “Org1”: 1,
          “Org2”: 1,
        }
   ],
   EndorsersByGroups: {
     “Org1”: [peer0.org1, peer1.org1],
     “Org2”: [peer0.org2, peer1.org2]
   }

换言之，背书策略需要两个分别来自 Org1 和 Org2 的 Peer 节点的签名。它提供了组织中可以背书的（Org1 和 Org2 中都有 ``peer0`` 和 ``peer1``） Peer 节点的名称。

之后 SDK 会随机从列表中选择一个布局。在上边的例子中的背书策略是 Org1 ``AND`` Org2。如果换成 ``OR`` 策略，SDK 就会随机选择 Org1 或 Org2，因为任意一个节点的签名都可以满足背书策略。

SDK 选择布局后，会根据客户端指定的标准对布局中的 Peer 节点进行选择（SDK 可以这样做是因为它能够访问元数据，比如账本高度）。例如，SDK 可以根据布局中每个组的 Peer 节点的数量，优先选择具有更高的账本高度的 Peer 节点，或者排除应用程序发现的处于脱机状态的 Peer 节点。如果无法根据标准选定一个 Peer 节点, SDK 将从最符合标准的 Peer 节点中随机选择。

发现服务的功能
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

发现服务可以响应以下查询：

* **配置查询**： 从通道的排序节点返回通道中所有组织的 ``MSPConfig``。
* **Peer 成员查询**： 返回已加入通道的 Peer 节点。
* **背书查询**： 返回通道中给定链码的背书描述符。
* **本地 Peer 成员查询**： 返回响应查询的 Peer 节点的本地成员信息。默认情况下，客户端需要是 Peer 节点的管理员才能响应此查询。

特殊要求
~~~~~~~~~~~~~~~~~~~~~~
当 Peer 节点在启用 TLS 的情况下运行时，客户端在连接到 Peer 节点时必须提供 TLS 证书。如果未将 Peer 节点配置为验证客户端证书（clientAuthRequired 为 false），则此 TLS 证书可以是自签名的。

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
   