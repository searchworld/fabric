# 4 Architecture Reference
## 4.1 Architecture Explained

## 4.2 Transaction Flow

## 4.3 Service Discovery
### Why
为了在peer上执行chaincode，提交transaction到orderer，更新交易状态，application要连接到SDK暴露的API。因此SDK需要许多新兴，比如orderer和peer的CA/TLS证书和IP端口，还有相应的endorsement策略，已经chaincode安装在哪些peer上。
在v1.2之前，这些信息是静态编码的，无法对网络变化进行动态调整（比如有新的peer按照了chaincode，或者peer临时下线），对背书策略的变化也没法动态调整（比如有新的org加入channel）。另外client application也没法知道哪些peer更新了ledger，哪些还没有，结果是application可能会提交proposal到账本数据还没有in sync的peer上，导致交易在commit的时候无效，浪费资源。
discovery service让peer动态计算需要的信息，并以可消费的方式提供给SDK。

### How
application启动的时候知道一组peer，这组peer是被application developer/administrator信任的，可以向discovery查询提供真实的响应。这种peer的一个好的候选者是同一个org的某个peer。
application向discovery service发起configuration query，获取所有的静态信息，如果没有发现服务，这些信息要通过与多有node通讯才能得到。这个信息可以随时发生查询请求进行更新。
discovery service运行在peers上，而不是application，使用gossip通讯层维护的网络meta信息来查找哪个peer是在线的。同时还从peer的state database获取信息，比如相关的endorsement policy。
有了服务发现，application不在需要指定需要哪些peer背书，SDK只要简单发生一个请求到discovery service，给定channel和chaincode id就可以知道需要哪些peer。服务发现会计算一个由两个对象组成的描述符：
- Layouts: peer组成的group的列表，和每个group需要选择的peer数量
- Group to peer mapping: layout中的group到channel的peer的映射，实际上每个group类似单独的org，但是service API是通用的，因此没有org的概念，而是使用group。

下面的例子是`AND(Org1, Org2)`策略的evaluation的描述符，每个org有两个peer：
```
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
```
换句话说背书策略要求来自Org1和Org2的各一个签名，service提供了这些org中存在的peer的名称。
然后SDK从列表中随机选择layout，这个例子中背书策略是Org1 AND Org2，如果是OR策略的话，SDK会随机选择Org1或者Org2，因为随意一个org的一个peer就可以满足策略。
SDK选择layout后，==根据client侧指定的条件从layout中的peers进行选择(SDK可以访问类似ledger hight这样的meta数据，因此可以提供==)。比如根据layout中每个group中peer的数量可以优先选择ledger height比较大的peer，或者排除掉application发现已经离线的。如果根据criteria没有比较好的选择，SDK会随机选择。

### Capabilities of the discovery service
服务发现可以响应下面的查询：
- Configuration query: 返回channel中所有orgs的`MSPConfig`，和channel的orderer endpoints
- Peer membership query: 返回加入channel的peer
- Endorsement query: 返回channel中指定chaincode的endorsement descriptor
- Local peer membership query: 返回响应查询的peer的本地membership信息，默认情况下client必须是管理员

### Special requirements
如果peer激活TLS功能，client连接peer的时候需要提供TLS证书，如果peer没有配置校验客户端证书(clientAuthRequired设为false)，这个TLS证书可以是self-signed。

## 4.4 Channels

## 4.5 Capability Requirements

## 4.6 CouchDB as the State Database

## 4.7 Peer channel-based event services
### General overview
之前版本中，peer event service称为event hub，==当有新的block被加入到peer的ledger的时候这个服务会发送事件，不管这个block属于哪个channel，只对运行eventing peer（比如被连接上等待事件的peer）的org的成员是可访问的。==
从v1.1开始，有两个新的服务提供事件。这些服务的设计完全不同，以每个per-channel为基础提供事件，即注册等待事件发生在channel这个层次，而不是peer，允许对peer的数据访问进行更细粒度控制。==接收事件的请求被peer所在组织之外的identities接受，==这提供了更大的可靠性，并且提供了一种接收已经错过的事件（因为连接问题或者peer加入到一个已经开始运行的网络）的方式

### Available services
- Deliver。这个服务发送已经committed到ledger的整个block，如果chaincode设置了事件，这些事件可以在block的`ChaincodeActionPayload`找到。
- DeliverFiltered。发送`filtered`的block，提交到ledger中的block的最少信息。==用于网络中的peer的owner希望外部客户端主要接受关于交易和交易状态的信息。==如果chaincode设置了事件，可以在`FilteredChaincodeAction`中找到。

==chaincode events的payload不会保护在filtered block中。==

### How to register for events
注册事件是通过发送一个保护deliver seek Info的envelope给peer来实现。deliver seek Info保护需要的start和stop位置，seek的行为(block知道准备好或者没有准备好久失败)。`SeekOldest`和`SeekNewest`用来指定第一个block和最新的block，如果要让服务一直发送事件，SeekInfo需要保护一个`MAXINT64`作为stop
==如果peer上TLS激活，TLS证书hash必须在envelope的channel header中设置。==
默认情况下两种服务都使用Channel Readers policy是否给请求事件的client授权。

### Overview of deliver response messages
event service返回`DeliverResponse`消息，每个消息保护下面中的一个：
- statue - HTTP status Code。如果请求失败会返回合适的failure Code；如果service成功发送了`SeekInfo`请求的所有消息则返回200 - SUCCESS
- block - `Deliver`服务返回
- filtered block - `DeliverFiltered`服务返回

一个filtered block包含：
- channel ID
- number(比如block number)
- filtered transactions数组
- transaction ID
	- type，比如`ENDORSER_TRANSACTION`,`CONFIG`
	- transaction validation code 
- filtered transaction actions。
	- filtered chaincode actions数组
		- transaction的chaincode事件(不包含payload)

### SDK event documentation
[SDK doc](https://fabric-sdk-node.github.io/tutorial-channel-events.html)

## 4.8 Private Data

## 4.9 Read-Write set semantics
### Transaction simulation and read-write set
在一个endorser模拟一个交易的时候会为这个transaction准备一个read-write set。`read set`包含模拟过程中读取的transaction的唯一key列表和对应的committed version。`write set`包含transaction写的唯一key列表和新值(如果是删除操作则新值是一个delete marker)。如果一个transaction多次写一个key的值，只有最后写的值才会保存；如果一个transaction读取一个key的值，处于committed状态的值被返回，即使在read之前transaction已经更新了这个key的值，==即不支持`Read-your-writers`语义。==
可以使用多种方式实现versions，最小的要求是同一个key不能产生重复的version，最简单的比如使用单调递增的数字。当前的实现中使用基于blockchain height的version方案，即committing transaction的height被用作这个transaction修改的所有key的最新的version。在这种方案中，transaction的高度使用一个tuple表示(txNumber是transaction在block中的高度)。相对于单调递增的数字方案这种方案有很多优势，==最主要的是让statedb/transaction simulation/validation这些组件可以做有效的设计选择。==
下面是一个假想的transaction的read-write set，简单起见，这里的version使用递增数字:
```
<TxReadWriteSet>
  <NsReadWriteSet name="chaincode1">
    <read-set>
      <read key="K1", version="1">
      <read key="K2", version="1">
    </read-set>
    <write-set>
      <write key="K1", value="V1"
      <write key="K3", value="V2"
      <write key="K4", isDelete="true"
    </write-set>
  </NsReadWriteSet>
<TxReadWriteSet>
```
如果transaction在模拟的时候进行了范围查询，则范围查询和结果以`query-info`的方式加入read-write set。

## 4.10 Gossip data dissemination protocol
