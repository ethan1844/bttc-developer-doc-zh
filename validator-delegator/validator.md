# 验证人

验证人(Validator)是网络中的参与者，他将代币锁定在网络中并运行验证人节点以帮助运行网络。验证人有以下职责：

* 质押网络令牌并运行验证人节点以作为验证器加入网络
* 通过验证区块链上的状态转换获得质押奖励
* 因停机等活动而受到处罚

区块链验证人是负责验证区块链内交易的人，对于 BitTorrent-Chain，任何参与者都可以通过运行全节点获得奖励和交易费用，从而有资格成为BitTorrent-Chain的验证人。BitTorrent-Chain中的验证人是通过定期发生的链上拍卖过程选择的，这些选定的验证人将作为区块生产者和验证者参与。

Validator需要有两个地址：

1. Owner 地址：验证人可以通过该地址处理`与管理相关`的功能，比如取消抵押、获取奖励、设置委托参数。
2. Signer 地址：验证人从这个地址签署检查点并运行节点。

## 架构

BitTorrent-Chain 网络分为3层

### Root Contracts

TRON及其他区块链网络上的Root合约，支持用户通过存取款的方式将代币映射到 BitTorrent-Chain，及支持质押等功能。

### Validator

验证BitTorrent-Chain区块，定期发送Checkpoint至支持的TRON及其他区块链网络。

**Bridge**：负责监听各链路事件，发送事件消息等。

**Core**：共识模块，包括Checkpoint(BitTorrent-Chain链的状态快照)的验证，Statesync事件\&Staking事件的共识。

**REST-Server**：提供相关API服务。

### BitTorrent-Chain

BitTorrent-Chain层的区块生产者是验证者的一个子集，由验证人定期改组。

![](https://i.imgur.com/Ohh3DKD.png)

## 功能

区块链验证人是负责验证区块链内交易的人。对于BitTorrent-Chain来说，任何参与者都有资格成为BitTorrent-Chain的验证人，通过运行一个完整的节点来获得奖励和交易费用。为了确保验证人的良好参与，他们锁定了他们的一些BTT代币作为生态系统的股份。

BitTorrent-Chain的验证人是通过链上的质押来选择的，这个过程会定期进行。这些被选中的验证人，作为区块生产者和验证者参与其中。一旦一个检查点（一组区块）被参与者验证，那么就会在TRON&以太坊\&BSC上进行更新，根据验证人在网络中的股份，为其发放奖励。

### 验证人的职责

* 通过在TRON上的质押合约中锁定BTT代币来加入网络。
* 验证人可以随时退出系统，可以通过unstake 在合约上执行交易来完成，
* 验证人可以随时增加质押BTT代币数量，以增加质押能力。
* 设置验证人节点后，验证人将执行以下操作： 1.区块生产者选择 2.在BitTorrent-Chain 上验证块 3.检查点提交 4.在以太坊上同步对BitTorrent-Chain 质押合约的更改 5.从TRON&以太坊\&BSC 到BitTorrent-Chain层的状态同步
* 验证人需要保持最低数量的代币来支付相关链上的交易费用。

### Validator层

Validator层将BitTorrent-Chain产生的区块聚合成默克尔树，并定期将默克尔根发布到根链。这种定期发布被称为“检查点”。对于 BitTorrent-Chain 上的每几个区块，一个验证人（Validator）：

1. 验证自上次检查点以来的所有块
2. 创建块哈希的默克尔树
3. 将merkle root发布到主链

检查点很重要，原因有两个：

1. 在根链上提供终结性
2. 在提取资产时提供销毁证明

### BitTorrent-Chain层

BitTorrent-chain层中的区块生产者，BitTorrent-chain层中的VM与EVM兼容，是一个基本的Geth实现，并对共识算法进行了自定义修改。

### 检查点机制(Checkpoint)

在Validator中通过Tendermint的加权轮回算法来选择一个提议者，在Tendermint上成功提交一个检查点有2个阶段的提交过程，一个是通过上述Tendermint算法选择的提议者发送一个检查点，在提议者字段中包含他的地址，所有其他提议者在将其添加到他们的状态中之前将对此进行验证。

然后下一个提议者发送一个确认交易，以证明之前的检查点交易在以太坊主网中已经成功了。每一个验证者集的变化将由Validator上的验证人节点转发，该节点被嵌入到验证人节点上。这使得验证人在任何时候都能与TRON\&Ethereum等链上的BitTorrent-chain合约状态保持同步。

部署在TRON\&Ethereum等链上的BitTorrent-chain合约被认为是最终的真相来源，因此所有的验证都是通过查询TRON\&Ethereum等链上的BitTorrent-chain合约完成的。

### 交易费用

BitTorrent-chain层的每个区块生产者都将获得每个区块收取的一定比例的交易费用。

### 状态同步机制

Validator层上的验证人接收StateSynced事件并将其传递给BitTorrent-chain层。

接收者合约继承了IStateReceiver,相关自定义逻辑位于onStateReceive函数内。

Dapp/用户 需要做的事情是与state-sync 一起工作。

1. 调用StateSender合约的 `syncState()`函数。
2. 上述函数将触发`StateSynced(uint256 indexed id, address indexed contractAddress, bytes data);`事件
3. Validator层上的所有验证人都会收到这个事件。
4. 一旦Validator层上的状态同步交易被包含在一个区块中，它就会被添加到待定状态同步列表中。
5. BitTorrent-chain层节点通过API调用从Validator上获取待定的状态同步事件。
6. 接收者合同继承了IStateReceiver接口，解码数据字节和执行任何行动的自定义逻辑位于onStateReceive函数中。

## 质押相关合约接口说明

### 质押

* 合约方法：StakeManagerProxy:stakeFor(address, uint256, uint256, bool, bytes memory)
* 参数
  * address user：质押账号地址，即验证人的owner地址
  * uint256 amount：质押的BTT数量
  * uint256 deliveryFee：中间层手续费
  * bool acceptDelegation：验证人是否接受委托人的投票；建议将其设置为true，即接收委托人的投票，之后可以通过ValidatorShare:updateDelegation方法关闭或开启；但是如果调用 stakeFor 时 acceptDelegation 是 false，之后就无法再更改，也就无法再接受投票。
  * bytes memory signerPubkey：签名账户公钥；即验证人的signer地址的公钥，需要把前导“04”去掉
* 说明
  1. 参数`amount`允许的最小值可通过StakeManagerProxy:minDeposit方法查询（目前为10^30， 也就是10^12个BTT）。
  2. 参数`deliveryFee`允许的最小值可通过StakeManagerProxy:minHeimdallFee方法查询（目前为10^23，也就是100000个BTT）。
  3. 在调用stakeFor方法之前，需要先调用[`BTT`](https://tronscan.org/#/contract/TAFjULxiVgT4qWk6UZwjqwZXTSaGaqnVp4/code)的approve方法授权大于质押数量的BTT给[`StakeManagerProxy`](https://tronscan.org/#/contract/TEpjT8xbAe3FPCPFziqFfEjLVXaw9NbGXj/code)合约。
  4. 用户质押成功后，可通过stakeManagerProxy:getValidatorId方法获取到验证人的validatorID，然后通过stakeManagerProxy:validators方法获取到validator的详细信息。

### 追加质押

* 合约方法：StakeManagerProxy:restake(uint256，uint256，bool)
* 参数
  * uint256 validatorId：质押的validator id
  * uint256 amount：质押数量
  * bool stakeRewards：奖励是否加入质押
* 说明
  1. 调用此方法的前提条件是成功调用了StakeManagerProxy:stakeFor方法并成为验证人。

### 领取奖励

* 合约方法：StakeManagerProxy:withdrawRewards(uint256)
* 参数
  * uint256 validatorId：领取奖励的validator id
* 说明
  1. 验证人可通过withdrawRewards方法来领取奖励，执行成功后奖励立刻到达验证人账户。

### 取消质押

当验证人想退出系统，停止验证区块和提交检查点时，验证人可以取消质押。为了保证良好的参与度，取消质押的验证人的质押部分代币将被锁定withdrawalDelay个周期。

* 合约方法：StakeManagerProxy:unstake(uint256)
* 参数
  * uint256 validatorId：解除质押的validator id
* 说明
  1. 验证人可通过unstake方法来取消质押，取消质押后立刻返还奖励的代币到验证人账户，但质押的代币需要通过unstakeClaim函数来申领。
  2. unstakeClaim方法必须等待withdrawalDelay（目前为80）个检查点后才可以被调用。

### 领取质押的BTT

* 合约方法：StakeManagerProxy:unstakeClaim(uint256)
* 参数
  * uint256 validatorId：领取质押金的validator id
* 说明
  1. 在取消质押后，需要等待withdrawalDelay（目前为80）个检查点后，才可以调用此方法来领取之前质押的BTT。

### 更新validator签名公钥

* 合约方法：StakeManagerProxy:updateSigner(uint256，bytes memory)
* 参数
  * uint256 validatorId：validator id
  * bytes memory signerPubkey：新签名公钥
* 说明
  1. 验证者可以更新签名账户，但是两次更新操作的时间间隔需要满足大于signerUpdateLimit（目前为100）个检查点。

### 更新佣金比例

* 合约方法：StakeManagerProxy:updateCommissionRate(uint256，uint256)
* 参数
  * uint256 validatorId：validator id
  * uint256 newCommissionRate：新佣金比例
* 说明
  1. 验证者可以更新佣金比例，但是两次更新操作的时间间隔需要满足大于WITHDRAWAL\_DELAY（目前为80）个检查点。
  2. 佣金比例需要小于等于100
