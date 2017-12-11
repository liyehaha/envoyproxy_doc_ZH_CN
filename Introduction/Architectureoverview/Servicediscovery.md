### 服务发现
在配置上游群集时，Envoy需要知道解析这些群集的成员。这被称为服务发现。

#### 支持的服务发现类型

##### 静态
静态是最简单的服务发现类型。通过静态配置明确每个上游主机的网络名称（IP地址/端口，unix域套接字等）。

##### 严格（Strict）DNS
当使用DNS服务发现时，Envoy将持续并异步地解析指定的DNS目标。DNS结果中的每个返回的IP地址将被视为上游群集中的显式主机。 这意味着如果查询返回三个IP地址，Envoy将假定集群有三个主机，并且三个主机都应该负载平衡。如果主机从结果中删除，则Envoy认为它不再存在，并将从任何现有的连接池中摄取流量。请注意，Envoy绝不会在转发路径中同步解析DNS。以最终一致性为代价，永远不会担心长时间运行的DNS查询会受到阻塞。

##### 逻辑（Logical）DNS
逻辑DNS使用类似的异步机制来解析严格DNS。但是，并不是严格考虑DNS查询的结果，而是假设它们构成整个上游集群，而逻辑DNS集群仅使用在需要启动新连接时返回的第一个IP地址。因此，单个逻辑连接池可以包含到各种不同上游主机的物理连接。连接永远不会流失。此服务发现类型适用于必须通过DNS访问的大型Web服务。这种服务通常使用循环法的DNS来返回许多不同的IP地址。通常会为每个查询返回不同的结果。如果在这种情况下使用严格的DNS，Envoy会认为集群的成员在每个解决时间间隔期间都会发生变化，这会导致连接池，连接循环等消失。相反，使用逻辑DNS，连接将保持活动状态，直到它们循环。在与大型Web服务交互时，这是所有可能的世界中最好的：异步/最终一致的DNS解析，长期连接，以及转发路径中的零阻塞。

##### 服务发现服务（SDS）
Envoy通过REST API向服务发现服务，获取集群的成员。Lyft通过Python提供了一个发现服务的参考实现。该实现是使用AWS DynamoDB作为后端存储，但该API非常简单，可以轻松地在各种不同的后备存储之上实现。对于每个SDS群集，Envoy将定期从发现服务中获取群集成员。SDS做为首选的服务发现机制，原因如下：
 - Envoy能够对每个上游主机都有更加详细的信息（相比通过DNS解析），并能做出更智能的负载均衡决策。
 - 在每个主机发现的API响应中，通过携带附加的属性，如负载均衡权重，灰度状态，区域等。这些附加属性在负载均衡，统计信息收集等过程中由Envoy网络全局使用。

通常，主动健康检查与最终一致的服务发现服务结合起来使用，以进行负载平衡和路由决策。这将在下一节进一步讨论。

##### 最终一致的服务发现
许多现有的RPC系统将服务发现视为完全一致的过程。为此，他们使用支持完全一致的leader选举存储，如Zookeeper，etcd，Consul等。我们的经验是，在大规模操作这些存储会很痛苦。

Envoy从一开始就设计了服务发现不需要完全一致的想法。相反，Envoy假定主机以一种最终一致的方式来自网格。在部署服务间Envoy网格时，我们推荐使用最终一致的服务发现，结合主动健康检查（Envoy主动健康检查上游集群成员状态）来确定集群运行状况。这种范例有许多好处：

- 有的健康决定是完全分配的。 因此，网络分区被正常处理（应用程序是否正常处理分区是另一回事）。
- 为上游群集配置运行状况检查时，Envoy使用2x2矩阵来确定是否路由到主机：

<table>
    <tr>
        <th> 服务发现状态 </th>
        <th> 健康检查正常 </th>
        <th> 健康检查失败 </th>
    </tr>
    <tr>
        <th>存在</th>
        <th>路由</th>
        <th>停止路由</th>
    </tr>
    <tr>
        <th>不存在</th>
        <th>路由</th>
        <th>停止路由/删除</th>
    </tr>
</table>

- **主机被发现/健康检查确定**</br>
特使将路由到目标主机。

- **主机无法被发现/健康检查确定**</br>
Envoy将路由到目标主机。这是非常重要的，因为设计假定发现服务可以随时失败。如果主机即使在发现数据缺失之后仍继续通过健康检查，Envoy仍将路由。 虽然在这种情况下添加新主机是不可能的，但现有的主机仍然可以正常运行。 当发现服务再次正常运行时，数据将最终重新收敛。

- **主机被发现/健康检查失败**</br>
Envoy不会路由到目标主机。假设健康检查数据比发现数据更准确。

- **主机无法被发现/健康检查失败**</br>
Envoy不会路由并将删除目标主机。这是Envoy将清除主机数据的唯一状态。

### 返回
- [架构介绍](../Architectureoverview.md)
- [简介](../../Introduction.md)
- [首页目录](../../README.md)