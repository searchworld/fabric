# 1. Introduction
## 1.1 企业应用需要考虑几个问题
- identified 参与者身份可识别 
- permissioned 网络必须是可允许的
- performance 高交易吞吐量
- latency 低交易确认延时
- privacy 交易数据的隐私和保密性



## 1.2 Fabric
Fabric从一开始就是设计给企业使用，open source enterprise-grade permissioned distributed ledger technology (DLT) platform, designed for use in enterprise contexts

高度模块化和配置化的架构可以支撑多种企业级应用，第一个支持使用通用编程语言编写智能合约，降低开发成本；Fabric平台是permissioned，即参与者互相认识，不完全互相信任，可以在法律约定的前提下建立对应的信任关系。

和其他平台最大的一个不同是支持可插拔的一共识协议(pluggable consensus protocols)，可以根据不同的信任模型和使用场景进行选择。crash fault-tolerant (CFT) 和 byzantine fault tolerant (BFT)，前者可以用于单独一个企业或者一个可信任的当局，性能比较高；后者用于在多方参与的分布式应用。Fabric可以使用没有native cryptocurrency的一致性协议，降低成本

## 1.3 Modularity
有以下几个模块：
- ordering service，按交易的顺序建立共识，并将block传播到别的peer上
- membership service provider，建立参与者和加密实体的关联关系
- peer-to-peer gossip service，可选，传播ordering service产生的block
- Smart contracts (“chaincode”)，在容器中隔离运行，可以使用通用编程语言，但是没法直接访问ledger state
- Ledger可以配置支持多种DBMS
- endorsement and validation policy enforcement，每个应用可以单独配置

## 1.4 Permissioned vs Permissionless Blockchains
permissionless blockchain上任何人都可以参与，并且都是匿名，除了区块链本身状态不可变外其他都是不可信任的，基于“proof of work” (PoW)的拜占庭共识协议来进行挖矿

Permissioned blockchains具有一定程度的信任，不需要使用需要挖矿的共识协议。而且在这种场景下参与者引入有害代码的可能性大大降低，因为所有操作都是可以追溯到被授权的个体上。

## 1.5 Smart Contracts
智能合约，也称为chaincode，可信任的分布式应用，从区块链和底层的共识算法获取信任，是一个区块链应用的业务逻辑。使用智能合约的时候有几个关键点：
- 网络中多个智能合约并行运行
- 可以被任何人在许多场景下动态部署
- 应用代码需要被认为不可信任的，甚至可能是有害的(malicious)

现有的大多数支持智能合约的区块链平台采用一种order-execute架构，即共识算法
- 对交易进行校验和排序，然后传播到其他peer
- 其他peer按顺序执行交易

这种架构下的智能合约必须是确定性的，不然共识可能永远无法达成，因此很多平台都要求使用DSL去除不确定性，提高了学习成本。另外每个peer按顺序执行交易限制了性能和规模；同时需要复杂的措施保证平台不会被有害的代码搞垮。

## 1.6 A New Approach
Fabric引入了execute-order-validate的交易架构：
- 执行交易，检查正确性，决定是否采用
- 通过可插拔的共识协议进行排序
- 在提交到Ledger前使用每个应用自己配置的endorsement policy对交易进行校验

和order-execute最根本的区别在于对顺序达成一致之前就已经执行。

application-specific endorsement policy 用于指定哪些节点或者其中多少节点来保证一个合约的正确性，因此只要所有节点的一个子集来执行这个合约。并行执行提高了整体性能和系统规模，而且在第一步就去除了不确定性，在order前就可以将不一致的结果去掉，因此也就可以采用通用的编程语言。

## 1.7 Privacy and Confidentiality
permissionless blockchain中所有数据对所有人都是可见的，但是企业应用中隐私和保密非常重要，比如在一个供应链合作者的网络中，有些消费者可能会被给与优惠的价格，如果所有人都知道合同和价格，那所有人都会要优惠价格。
区块链平台采用多种方式来实现，每种实现方式都有取舍：
- 数据加密。但是由于数据在每个节点上都有，给与足够时间还是有可能被解密
- Zero knowledge proofs (ZKP)，需要比较多的时间和计算资源
- 在permissioned context现在保密信息的分布，只有授权的节点才能获取

Fabric使用channel架构来实现隐私和保密。参与者和网络中其他被授予某些交易可见性的的参与者建立channel，只有参与到channel中的节点才能访问智能合约和交易数据

## 1.8 Pluggable Consensus
交易的顺序代理给一个共识的模块组件，逻辑上执行交易/维护账本的节点解耦，也就是ordering service。Fabric提供了一个使用kafka和zookeeper实现的CFT ordering service，未来会提供一个Raft实现版本和一个完全分布式的BFT ordering service
