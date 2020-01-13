## Punting数据包(Punting Packets)
### 概述
“Punt”对不同的人可能有不同的含义。在VPP中，Punt是指数据平面（Data-Plane）的任何节点都无法处理数据包时所采取的动作。与drop不动的是，VPP为系统的其他元素提供了处理此数据包的机会。

Punt的普遍含义是将数据包发送到用户/控制平面。这是上述更一般情况的特定选项，其中VPP将数据包移交给控制平面以进行进一步处理。

### Punt基础设施
异常数据包是给定节点无法通过常规机制处理的那些数据包。Punt异常数据包是通过VLIB的“Punt基础设施”处理的。有两种类型的节点：来源（sources）和接收器（sinks）。在加载时，来源从基础架构分配了一个Punt“原因”。当它在交换期间遇到这种异常时，它将使用这个原因标记数据包，并发送数据包到Punt调度节点。在加载时，接收器向基础设施注册Punt处理器，因此它可以接收由于该原因而被Punt的数据包。如果由于给定的原因没有注册接收器，则丢弃数据包，如果注册了多个接收器，则复制数据包。

这种机制允许我们扩展系统以处理源节点否则会丢弃的数据包。

### Punt数据包到控制平面
#### 主动Punt
用户/控制平面指定这是我要接收的数据包类型，这是我要发送的数据包的位置。

当前存在如下3种方法来描述如何对要Punt的数据包进行匹配/分类:

1. 匹配UDP端口
2. 匹配IP协议 (例如OSPF协议)
3. 匹配到Punt原因 (如上文所述)

根据Punt的数据包的类型/分类，该主动Punt将自己注册到VLIB图中以接收那些数据包。例如，如果它是与UDP端口匹配的数据包，则它将挂接到UDP端口分派功能: udp_register_port()。

对于被动Punt，只有一个接收器，一个unix domain套接字。但是，这一领域正在开展更多工作。

请参阅API：vnet/ip/punt.api

#### 被动Punt
VPP对输入数据包处理可以描述为一系列分类器。例如，一系列输入分类器：是IP吗? 是给我们的吗? 是UDP吗? 它是已知的UDP端口吗? 如果在此管道(Pipeline)中的某个时刻VPP没有相关分类，则可以对数据包进行Punt处理，这意味着将其发送到ipX-punt节点。这被描述为无源的，因为控制平面正在接收VPP本身不处理的数据包。对于被动Punt，用户可以指定将数据包发送到何处，以被管理（whether/how they should be policed）和限制线速（rate-limited）。