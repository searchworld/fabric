# 2. Key concept
## 2.1 Blockchain
- What，通过智能合约更新，使用Consensus进行一致性同步的共享复制交易系统。
	- A Distributed Ledger，区块链网络的核心是记录网络上所有交易的分布式账本。去中心化协作反应了现实世界中很多交易的本质。消息不可变，区块链也被称为证明系统（system of proof）
	- Smart Contracts，支持信息的持续更新，提供对账本的可控访问。比如可以约定按物品到达快慢来付钱，当物品到达的时候自动付款。
	- Consensus，保持交易在网络间同步，保证只有指定参与者通过之后才会更新账本，而且当账本更新之后，参与者按相同的交易顺序更新交易，这个过程称为Consensus
- Why useful
	- 现状，今天的交易网络和交易刚出现的时候的记录方式没有太大区别，交易双方分别记录，每次交易的时候都要证明被交易物的归属权。
	- blockchain，每个参与者都对交易进行记录，且记录无法更改，这样可以通过查看交易记录来证明归属权。
- Fabric
	- private and permissioned，通过MSP确认身份
	- share Ledger，ledger子系统有两个部件组成：world state和transaction log。前者用来描述在某个时间点ledger的状态，是ledger的数据库。后者记录所有达到当前的world state的交易数据，是更新历史。Fabric默认使用LevelDb保存world state，可以更换成其他数据库
	- Smart Contract，由区块链外的应用执行，大多数情况下只和world state交互，很少和transaction log交互。

## 2.2 Functionalities
### Identity management
Fabric提供成员身份管理服务来管理用户ID和给网络所有参与者授权，最终形成permissioned的网络。另外Access Control List可以对特定的网络操作授权来提供额外的一层许可，比如一个用户id只能调用chaincode，但是不能部署。

### Privacy and confidentiality
通过channel进行隔离，只能授权访问channel的成员才有权限查看channel内的数据

### Efficient processing
使用execute-order-valide可以并行处理，提高执行效率。另外Fabric通过node的类型来指定网络角色，进而将工作量分开：orderer不需要执行交易和管理账本，peer不需要执行order。

### Chaincode functionality
普通chaincode用于执行业务逻辑。system chaincode对整个channel进行操作：Lifecycle and configuration system chaincode定义channel的规则；endorsement and validation system chaincode定义背书或者校验交易的要求。

### Modular design
Fabric实现模块架构，为网络设计者提供功能选择。不如不同的身份、共识、加密算法可以插入任何的Fabric网络，最终成为一个通用的blockchain体系。

## 2.3 Model
Fabric的关键设计特性：
### Assets
Asset的定义使得具有货币属性的任何东西都可以在网络上进行交换。Asset可以是有形的（不如硬件或者房地产），也可以是无形的（比如合约或者知识产权）。Fabric使用chaincode来修改Asset。在Asset中使用key-value集合来表示asset，状态的变化记录在ledger上。也可以使用二进制或者json来表示asset。

### Chaincode

### Ledger Features
Fabric上所有状态转移的顺序不可篡改的记录。状态转移是chaincode调用(transaction)的结果，每个交易形成一个asset的k-v集合，最终提交给ledger创建、更新或者删除相应的key。
ledger由一个blockchain和一个state database组成，前者以block为单位存储不可变的顺序的记录，后者记录当前的状态。每个channel一个ledger，channel内的成员维护一份ledger的copy。
一些特性：
- 使用基于key的lookup、范围查询和复合key查询来进行ledger的查询和更新
- 使用丰富的查询语言进行只读查询（CouchDB）
- 历史的只读查询，用于数据溯源
- transaction由chaincode读取的k-v（read set）的版本（不是具体值）和chaincode写的k-v（write set）组成
- transaction包含每一个背书节点的签名，并且被提交到order服务
- transaction按顺序写入block，由order服务传送给peer
- peer根据背书策略对transaction进行校验
- 在添加block的时候，通过检查read set的version保证从chaincode执行到现在asset的状态没有变化
- transaction一旦被校验通过并且提交就不可变
- 一个ledger包含一个配置block，定义策略、acl和其他相关信息

### Privacy
channel能够进行隔离。当一个channel上的一部分组织需要保证他们交易数据的秘密性，可以使用private data collection，将这些数据隔离在一个private数据库，逻辑上和ledger分开，只有授权的组织才可以访问。channel是在网络这个level进行隔离；collections是在channel这个level上进行隔离。另外在数据发送给order服务之前还可以对交易数据进行加密，只有拥有相应秘钥的节点才可以解密

### Security & Membership Services
使用MSP，网络内的所有参与者都有一个已知的身份。

### Consensus
consensus并不仅仅只是简单对交易顺序达成一致。consensus可以定义为对组成一个block的所有交易的正确性全面的验证。
当一个block的交易的顺序和结果满足明确的策略检查，consensus最终达成。这些检查发生在交易的整个生命周期，包括使用背书策略强制哪个特定的成员要对交易背书，使用系统chaincode保证这些策略强制实施。在commit之前，peers会使用system chaincode保证有足够的背书且这些背书来自合格的实体。在block追加到ledger之前还会进行version检查，防止出现类似double spend或者破坏数据完整性的其他操作。
除了背书、校验和版本检查，在交易流中还包含各个方向的身份验证。

## 2.4 Blockchain Network

### What
区块链网络是为应用提供ledger和chaincode服务的技术基础设施。大多数情况下，多个组织组成一个财团组建这个网络，各个组织的许可由一个财团达成一致的策略组成。网络策略可以随着时间变化。

### Applications and Smart Contract chaincode
client application只能通过chaincode来查询或者修改ledger。只有组织的管理员才能安装chaincode，当在P1上安装成功之后，P1指定chaincode的所有信息，包括实现逻辑。和实现相对应的是chaincode interface，只描述chaincode的输入和输出，不管具体实现。当有多个peer的时候，可以选择几个来安装，不需要全部安装。安装完成之后，channel中的其他组件并不知道chaincode的存在，必须先在channel上实例化，这个也是由组织的管理员操作。实例化之后channel中的所有组件都知道chaincode的存在，但是并不知道具体的实现逻辑，也就是只知道interface，即实例化的是接口，而安装的是具体实现。可以理解为智能合约是在peer上物理安装，实例化是在channel上逻辑安装。
chaincode实例化的时候要提供的最重要的额外信息是endorsement policy，描述哪些组织必须同意交易，这个交易才会被接受到ledger里。背书策略会被写入到channel的配置里。

### Network completed
chaincode在不同的peer上可以有不同的实现，比如一个用golang，一个用js，只要实现相同的智能合约接口，而且实现了相同的逻辑。

只有安装了chaincode的peer才能运行智能合约，但是channel中的其他节点可以知道智能合约的接口。安装了chaincode的peer负责生成交易。所有peer都可以校验，然后接受或者拒绝交易进入ledger，==但是只有安装chaincode的peer可以参加交易背书==。

peer的类型：
- committing peer，channel中的每个peer都是，接受交易block，然后校验，最后追加到ledger
- endorsing peer，安装有chaincode的peer，客户端利用peer上安装的智能合约产生带有数字签名的交易回复。chaincode的endorsement policy指定组织中哪些节点需要对交易进行数字签名。

peer的角色：
- Leader peer，当一个组织在channel中有多个peer的时候，leader peer负责向组织中其他peer分发来自orderer的交易。一个peer可以选择参加静态（配置指定leader）或者动态（动态选举）的leader select。一个组织可以有一个或者多个leader
- Anchor peer，当一个peer需要和另一个组织通讯的时候，可以使用channel配置中为那个组织定义的anchor peer。一个组织可以有0或者多个anchor peer，一个anchor peer可以负责多种跨组织通讯场景。

chaincode只需要实例化一次，新加进来的节点如果要执行chaincode只需要安装，不需要再实例化。

网络和channel策略的小心使用使得大型网络可以被管理。网络和channel策略构成了automomy（每个channel只受自己的channel策略管理）和control（网络策略可以进行调整，比如确认新的管理员）之间的平衡，形成一个去中心化的网络。

### Adding a new channel
网络和channel配置随着时间变化，有一个进程专门产生配置transaction，捕获配置变更。每个配置变更都会产生一个configuration block transaction。

网络配置和channel配置封装了网络成员同意的策略，用于控制网络资源的访问权限。这两种配置非常重要，网络或者channel的成员只是按照配置运行。这些配置逻辑上只有一份，一份网络配置，每个channel一份自己的channel配置，但是每个peer会在物理上保存一份channel配置副本，每个orderer会保存网络配置的副本。
configuration transaction保持一致使用的区块链技术和user transaction一样。为了修改配置，管理员要提交一个configuration transaction，这个transaction需要mod_policy指定的组织签名。
ordering service节点运行一个mini-blockchain，通过system channel连接和分发network configuration transaction。同样的，application channel的peer可以分发channel configuration transaction。通过物理分发保证逻辑唯一是Fabric的通用模式。这种模式是让Fabric去中心化的同时也是可管理的。

### Changing policy
Fabric认为policy变化是一个常态，不管是组织之间，还是外部监管。policy change通过一个称为modification policy或者mod_policy管理，是first class policy。


## 2.5 Identity
### 什么是身份？
区块链的参与者包括peer、orderer、client application、administrator等，这些参与者都有一个使用X.509封装的数字身份，这个身份用于决定对网络中资源的权限和其他参与者信息的访问权。数字身份有一些额外的属性（比如组织名称等）用于确定permissioned，在Fabric中数字身份和这些属性称为principal。
身份必须是来自可信任的authority才能被验证，Membership service Provider(MSP)就是用来提供这个功能。MSP定义了管理组织中有效身份的规则，默认的MSP实现使用X.509证书作为身份，采用传统的Public Key Infrastructure (PKI)层级模型。

### 一个简单的场景
去超市买东西，只接受Visa/Mastercard，如果你只有一张AMEX，即使这个卡是真的，而且有足够的额度也没法使用。PKI提供提供一个身份列表，就像上面的卡提供商；MSP决定这里面哪些是网络中的组织成员，就像上面的超市决定哪些卡可以使用。MSP将可验证的身份转为区块链网络的成员。

### PKI
public key infrastructure (PKI) 是一系列提供网络安全通讯的技术集合。区块链参与者依赖PKI进行安全通讯，同时保证区块链网络中的消息是已授权的。PKI由下面几部分组成：
- Digital Certificates，一个包含证书持有者信息的文档，最常用的是 X.509。证书使用密码学技术，一旦被篡改就会失效。只要其他人相信Certificate Authority(CA)，这份证书就可以做为身份证明。
- Authentication, Public keys, and Private Keys。验证和数据完整性是安全通讯的重要概念。传统的验证方式使用digital signatures，签名使用私钥加密，使用公钥可以验证签名的有效性。
- Certificate Authorities，用于颁发证书，只要相信CA就可以信任CA颁发的证书。
- Root CAs, Intermediate CAs and Chains of Trust。Intermediate CA含有Root CA颁发的证书，自己本身也可以颁发证书，形成下级的中间CA，最终构成一个信任链。
  Fabric CA是一个private的Root CA，用于管理X.509证书的数字身份。当然也可以使用公共的CA。
- Certificate Revocation Lists，CRL，废除的证书列表，第三方要验证数字身份的时候会先查询CA的CRL，如果证书在CRL里就验证不通过。
- 数字签名保证数据有效性和完整性。CA使用私钥A1(配对的公钥A2)对用户Ben提供的公钥B2(对应的私钥B1)进行加密，生成证书，上面带有公钥B2的明文和加密后的签名。第三方Jack可以从CA获得公钥A2，Ben发送消息的时候带有消息hash的数字签名和证书，Jack使用A2对证书上的签名解密和明文做对比，如果一致说明证书有效，证明消息确实是Ben发出的(有效性);然后使用B2对消息的数字签名解密得到消息的hash，和消息生成的hash做对比，如果一致说明消息没有被篡改(完整性)

### RSA使用过程
```
一个具体的RSA签名过程如下：

1. 小明对外发布公钥，并声明对应的私钥在自己手上
2. 小明对消息M计算摘要，得到摘要D
3. 小明使用私钥对D进行签名，得到签名S
4. 将M和S一起发送出去

验证过程如下：

1. 接收者首先对M使用跟小明一样的摘要算法计算摘要，得到D
2. 使用小明公钥对S进行解签，得到D’
3. 如果D和D’相同，那么证明M确实是小明发出的，并且没有被篡改过
```
[数字签名原理](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)


## 2.6 MemberShip
通过罗列成员的身份，或者识别哪个CA被授权向其成员颁发有效身份，或者两者结合，Membership Service Provider (MSP)识别哪些Root CA和Intermediate CA是可信任的，可以用来定义一个可信域（比如一个组织）的成员，
MSP不只是列出网络参与者或者channel成员，还可以指定在MSP代表的组织内的角色(比如管理员)，在网络和channel环境下定义访问权限(比如channel admins, readers, writers)。channel MSP的配置会广播到相应组织的成员参与的channel。peers, orderers, and clients在channel之外维护local MSP用于验证成员信息和定义对某些组件的权限(比如是否可以安装chaincode)。

### Mapping MSPs to Organizations
organization是一个受控的成员组，可以是一家跨国公司，也可以是一家花店。==organization最终使用一个MSP来管理成员==，这个和X.509的组织的概念不一样。org和MSP的一对一关系适合将MSP命名为ORG1-MSP，有些场景下一个组织有多个成员组，可以再加后缀区分：ORG2-MSP-NATIONAL和ORG2-MSP-GOVERNMENT，==代表不同的信任Root==。
一个org经常被分成多个的organizational units (OUs)，每个ou具有不同的职责，X.509证书中的OU字段指定身份所属的业务线。OU可以用来控制org中的哪部分属于某个区块链网络的成员。另外在一个由多个org组成的consortium里，使用同一个CA，即同一个信任链，这时候OU可以用来区分不同的org。

### Local and Channel MSPs
MSP出现在区块链的两个地方：channel配置(channel MSP)和参与者本地(local MSP)。Local MSP是定义给client(比如用户在chaincode交易的时候在用户侧证明自己是channel的一个成员，或者作为系统的某个角色，比如交易配置管理员)和node(比如peer和orderer，定义node的permission)。
每个node或者user都要定义一个local MSP来指定在node或者user这个level上的管理或者参与权。channel MSP指定在channel这个level上的管理或者参与权，参与到channel的每个组织都必须有一个MSP，peer和orderer共享channel MSPs相同的视图，可以证明网络参与者。这意味着如果一个组织要加入到一个channel，含有这个组织成员信任链的MSP必须加入到channel配置中，否则这个组织发起的交易会被拒绝。
local MSP和channel MSP的功能都是将身份转成角色，关键的区别在于他们的scope。
![undefined](http://hyperledger-fabric.readthedocs.io/en/latest/_images/membership.diagram.4.png) 
上图中当B要在peer上安装一个智能合约的时候，peer会检查local MSP上是否有对用户的授权。如果B使用RCA1授权则可以安装成功。当B要在channel上实例化智能合约的时候，由于是channel上的操作，需要检查channel上所有的channel MSP是否验证通过。
每个node上有且仅有一个local MSP，每个channel配置有一个逻辑上的channel MSP，通过共识算法分发给channel的参与者，因此每个node物理上可以有多份的channel MSP。

### MSP Levels
local MSP和channel MSP的拆分反应了对本地资源和channel资源管理的需求，可以认为是在不同的level上，高level的反应了网络管理需求，低level的用于处理私有资源的管理身份。每个level都需要定义MSP。
![MSP level](http://hyperledger-fabric.readthedocs.io/en/latest/_images/membership.diagram.2.png) 

- Network MSP，通过为参与组织定义MSP来定义谁是网络的成员，这些成员哪些被授权执行管理任务。==在哪里定义？==
- Channel MSP，channel需要单独对自己成员的MSPs进行管理，channel在不同的组织之间建立私人通讯，反过来这些组织可以对channel进行管理。在channel MSPs的背景下，Channel policies可以理解为定义了哪些成员可以对channel进行某种操作，比如增加org，实例化智能合约。管理channel和管理网络两者之间没有什么关系
- Peer MSP，概念上和Channel MSP的功能相同，但是只能应用在local MSP定义的peer上，比如对能否在当前peer上安装智能合约进行验证。
- Orderer MSP

### MSP Structure
MSP可以看做是RCA和ICA的规范说明，这些CA用来确定actor在不同组织的成员资格。除了这两者还有其他需要用来辅助实现这个功能。
![MSP Structure](http://hyperledger-fabric.readthedocs.io/en/latest/_images/membership.diagram.5.png) 
上面的结构可以看做是一个目录树，MSP是根目录，每个组成部分是一个子目录。
- Root CAs，至少含有一个组织信任的Root CA颁发的证书，这个是最重要的，其他CA都是这个发起的。
- Intermediate CAs，可以用来代表org的不同组成部分，在财团中可以代表不同的org，其issue链最终可以到达Root CA。这个目录可以为空。
- Organizational Units (OUs)，包含一个ou列表，认为是这个MSP代表的org的一部分，==当要限制一个指定ou身份的成员的时候有用。==指定ou是可选的，如果不指定则MSP的所有identities都是组织的成员。
- Administrators，包含一个或者多个证书，定义这个组织的管理员。管理员角色并不代表可以管理特定的资源，这个资源是由 policies决定的，比如一个channel policy可以指定ORG1-MANUFACTURING管理员有权向channel中添加组织。
- Revoked Certificates，概念上类似CRL，但是不是实际存储证书， X.509-based identities, these identifiers are pairs of strings known as Subject Key Identifier (SKI) and Authority Access Identifier (AKI)
- Node Identity，包含这个node的身份，对于X.509体系，这里是一张X.509 certificate，允许node在发送给网络中其他成员的信息里证明自己。比如在交易请求回复中加上这个证书，证明这个peer已经接受了这笔交易。对于local MSP这个目录是必须有的，且只能有一张证书。channel MSP不需要。
- KeyStore for Private Key，这个目录为peer或者orderer的local MSP定义，包含这个node的signing key，这个key和Node Identity中的身份对应，用于对数据做签名。local MSP必须有，且只能有一个private key。==channel MSP只用来验证，不需要签名，因此没有这个目录。==
- TLS Root CA，Root CA颁发的用于TLS communications的self-signed X.509 certificates(自签名根证书是指一对密钥对的私钥对自己相应的公钥生成的证书请求进行签名而颁发的证书，证书的申请人和签发人都是同一个。)。当一个peer需要连接到一个orderer获取ledger更新的时候就需要TLS通讯。MSP TLS信息和网络中的node有关，跟应用或者管理员无关。这个目录至少要有一个证书。
- TLS Intermediate CA，0或者多个中间证书，当使用商用CA的时候比较有用。

## 2.7 Peers
区块链网络由一组peer组成，peer是网络的基础元素，用于host账本和智能合约。peer暴露了一组API用于管理员和应用交互。

### Ledgers and Chaincode
准确的说，peer实际上host的是ledger的实例和chaincode的实例。这提供了网络冗余，避免单点失败。
应用和管理员如果要和ledger chaincode打交道需要通过peer，因此peer是Fabric网络的最基础模块。peer刚创建的时候既没有ledger也没有chaincode。
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.2.png)

### Multiple Ledgers & Chaincode
一个peer可以host多个ledger(每个channel对应一个)，绝大多数peer会安装至少一个chaincode，提供ledger的查询和更新功能。每个peer都会有system chaincode。
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.4.png)

### Applications and Peers
Ledger-query交互需要application和peer间3个步骤的会话；ledger-update需要额外的两个步骤。当需要访问ledger或者Chaincode的时候application总是连接到peer上。SDK提供的API可以让application连接peer，invoke chaincode来产生交易，提交交易到网络中最终排序后写到ledger，当这些都做完的时候接收消息。
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.6.png)

ledger-query立即返回，在上图中到3就结束了。因为peer本地含有需要查询的所有信息，不需要跟其他peer做交互。application可以连接到多个peer进行查询，比如怀疑当前查询的信息不是最新的。上图中可以看出查询只要三个步骤。
ledger-update除了查询的三个步骤之外还需要额外两个步骤，因为一个peer不能独立完成更新操作，需要先对这个变化达成共识。

### Peers and Channels
channel可以看做一个物理peer集合组成的逻辑结构。peer提供访问和管理channel的控制点。

### Peers and Organization
区块链网络由很多组织组成，不同的peer可能属于不同的组织。这个网络既是由贡献资源（就是这里的peer）的组织组成，也是由他们管理。只要这些组织里有一个存在，这个网络就可以运行，这就是去中心化的核心。不同组织的应用可以不一样，每个组织可以按自己的逻辑处理相同的ledger副本。对于查询操作，application只要连接到自己组织的peer；对于更新操作，可能需要连接到其他组织的背书节点。
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.8.png)

### Peers and Identity
peer从CA拿到一张数字证书作为自己的身份，网络中的每个peer从org管理员分配一张数字证书。
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.9.png)
当一个peer通过channel连接到区块链网络的时候，channel配置里的policy使用peer的身份来决定peer的权利。身份到组织的映射是由MSP来完成的，MSP决定了一个特定组织里peer分配到的角色，相应的可以访问区块链资源。而且一个peer只能属于一个组织，因此关联到单独的一个MSP。
除了peer，其他需要和区块链网络交互的也需要从MSP和数字证书中获得自己的组织身份，比如application、end user、administrator、orderer。使用身份和区块链网络交互的每个实体称为一个**principal**(主角)。
另外peer的物理位置已经不重要，因为是通过身份来识别属于哪个组织的。

### Peers and Orderers
一个peer需要其他peer通过一个ledger更新操作，然后才能应用到这个peer的本地ledger，这个过程称为consensus。当所有要求的peer通过了这个交易并且交易提交给ledger，peer会通知application更新成功。这是ledger-update操作比ledger-query多的两个步骤。
为了保证区块链网络中所有peer的ledger保证一致，需要一个3阶段过程。
- Phase1: Proposal。application和endorsing peer集合交互，每个endorsing peer为ledger更新提供背书，但是不会将这个更新写入ledger中。这一步和order无关，只是应用要求背书节点通过更新操作。
  阶段1开始的时候，application生成一个transaction proposal，发送给所有需要的背书节点（在背书策略里指定）。每个endorsing peer使用交易提议独立执行chaincode，产生transaction proposal response，这个更新还不会应用到ledger上，而是简单的加上签名，然后返回给application。当application收到足够的带签名的提议响应，第一节点完成。
  ![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.10.png)
  背书节点的选择依赖endorsement policy（为chaincode定义），定义了哪些组织需要为一个ledger变化提议做背书。这里可以将达成共识理解为每个利益相关的组织必须对账本变更提议做背书，否则这个变化就不能写入账本。
  一个peer通过对提议相应加数字签名，并且对整个payload使用私钥加密，这样可以证明这个组织确实产生了一个响应。
  对于同一个交易提议不同的peer可以返回不同即不一致的交易响应，这可能是由于这个响应是在不同的时间生成的，此时ledger的状态也不一样，这种情况下application只要要求一个up-to-date的proposal response。比较少见，但是比较严重的可能是因为chaincode是non-deterministic，因为这会导致不一致的状态，最终无法写入ledger。
  Phase1结束的时候应用可以选择丢弃不一致的交易响应，提前终止交易流程。如果不一致的交易响应被提交了，也会被拒绝。
- Phase2: Packaging。上一步单独的endorsement收集到一起作为一个交易，打包到block里。在这个阶段order是主角。order从很多application接收包含交易提议响应的交易，然后对其他交易排序，然后对一批交易打包成一个block，最后分发给连接到order的所有peer，包括之前的endorsing peer。
  ![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.11.png)
  order并发接收来自网络中不同channel下不同application的proposed ledger更新，其工作就是把这些更新组织成定义好的顺序，然后打包成block用于分发。这些block最终就成为blockchain的block。当order产生了一个指定大小的block，或者最大超时时间过了，这个block就会发送到连接到特定channel的所有peer上。
  block中交易的顺序并不一定是交易到达order的顺序，交易可以按任意顺序打包成block，这个顺序就成为执行的顺序。==重要的不是这个顺序是什么，而是有一个严格的顺序。==
  严格的交易顺序使得Fabric和其他可以将相同的交易打包到多个不同的block的区块链有点不同。在Fabric里这种情况不可能发生，一个orderer集合产生的block是final的，因为一个交易一旦写入一个block，他在ledger上的位置就是保证不变的。Fabric的finality意味着ledger fork不可能发生。交易一旦写入一个block，在未来任何时间点那个交易的历史不能重写。
  peer host ledger和chaincode，orderer一般不会。orderer打包的时候不会关心交易的值是什么，只是简单的打包。这是Fabric的一个重要属性，交易从来不会被丢弃或者de-prioritized。
- Phase3: Validation。上一步的block分发到每个peer，在peer做完校验后写入ledger。在每个peer上block的每个交易都会被校验，保证交易已经是相关组织背书过的，验证失败的交易不会applied到ledger，但是会保存起来供审计。
  ![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/peers.diagram.12.png)
  peer连接到orderer，当新的block产生的时候，会收到一个copy。每个peer会单独处理这个block，但是和其他peer的处理方式是一致的，因此ledger会保持一致。并不是所有peer都要连接到order上，peer也会通过gossip协议把block传给其他peer，然后那些peer再单独处理。
  peer收到block后会按block中交易的顺序来处理每个交易。对于每个交易，每个peer会验证这个交易是否已经被背书策略规定的组织背书。校验的过程验证相关的组织已经产生相同的结果。这个校验也不同于Phase1的背书检查，Phase1中是application从背书节点接受响应，然后决定要不要发送交易提议，如果application违反了背书策略，发送了错误的交易，这一步的校验还是会拒绝这个交易。
  如果交易是正确背书的，peer会尝试写入ledger。为了做到这个，peer要进行ledger一致性检查，来验证ledger的当前状态和交易更新产生的时候的ledger状态是兼容的。比如block中前面的某个交易可能已经更新了ledger中相同的asset，当前的交易更新不再有效，因此不能写入ledger。通过这种方式每个peer的ledger副本保持一致。
  peer成功校验每个单独的交易之后就会更新ledger。校验失败的ledger不会applied到ledger，但是会和成功的交易一样保留供审计。这意味着peer上的block和从order上收到的几乎是一样的，除了每个交易上加了有效或者无效的标识。
  同时这一步也不需要像Phase1那样执行chaincode，因此chaincode只需要在endorsing peer上存在，而不是整个区块链网络。这可以保证chaincode的逻辑对背书组织是机密的，但是chaincode的结果是channel内共享的，这样提高了scalability。
  最后每次block提交到ledger上的时候，peer会产生一个事件。Block事件包括所有的block内容，block transaction事件只包含汇总信息，比如block中的每个transaction是否是有效的还是无效的。chaincode事件表示chaincode已经产生结果也是这个时候产生，application可以注册这些事件。
  
### Orderers and Consensus
整个交易工作流过程称为consensus，因为在order的参与下所有peer已经对交易的顺序和内容达成一致。这是一个多步骤的过程，application只有在process完成的时候才会收到通知，在不同的peer上可能是不同的时间。

## 2.8 Private data
数据隔离可以使用两种方式，一种是创建不同的channel，但是需要配置运维，成本比较高，而且对于下面这种场景也不满足：要所有channel参与者都可以看到交易，但是有部分数据是private的。第二种是使用private data collection(PDC)

### what
PDC由两部分组成：
- actual private data，通过gossip发送给授权可以看到的peer。这部分数据存储在peer上一个私有数据库（有时候称为SideDB）。ordering服务不参与这一步，看不到这部分数据。
- hash of that data，像正常的transaction一样被背书、order然后写入channel中每个peer的ledger上。这部分数据作为交易的证据用于状态验证，可以用于审计。
![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/PrivateDataConcept-2.png)
PDC的成员在遇到纠纷或者想要将asset传输到第三方的时候可以分享私有数据，第三方只要计算hash值做对比就可以确定这份私有数据是否可信。

PDC vs new channel
- 当所有的交易的机密性只要在整个channel内的org间保持的时候选择使用channel
- 当交易需要在多个org直接共享，但是只有这些org的一个子集可以访问交易的部分或所有数据。另外PDC直接点对点传播，不使用block，因此当需要对order保持机密的时候也要使用PDC

### Transaction flow with private data
为了保证PDC的机密性，当有PDC参与的时候交易流程会出现变化：
1. 客户端提交提议请求到endorsing peer，这些peer是授权的org的成员。私有数据或者用于在chaincode中生成私有数据的数据放在提议的一个`transient`字段里
2. endorsing peer执行交易，将私有数据保存在`transient data store`(peer本地上的一个临时存储)，这些peer根据PDC的协议通过gossip向其他授权的peer分发私有数据
3. endorsing peer向client返回带有公开数据的提议响应，包括私有数据key、value的hash。私有数据不会发送到client
4. client应用向ordering service提交交易（含有PDC的hash），然后像正常交易一样写入block，然后分发到所有peer。channel上所有peer都可以使用hash值验证PDC，不需要知道真正的PDC。
5. 在block commit的时候，peer使用collection 策略决定是否被授权访问私有数据，如果有就检查本地的`transient data store`，看是否已经收到PDC，如果没有会尝试从别的peer拉取，然后使用hash校验数据，最后commit。在validation/commit的时候，私有数据被移动到相应的private state database和private writeset storage，然后从`transient data store`中删除

### Purging data
对于非常敏感的数据，一段时间后私有数据可能需要删除，只留下hash作为交易的证据。另外的场景可能私有数据可以移动到区块链外部的数据库。或者在chaincode处理完之后就可以清除，这种场景通过当PDC有几个block没有发生变化之后可以清除来支持。

## 2.9 Ledger
### A Blockchain Ledger
由两部分组成：
- world state，保存ledger状态当前值的数据库，方便程序获取状态的当前值，而不用遍历整个交易日志来计算。默认情况下ledger状态使用key-value表示。world state可以频繁创建、更新和删除。
- blockchain，记录决定world state的所有变化的交易日志。交易收集到block里，然后追加到blockchain里，可以看出导致当前world state的历史变化。blockchain的数据结构和world state很不一样，因为一旦写入就不能修改。是block的不可变序列。

![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/ledger.diagram.1.png)
可以认为逻辑上只有一个ledger，但是实际上通过consensus每个peer都保持一份完全一样的副本。Distributed Ledger Technology(DLT)就是实现这种ledger的技术。

### World State
![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/ledger.diagram.3.png)
application通过chaincode来访问world state，API包括使用key来get/put/delete
状态。物理上world state实现成数据库，这样可以提供丰富的存储和获取数据的操作。Fabric可以支持多种数据库。只有校验通过的交易才会应用到world state上，校验失败的不会。每个状态都有一个version号，状态每次变化版本号都会递增，当交易创建的时候需要满足版本的要求。这个检查保证创建交易的时候world state从相同的预期值变化到相同的预期值。
world state可以随时从blockchain重新生成，因此当一个peer新加入的时候，或者peer重启，在接受新的block之前world state会自动重新生成

### Blockchain
Blockchain是一个交易日志，结构上类似互相连接的block，每个block包含一系列的交易，每个代表对world state的一个查询或者更新。
每个block的header包括这个block的所有交易的hash和前一个block头部保存的hash值，通过这种方式ledger上所有交易是顺序的，使用密码学连接在一起。这种hash和连接使得ledger数据非常安全。如果有一个节点的ledger被篡改，没法使其他节点相信他的ledger是正确的，因为ledger分布在各个节点上。
物理上Blockchain使用文件来实现，因为Blockchain的数据结构只有少数几种简单操作，追加到结尾是最主要的操作，查询通常很少。
![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/ledger.diagram.2.png)
第一个节点称为genesis block，是ledger的起点，但是不包含任何的用户交易，只包含一个配置交易，包含channel的初始状态。

### Block
一个block包含三个部分：
- Block Header.包含三个部分，在block创建的时候写入
	- Block number，从0开始（Genesis block），每新增一个block加1
	- Current Block Hash，当前block的所有交易的hash
	- Previous Block Hash，前一个block的hash副本
	![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/ledger.diagram.4.png)
- Block Data，包含一个交易列表，当block创建的时候写入
- Block Metadata，包含block的创建时间、block创建者的证书公钥和签名、block committer对每个交易添加的有效/无效标志，这部分数据不包含在hash里面。

### Transaction
![](https://hyperledger-fabric.readthedocs.io/en/latest/_images/ledger.diagram.5.png)
- Header,交易的meta信息，比如相关chaincode的名称、版本
- Signature，一个application创建的数字签名，用于校验交易明细没有被篡改，需要application的私钥来生成。
- Proposal，application提供给chaincode的输入参数，用于创建proposed ledger update。当chaincode运行的时候，这个proposal提供输入参数，和当前的world state决定一个新的world state
- Response，world state的前后值，用Read Write set(RW-set)表示，是chaincode的输出，如果transaction校验通过，会应用到ledger中更新world state
- Endorsements，来自满足endorsement policy的足够多的每个org的带签名的交易响应列表，可能会有多个。每个背书封装了自己组织的特定的交易响应，不需要包含不满足背书的交易响应。

这些是ledger数据结构的主要部分。

### World State database options
当前world state的数据库包括LevelDB和CouchDB，前者是默认选项，尤其适合ledger state是简单的k-v结构。LevelDB嵌入到节点详图的操作系统进程。CouchDB适合JSON格式的ledger state，支持丰富的查询和更新，在操作系统进程外单独运行。
