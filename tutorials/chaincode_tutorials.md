# chaincode指南

## 什么是Chaincode

Chaincode是一个应用程序，可以用Go、node.js、或者Java来编写。它实现了一组约定的接口。Chaincode运行在一个安全的Docker容器里，并且与背书的peer节点进程隔离。Chaincode通过应用提交的交易，来初始化和管理账本的状态。

Chaincode典型应用场景是用来处理一个网络内部成员达成共识的业务逻辑，因此可以被看作是大家熟知的"智能合约"。Chaincode自身产生的状态，只能在该chaincode内部才可以访问，不能直接被其他的chaincode访问。然而，同一个网络下，如果授予适当的权限，一个chaincode是可以调用另外一个chaincode来访问被调用chaincode的状态的。

## 两种视角角色

关于chaicode，我们通常提供两种不同的视角。一个视角是从开发区块链应用和解决方案的应用开发者来看，我们称之为Chaincode开发者视角；一个视角是Chaincode运营者视角，运营者负责管理区块链网络，通过超级账本API来安装、初始化、和升级chaincode，运营者并不参与chaincode应用的开发。





