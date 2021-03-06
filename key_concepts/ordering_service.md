# 排序服务Ordering Service

**适合阅读本文档的角色**：架构师、ordering service管理员、channel创建者

本章节，将从概念性诠释的角度，介绍ordering排序的概念、orderer节点如何与peer节点进行交互、orderer节点在交易流中所扮演的角色、目前ordering服务实现的综述，以及特别关注与Raft排序服务的实现。

## 什么是ordering排序

许多分布式区块链网络，比如比特币和以太坊，都不是许可授权网络，这意味着任何节点都可以参与共识过程，来将交易排序并打包进区块block。这种情况下，这些网络就必须依赖`probablistic概率论`的共识算法，确保账本最终能在很高概率下，保持一致性，但是依然无法避免出现分歧（这就是大家熟知的`fork分叉`），这种情况就是网络的多个参与方，对于已经写入的交易的顺序，无法达成一致。

超级账本采取了不同的方式。它引入了一种特殊的节点，称之为orderer节点或者ordering节点，该类型节点专门负责交易的排序，跟其他的节点一起，组成了ordering service排序服务。因为Fabric的共识设计依赖与确定性的共识算法上，一个peer节点验证的任何区块，都是由ordering service排序服务产生的，从而确保了最终的一致性和正确性。因此Fabric区块链网络或ledger账本，不会像其他区块链网络一样，发生fork分叉。

Fabric将chaincode执行背书从orderer排序节点里面分离出来，从而赋予了超级账本网络更高的性能和可扩展性，同时又消除了代码执行和排序发生在同一个节点上的性能瓶颈。

## 排序节点和channel配置

除了排序的角色外，排序节点也维护了一个组织列表，该列表里的组织，可以创建channel通道。这些组织organizations就是我们熟知的"consortium"工会，该列表被保存于"orderer system channel"排序节点系统通道的配置中。默认情况下，该列表和通道，只能由排序节点的管理员来编辑。注意：ordering service排序服务可以保持多个这样的列表，从而使得工会成为一个重要的工具，实现Fabric的multi-tenancy（多重租赁？）。

Orderer节点也被赋予了对于通道的基本访问控制，比如限制谁可以读写通道数据，谁可以配置通道。记住一点：当创建工会或者通道的时候，被授权的管理员是通过通道配置文件中的policies元素，来进行配置的。交易的配置是有orderer节点处理的，因为它需要知道当前的policies集合，从而进行基本的访问控制。在这种情况下，orderer节点处理配置的更新，从而确保发起者有正确的管理权力。如果是这样的话，orderer要验证对于当前配置的更新请求，产生一个新的配置更改交易，并将其打包进一个区块，该区块将传播到通道内的所有peer节点。peer节点处理收到的配置更新交易，来验证order节点通过的配置修改，是否满足通道Policies的要求。

## 排序节点和身份识别

超级账本网络中，任何与区块链网络的交互，无论是peer节点，应用，管理员还是orderer节点，都要求从他们的数字证书和`Membership Service Provider`(**MSP**)成员认证服务定义中，获取所属组织的身份识别信息。

关于身份识别和MSPs的更多信息，

请参见文档[Identity](identity.html)和[Membership](membership.html)。

像peer节点一样，orderer节点也隶属于一个组织。每一个组织都有一个单独的CA证书。可以每个组织都有一个root CA，或者统一部署一个root CA，再从该root CA衍生出CA，分配给每个组织。

## 排序节点和交易流

### 第一阶段：Proposal提案

从peer节点的章节中，我们了解到peer节点构成了超级账本网络的基础。peer节点保存账本，使得应用程序可以通过智能合约来查询和更新账本状态。

典型的情况下，应用程序对账本状态的更新要涉及到三个阶段的流程，从而确保区块链网络里所有的节点都能保持账本一致。

在第一阶段，客户端应用向部分peer节点发出一条交易提案，收到提案交易的peer节点将会调用智能和约，来产生一个账本状态更新提议，并且对结果进行背书。背书节点并不将这个提议的更新，应用到它们的账本上。相反，背书节点会返回一个提案的相应给提案的发起者-客户端应用。被背书过了的交易提案将在第二阶段被排序打包进区块，然后发不到所有的peer节点，做最后的验证，并在第三阶段被提交。

想深入了解第一阶段，请参考[Peers](https://hyperledger-fabric.readthedocs.io/en/release-1.4/peers/peers.html#phase-1-proposal)章节。

### 第二阶段：排序及打包交易进区块

在交易完成第一阶段之后，客户端应用会收到背书节点发送回来的已经背书过的交易提案的响应。接下来就要进入交易的第二阶段。

在这个阶段，客户端应用会向排序节点发送那些包含了已经背书过的交易提案响应的交易。排序服务会将这些交易打包进区块，并最终发布给通道中的所有peer节点。peer节点在第三阶段完成最终的验证，并提交交易到本地的账本里。

排序服务节点会同时收到来自不用客户端应用提交的交易。多个排序服务节点需要齐心合力提供排序服务。排序服务的工作就是将提交过来的，多批无序的交易，排列组织成一个清楚明确的序列，并且将它们打包进块。这些块将构成区块链中的区块。

一个区块中包含的交易数量，取决于通道配置中跟期望大小和最大出块时间相关的两个参数：`BatchSize`和`BatchTimeout`。区块会存入orderer排序节点的账本，并被分发给通道里所有的peer节点。如果一个peer节点碰巧不在线，或者后加入的通道，在重新连接到排序服务的节点后，将会收到它错过的那些区块，或者不连接排序节点，通过与其他peer节点的gossiple（p2p通信）来获取新的区块。我们将在第三阶段，了解peer节点如何处理区块。

![](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/orderer.diagram.1.png)

_排序节点的第一个角色就是打包提案的账本更新。在上图的例子中，应用A1将节点E1和E2背书过的交易T1，发送给排序节点O1。同时应用A2也将节点E1背书过的交易T2，发送给排序节点O1。O1将交易T1，T2，以及来自于其他应用的交易，一起打包进区块B2。从图中我们能够看出，在区块B2内部，交易的顺序是：T1，T2，T3，T4，T6，T5 - 这个顺序可能不同于orderer节点接受到交易的顺序！（上图的例子展示的是只配置了一个排序节点的简单排序服务）_

需要注明的是：既然可以有多个排序节点，在大约相同的时间，接受到多笔交易，那么交易被打包进区块的顺序，没有必要一定与排序服务接受到交易的顺序一致。重要的是：排序交易吧这些交易排列成了一个严格的顺序，而且peer节点将会按照这个顺序来验证和提交交易。

区块内交易的这种严格的排序，使得超级账本不同于其他那些同一个区块可以被打包进不同的区块，然后通过竞争来形成一条链的区块链。在超级账本里面，排序服务产生的区块是确定的终极状态。一旦交易被写入区块，它在整个账本中的位置，就是确定不可更改的。就像我们之前说过的那样，超级账本的终极状态意味着没有账本分叉 - 被验证过的交易，永远不会被回退或者丢弃。

我们也看到，合约的执行和交易的处理都由peer节点来承担，orderer排序节点只负责将背书过的交易进行打包进区块 - 排序节点并不关心交易的具体内容（通道配置的更新交易例外，这一点我们先前提到过）。

总结一下，orderer节点负责收集提案的交易，对其进行排序，并将多笔交易打包进区块，准备后面的分发。这些工作看似简单，却是一个至关重要的过程。它确保了整个账本的一致性。

### 第三阶段：验证和提交

交易流的第三阶段，涉及到区块从排序节点到peer节点的分发，以及后续在peer节点上的验证和提交写入账本。

当排序节点向所有与其连接的peer节点分发区块的时候，第三阶段就开始了。在这里，需要指明的是：不是所有peer节点都需要与orderer节点相连 - peer节点间可以通过[**gossip**](https://hyperledger-fabric.readthedocs.io/en/release-1.4/gossip.html)协议（p2p)传递区块。

每个peer节点都独立验证接收到的区块，但是是以一种无异议，确定的方式，以确保账本保持一致。具体一点，通道里的每个peer 节点都会验证区块里的每笔交易。确保交易被规定的组织节点背书过，而且背书是匹配无误的；确保交易并没有被最近提交过的交易标注为无效。无效的交易仍然会被保留在排序节点创建的区块里，但是它们会被peer节点标注为无效，对应的账本状态更新也不会写入账本。

![](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/orderer.diagram.2.png)

_排序节点的第二个角色就是将区块分发给peer节点。在上图的例子中，排序节点O1将区块B2发送给peer 节点P1和peer节点P2。Peer节点P1处理区块B2，并将其写入P1的账本L1。同时Peer节点P2处理区块B2，并将其写入P2的账本L1。一旦这个过程完成，节点P1和P2上的账本L1就被更新一致。接下来P1和P2会各自通知与其相连的应用：交易已经被处理。_

总结一下，第三阶段就是将排序服务产生的区块，保持一致性的前提下，应用到账本。交易进入区块的严格排序，使得每个peer节点都可以验证交易更新被一致的的应用到整个网络。

想深度了解第三阶段，请参考[Peers章节](peers.html)。

## 排序服务的实现

