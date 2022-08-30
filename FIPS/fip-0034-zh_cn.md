---
fip: 34
title: Fix pre-commit deposit independent of sector content
author: Alex North (@anorth), Jakub Sztandera (@Kubuxu)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/290
status: Accepted
type: Technical
category: Core
created: 2022-0218
---

## Simple Summary 简述
Set the pre-commit deposit to a fixed value regardless of sector content.
将pre-commit保证金设置为固定值，而不考虑扇区内容。

## Abstract 摘要
The sector pre-commit deposit (PCD) is currently set equal to an estimate of the _storage pledge_ for a sector.
The storage pledge is a 20-day projection of the expected reward to be earned by the sector.
Expected reward depends on power, which in turn depends on the presence of verified deals hosted by the sector.

扇区pre-commit保证金（PCD）当前设置为等于扇区的_storage pledge_估计值。
存储承诺是对该扇区预期奖励的20天预测。
预期奖励取决于存力，而存力又取决于该扇区是否存在经验证的交易。

This proposal sets the pre-commit deposit to a value independent of sector content:
the estimated storage pledge of a sector with space-time that is exactly filled by verified deals.
This breaks coupling between storage power mechanics and the built-in (privileged) storage market actor, 
which is necessary as a prerequisite to a larger re-architecture supporting alternative deal markets on the FVM.
It also presents an opportunity to reduce deal-related gas costs.

本提案将pre-commit保证金设定为独立于扇区内容的价值：
一个具有时空扇区的预计存储抵押，完全由经验证的交易填补。
这打破了存储能力机制和内置（特权）存储市场actor之间的耦合，
这是支持FVM上替代交易市场的更大重新架构的必要前提。
它也提供了降低交易相关gas成本的机会。

## Change Motivation 改变动机
Using the sector-specific storage pledge as PCD is inefficient and constrains the design space of APIs
for moving to a programmable storage and market architecture for the FVM.
The proposal at https://github.com/filecoin-project/FIPs/discussions/298 outlines an
architectural change to the built-in miner, market, and verified registry APIs which will 
support the deployment of alternative, equally-capable marketplace actors as user-programmed actors 
on the FVM by removing  the privileged place of the built-in storage market actor as a mediator of storage quality.

使用特定于扇区的存储保证作为PCD是低效的，并且限制了API的设计空间
用于移动到FVM的可编程存储和市场架构。
建议https://github.com/filecoin-project/FIPs/discussions/298概述
对内置的miner、market和verified registry API的架构更改，将
支持部署具有同等能力的替代市场参与者作为用户编程参与者
在FVM上，取消了内置存储市场actor作为存储质量中介的特权地位。

This proposal is a step towards that architecture, changing the sector onboarding process to:  
本提案是朝着该架构迈出的一步，将扇区登记流程改变为：
- calculate a fixed PCD regardless of sector content, and
- 计算固定PCD，而不考虑扇区内容，以及
- commit to the sector's unsealed CID (aka CommD) at pre-commit time.
- 在预提交时间提交到扇区的未密封CID（又名CommD）。

This will establish the pre-commit on-chain state schema matching that required by subsequent
removal of the market actor from the pre-commit flow entirely (simplifying future changes). 
It also presents an opportunity to resolve some redundant loading of deal metadata from state during prove-commit,
and so reduces the total gas consumed when onboarding sectors with deals.

这将建立链上pre-commit状态模式，与后续
完全从pre-commit流程中删除市场actor（简化未来的更改）。
它还提供了一个机会来解决在证明提交期间从状态对交易元数据的一些冗余加载，
因此，在加入交易扇区时，减少了总耗gas。

This change also resolves a reduction in security buffers inadvertently introduced in 
[FIP-0019](./fip-0019.md) SnapDeals. 
The PCD should be calculated according to the expected reward of a sector, 
but when a committed-capacity sector is subsequently upgraded with verified deals, 
its reward increases from that for which its PCD was calculated.
There is sufficient buffer in the PCD calculation that no practical risk was introduced, 
but setting the PCD to a fixed value based on the _maximum_ reward a sector might earn restores that security buffer.   

此更改还解决了在[FIP-0019](./fip-0019.md)快照交易中意外引入的安全缓冲区的减少。
PCD应该根据一个扇区的预期回报来计算，
但是当承诺的容量扇区随后用经验证的交易升级时，
其奖励比计算其PCD的奖励增加。
在PCD计算中有足够的缓冲，没有引入实际风险，
但是，基于一个扇区可能获得的 _maximum_ 奖励将PCD设置为固定值会恢复该安全缓冲区。

The specific mechanism for implementing this change is driven by goals of:  
实施这一变化的具体机制由以下目标驱动：
- retaining the current exported API, to minimise integration/operational effort,
- 保留当前导出的API，以最小化集成/操作工作，
- retaining the current operator-friendly safety checks that prevent most instances of invalid deals
from costing a provider their pre-commit deposit.
- 保留当前对运营商友好的安全检查，以防止大多数无效交易的情况导致提供商支付预提交保证金。

Note that it would be possible to reduce gas costs even further by sacrificing the safety checks but,
since those gains will be realised anyway without sacrificing ergonomics by subsequent re-architecture, 
this proposal retains the checks (while still providing a net gas usage reduction).

注意，可以通过牺牲安全检查来进一步降低gas成本，
由于这些增益无论如何都将实现，而不会通过随后的重新架构而牺牲人体工程学，
该提案保留了检查（同时仍提供网络gas使用量减少使用）。

The architecture for programmable markets on the FVM relies on this change to pre-commit deposit as a pre-requisite.
If the pre-commit deposit is not changed to be independent of content,
then further state and interactions would need to be added to compute the verified deal weight
during sector pre-commit.

FVM上可编程市场的架构依赖于将预提交存款作为先决条件。
如果预提交存款未被改变为独立于内容，
然后，需要添加更多的状态和交互来计算验证的交易权重
在扇区预提交期间。

## Specification 详述
Calculate pre-commit deposit as the 20-day projection of expected reward earned by a sector with a 
sector quality of 10 (i.e. full of verified deals), _regardless of sector content_.
Leave the initial pledge value, due when the sector is proven, unchanged.

将预承诺保证金计算为扇区质量为10的扇区（即，全部已验证交易）所获得的预期回报的20天预测，_无论扇区内容如何_。
保留该扇区经证明后到期的初始质押价值不变。

Change the `miner.SectorPreCommitOnChainInfo` structure to
- remove the `DealWeight` and `VerifiedDealWeight` fields, and
- add an `UnsealedCID` field.

During sector pre-commit change the call to `market.VerifyDealsForActivation` to 
- validate the deals, and
- compute and return each sector's unsealed CID, and
- no longer compute or return deal weight.

Store the unsealed CID in the `SectorPreCommitOnChainInfo`.
(This mechanism matches the future schema and flow of committing to the unsealed CID during pre-commit,
while retaining the ergonomic deal-validity checks, which will be removed later).

将未密封的CID存储在`SectorPreCommitOnChainInfo`中。
（该机制匹配在预提交期间提交到未密封CID的未来模式和流程，
同时保留人体工程学交易有效性检查（稍后将删除）。

During sector prove-commit, remove the invocation to `market.ComputeDataCommitment`, 
and deprecate that method. The data commitment was calculated while validating deals during pre-commit.
This provides a net gas saving.

在扇区证明提交期间，删除对`market.ComputedDataCommit`的调用，
并反对该方法。数据提交是在预提交期间验证交易时计算的。
这提供了网络gas的节省。

Change the parameters and return type of `market.ActivateDeals` to match the prior types of
`market.VerifyDealsForActivation`, so that the market actor returns the deal weights to the miner while
activating deals (and also [supports batching](https://github.com/filecoin-project/specs-actors/issues/474)).
更改`market.ActiveDeals`的参数和返回类型。来匹配以前`market.VerifyDealsForActivation`的类型, 
当激活交易时以便市场参与者将交易权重返回给矿工，
以及[支持批处理](https://github.com/filecoin-project/specs-actors/issues/474)).

Change `miner.ProveReplicaUpdate` to similarly call `market.VerifyDealsForActivation` and 
`market.ActivateDeals`, obtaining the unsealed sector CID and deal weights from the new return values.

变更`miner.ProveReplicaUpdate`类似调用`market.VerifyDealsForActivation`和`market.ActivateDeals`, 
从新的返回值获得未密封的扇区CID和交易权重。

## Design Rationale 设计原理
This value for the pre-commit deposit is simple to calculate, 
and is always less than the value due for the sector's initial pledge.
This means that the provider will have to lock at least this much anyway by the time the sector is proven.
Since the window between pre-commit and prove-commit is only just over 1 day,
this represents a very small additional opportunity cost of locked funds for the provider.

这个预提交保证金的值易于计算，
并且总是低于该扇区初始承诺的到期价值。
这意味着，当该扇区被证明时，提供商无论如何都必须锁定至少这么多。
由于预提交和证明提交之间的窗口仅略多于1天，
这意味着提供商锁定资金的额外机会成本非常小。

As of January 2022, the calculations are roughly:
```
EpochReward := 22.54 * 5 = 112.25
NetworkPower := 16.0954 * 10^9
CirculatingSupply := 229 * 10^6

// Sector quality = 1
StoragePledge := 0.0128
ConsensusPledge := 0.1365
PreCommitDeposit := StoragePledge = 0.0128
InitialPledge := StoragePledge + ConsensusPledge = 0.1494

// Sector quality = 10 (values are appox 10x greater)
StoragePledge := 0.1285
ConsensusPledge := 1.3658
PreCommitDeposit := StoragePledge = 0.1285
InitialPledge := StoragePledge + ConsensusPledge = 1.4944
```

Note that the pre-commit deposit for sector quality = 10 is less than (86% of) the initial pledge
for a sector with quality = 1, so PCD is always less than initial pledge.

请注意，扇区质量=10的预承诺保证金低于初始承诺的86%
对于质量=1的部门，因此PCD始终低于初始承诺。

This relationship depends on the consensus pledge (a function of the circulating money supply)
being much larger than the storage pledge (a function of per-sector expected rewards).
We expect this relationship to continue to hold, as  
这种关系取决于共识承诺（流通货币供应量的函数）
远大于存储承诺（一个每个扇区预期回报的函数）。
我们预计这种关系将持续下去，因为
(1) the money supply is expands over time with long-term vesting and emissions, and
(1) 随着时间的推移，货币供应量随着长期归属和排放量的增加而增加，以及
(2) the expected reward per sector falls over time with growth in network capacity and decaying block reward.
(2) 随着网络容量的增长和区块奖励的衰减，每个扇区的预期回报会随着时间的推移而下降。

## Backwards Compatibility 向后兼容性
This proposal changes the behaviour of the built-in storage miner and storage market actors,
and so requires a network upgrade.

本提案改变了内置存储矿工和存储市场参与者的行为，
因此需要网络升级。

This proposal changes the schema of `miner.SectorPreCommitInfo`, and so requires a state migration
for in-flight pre-commits at the time of the upgrade.
This migration must calculate the unsealed CID for those sectors by loading the relevant deal proposals
in the same way that the new code will do so at sector pre-commit.

这提案改变了`miner.SectorPreCommitInfo`的模式。
因此需要在升级时进行状态迁移以进行飞行中的预提交。
此迁移必须通过加载相关交易提案来计算这些行业的未密封CID，
其方式与新代码在行业预提交时的方式相同。

This proposal changes only APIs for inter-actor calls, not those invoked by top-level messages,
so APIs remain backwards compatible with existing external callers.

本提案仅更改用于参与者间调用的API，而不是顶级消息调用的API，
因此，API与现有外部调用者保持向后兼容。

## Test Cases
To be provided with implementation.

## Security Considerations 安全考虑
The pre-commit deposit provides as a disincentive to providers to attempt to cheat proof of replication.
This proposal makes the pre-commit deposit for committed-capacity sectors larger than the current value,
which restores the buffers to security inadvertantly reduced in FIP-0019.
This deposit is still smaller than the sector initial pledge due at PoRep.

预提交保证金可阻止提供商试图欺骗复制证明。
本提议使得承诺容量扇区的预承诺存款大于当前值，
这将缓冲区恢复到FIP-0019中意外减少的安全性。
该保证金仍低于在PoRep到期的扇区初始质押。

## Incentive Considerations 激励因素
A larger pre-commit deposit increases the funds at risk in case a provider fails to follow through
with a sector commitment, such as in case of an operational failure.
If a provider expects failed pre-commitments of committed-capacity sectors to be a significant part of operational costs,
this proposal will increase that cost by up to a factor of 10.
This proposal retains ergonomic validity checks on deals at pre-commit in order to minimise the possibility
of operator error in specifying deals to result in loss of pre-commit deposit.

较大的预承诺保证金会增加资金风险，以防提供商未能跟进
具有一个扇区的承诺，例如在操作故障的情况下。
如果供应商预计承诺容量的扇区未兑现预承诺将成为运营成本的重要部分，
这项提议将使该成本增加10倍。
该提案保留了在提交前对交易进行的人体工程学有效性检查，以最大限度地降低
操作员在指定交易时出错的可能性而导致损失预提交保证金。

## Product Considerations 产品考虑
As noted above, this proposal slightly increases the amount of time that some funds are locked by the miner actor.
In practise, we expect most providers deposit at least the expected initial pledge value
up front when pre-commiting a sector, in which case this proposal requires no operational change.

如上所述，本提议略微增加了时间总和, 而一些资金被矿工参与者锁定。
在实践中，我们希望大多数供应商至少存入预期的初始质押价值
在预委托一个预承诺扇区时，在这种情况下，该提案不需要任何运营变更。

## Implementation
Prototype implementation was provided in Go in https://github.com/filecoin-project/specs-actors/pull/1575.

Slighly improved implementation for Rust built-in actors in https://github.com/filecoin-project/builtin-actors/pull/484.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
