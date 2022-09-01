---
fip: "0033"
title: Explicit premium for FIL+ verified deals
author: Alex North (@anorth)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/243
status: Deferred
type: Technical (Core)
created: 2022-01-12
---

This FIP is not yet 100% complete, but is substantial enough to warrant community involvement prior to completion.
Further development is deferred until after the FVM and associated built-in actor architectural changes are completed. 

该FIP尚未100%完成，但足够大，足以保证在完成之前社区参与。
进一步的开发将推迟到FVM和相关的内置参与者架构更改完成之后。

Outstanding items:  
未决项目：
- A scheme for withholding deal reward from faulty sectors and enacting penalties
- 从失败的扇区扣留交易奖励并实施处罚的计划
- Implementation details for pledge and penalties
- 质押及处罚实施细则
- Validation of the reward, pledge, and penalty function behaviour and ~equivalence with existing implementation.
- 奖励、承诺和惩罚函数行为的验证以及与现有实现的等价性。
- More detailed spec of migration logic
- 更详细的迁移逻辑规范
- Decide what to do with unclaimed rewards
- 决定如何处理无人领取的奖励


## Simple Summary 简述
Removes the concept of quality-adjusted power from the storage powered consensus and instead 
redirects part of the block reward as an explicit premium paid to providers of Filecoin Plus verified deals.
The rewards paid remain the same, though with timing more advantageous to the provider.

从存储驱动共识中删除了质量调整过存力的概念，而是将部分区块奖励重定向为支付给Filecoin Plus验证交易提供商的明确溢价。
支付的奖励保持不变，尽管时机对提供商更有利。

## Abstract 摘要
The Filecoin network is secured by continually proven storage capacity, for which providers (miners) earn block reward.
To incentivise the storage of useful data, the Filecoin Plus program rewards the storage of verified deals
by causing such committed space to count for 10 times the storage power of other capacity. 
This is the quality-adjusted power mechanism.
The increased power increases the possibility of winning block rewards by a factor of 10.

Filecoin网络通过不断验证的存储容量得到保护，供应商（矿工）因此获得区块奖励。
为了激励有用数据的存储，Filecoin Plus计划通过使此类承诺空间的存储能力达到其他容量的10倍来奖励已验证交易的存储。
这是质量调整过的存力机制。
增加的存力将赢得区块奖励的可能性增加10倍。

QA power couples the storage deal market and the content of individual sectors to consensus.
This coupling greatly constrains the design space of both storage accounting and deal markets, and  
QA存力将存储交易市场和各个扇区的内容结合到共识中。
这种耦合极大地限制了存储会计和交易市场的设计空间，并且
- delays rewards for hosting verified deals, and makes those rewards uneven and unreliable;
- 延迟对托管验证交易的奖励，并使这些奖励不均衡和不可靠；
- confuses FIL+ performance and reputation with raw capacity;
- 将FIL+性能和声誉与原始容量混淆；
- prevents the flexible use of sector space to serve deals, restricting storage semantics to "write-once";
- 阻止灵活使用扇区空间服务交易，将存储语义限制为“一次写入”；
- will impede the development of alternative storage markets once the FVM enables user-programmable contracts; and
- 一旦FVM启用用户可编程合同，将阻碍替代存储市场的发展；和
- impedes the design of more scalable sector accounting mechanisms.
- 阻碍了更可扩展的扇区帐号机制的设计。

The proposal breaks the coupling by accounting for and rewarding Filecoin Plus verified deals explicitly,
rather than piggybacking on the storage power block reward. In brief:

该提案打破了这种耦合，明确说明并奖励Filecoin Plus验证交易，而不是依赖于存储能力块奖励。简言之：

- Remove per-sector on-chain sector quality information from state; use raw-byte power for consensus power.
- 从状态中删除链上每个扇区的质量信息；使用原始字节存力作为共识存力。
- Account for verified deal space-time directly in the market actor.
- 直接在市场参与者中说明已验证的交易时空。
- Split the block reward into consensus reward and verified deal premium, paid separately.
- 将区块奖励分成共识奖励和验证交易溢价，分别支付。
- Add pledge and penalty mechanisms to verified deals to retain the existing economic incentives.
- 在已验证的交易中添加质押和惩罚机制，以保留现有的经济激励。

## Change Motivation 变更动机
The [FVM](https://github.com/filecoin-project/FIPs/issues/113) will enable
user-defined contracts to execute in the Filecoin state machine.
Our hopes for these contracts include innovations on storage markets beyond the built-in one,
supporting novel types of storage deals and functionality including
deal extension, multi-sector deals, re-negotiation, deal transfer, capacity deals, auction markets,
repair markets, insurance, and derivatives.
But today, individual sector content is coupled to consensus (through QA power) and
the storage miner actor is tightly coupled to the built-in storage market actor
(e.g. the miner actor calls the market actor to compute deal weight).
Such coupling is a significant barrier to feature development in storage markets,
preventing straightforward implementation of even simple features like time-extension of deals or moving deal data between sectors.
Even if we had the FVM, most of this functionality would be either impossible or very complex to
design and build within the current architecture.

[FVM](https://github.com/filecoin-project/FIPs/issues/113)将允许用户定义的合约在Filecoin状态机中执行。
我们对这些契约的希望包括对存储市场的创新，而不是内置的，支持新型存储交易和功能，包括交易扩展、多扇区交易、重新谈判、交易转移、容量交易、拍卖市场、维修市场、保险和衍生品。
但如今，单个扇区内容与共识（通过QA存力）耦合，而存储矿工参与者与内置存储市场参与者紧密耦合（例如，矿工参与者调用市场参与者来计算交易权重）。
这种耦合是存储市场中功能开发的一大障碍，阻碍了简单功能的直接实现，如交易时间延长或在扇区之间移动交易数据。
即使我们有FVM，大多数功能在当前架构中设计和构建都是不可能的或非常复杂的。

Improvements are sought within existing product and crypto-economic constraints around sound storage and economic security.
The proposal aims to preserve existing reward, pledge, and penalty behaviour as far as remains sensible
given the conceptual separation.

围绕声音存储和经济安全，在现有产品和加密经济约束范围内寻求改进。
该提案旨在在概念分离的情况下，尽可能合理地保留现有的奖励、承诺和惩罚行为。

### Improvements to Filecoin Plus Filecoin Plus的改进
This proposal enhances Filecoin Plus by improving the timing, predictability, and reliability of rewards. 

该方案通过改进奖励的时机、可预测性和可靠性来增强Filecoin Plus。

- With QA power, the reward for hosting verified deals is uneven (lottery), unreliable (need blocks included), and delayed.
- 使用QA存力，托管验证交易的回报是不均匀的（抽奖）、不可靠的（包括需要区块）和延迟的。
- FIL+ premium will be **more reliable**, especially for small providers, because payment doesn’t depend on 
winning the consensus lottery or, after winning, getting a block included in the chain.
- FIL+溢价将**更加可靠**，特别是对于小型供应商，因为支付不取决于赢得共识抽奖，也不取决于中奖后获得区块链中的区块。
- FIL+ premium **pays sooner**, aligned with the storage deal term. 
The QAP mechanism spreads reward out over the whole sector lifespan. 
This delays rewards, sometimes far after the deal has completed.
- FIL+溢价**支付更快**，与存储交易条款一致。
QAP机制在整个扇区生命周期内分散奖励。
这会延迟奖励，有时会在交易完成后很久。

### Improvements to programmability and utility 可编程性和实用性的改进
Decoupling sector content from consensus power makes storage much more flexible.
The ideal storage API would be a big fungible “disk”, where providers can add/remove/update storage freely to satisfy new deals.
QAP opposes altering data, because mechanism spreads out the deal’s influence on power over the sector’s lifetime, 
regardless of deal term.

将扇区内容与共识力量分离使存储更加灵活。
理想的存储API将是一个大的可替换“磁盘”，供应商可以在其中自由添加/删除/更新存储以满足新的交易。
QAP反对修改数据，因为无论交易期限如何，该机制都会将交易对扇区存力的影响扩展到整个扇区。

- With QAP, we can add new data to a sector, but cannot remove or replace data.
The provider would be either overpaid or underpaid, depending on deal term.
- 使用QAP，我们可以向扇区添加新数据，但不能删除或替换数据。
根据交易条款的不同，供应商的薪酬要么过高，要么过低。
- Expired data cannot be removed from a sector.
- 无法从扇区中删除过期数据。
- Data cannot be replaced even if the deal is complete.
- 即使交易完成，数据也无法替换。
- QAP restricts storage to **write-once**. A provider cannot utilise the full space-time of committed storage for deals; they can only use each byte of space once, regardless of deal duration.
- QAP将存储限制为**写一次**。提供商不能将承诺存储的全部时空用于交易；无论交易持续时间如何，他们只能使用每个字节的空间一次。

This inflexible storage limits feature development:  
这种不灵活的存储限制了功能开发：
- Extending deal by transferring to new sector: only possible exactly at end of original sector lifespan
- 通过转移到新扇区来延长交易：只有在原始扇区寿命结束时才可能
- Transferring deal to another provider: not possible except at sector expiration
- 将交易转移到其他提供商：不可能，除非在扇区到期时
- Taking new deals (”snap-on-snap”): not possible to re-use storage that was already used for one deal, even if the deal is complete.
Cannot have a short, full-sector deal, and then replace with another full-sector deal.
- 接受新交易（“快照”）：即使交易完成，也不可能重复使用已用于一笔交易的存储。
不能有一个短的、完整的扇区交易，然后用另一个完整扇区交易代替。
- Capacity deals: client cannot remove or replace data at will, though there might be specific cases that work.
- 容量交易：客户机不能随意删除或替换数据，尽管可能存在某些特定的情况。

Storage that is more flexible is more valuable. 
After decoupling, a provider can extract more reward per sector because they can use the full space-time for verified deals,
even if those deals are not known when the sector is committed.  
更灵活的存储更有价值。
在解耦之后，提供商可以在每个扇区提取更多的奖励，因为他们可以将整个时空用于验证交易，
即使这些交易不知道该扇区何时承诺。
- QAP inflexibility incentivises short sector commitments, because FIL+ rewards are delayed until sector expiration and
space-time after deal expiration cannot be re-used
- QAP的不灵活性激励了短期部门承诺，因为FIL+奖励延迟到部门到期，交易到期后的时空不能重复使用
- QAP disincentivizes extending sector lifespan because miner needs to re-seal in order to take new deals for the space.
- QAP不鼓励延长扇区寿命，因为矿工需要重新密封，以获得新的空间交易。

This proposal removes all such restrictions, making sector storage fungible and removing disincentives to long commitments.

本提案取消了所有此类限制，使扇区存储可替代，并消除了长期承诺的抑制因素。

### Enabling innovation 促进创新
The built-in storage market actor is privileged as the arbiter of deal weight, and hence QA power.
The market is consensus-critical, inefficient, and very slow/hard to change (requires network upgrade).
The miner actor is tightly coupled to this market actor:
e.g. the miner invokes and trusts the built-in market to compute weight/power.

内置存储市场参与者作为交易权重的仲裁者享有特权，因此拥有QA存力。
市场对共识至关重要，效率低下，而且非常缓慢/难以改变（需要网络升级）。
矿工参与者与该市场参与者紧密耦合：例如，矿工调用并信任内置市场来计算权重/存力。

This privilege precludes alternative markets built on the FVM from competing.
Alternative markets might be more efficient, offer different features, make different trade-offs, integrate with other chains, etc.
But if they can't serve Filecoin Plus deals, they will not be competitive.

这种特权排除了基于FVM的替代市场的竞争。
替代市场可能更有效率，提供不同的功能，做出不同的权衡，与其他链整合，等等。
但如果他们不能提供Filecoin Plus交易，他们就没有竞争力

By decoupling markets from consensus, this proposal  
通过将市场与共识脱钩
- reduces consensus-critical, slow-moving, network-level code,
- 减少共识关键、移动缓慢的网络级代码，
- reduces miner actor complexity,
- 降低了矿工参与者的复杂性，
- moves code out of the network-critical trust base,
- 将代码移出网络关键信任库，
- reduces the privilege of the built-in market actor, as a step towards realising markets as pure smart contracts,
evolving as fast as their independent development teams desire.
- 减少了内置市场参与者的特权，作为实现市场作为纯智能合约的一步，其发展速度与独立开发团队的期望一样快。

We must enable flexible storage and remove market actor privilege in order to support rapid ecosystem development of
actually-useful storage-related applications and features.

我们必须启用灵活存储并取消市场参与者特权，以支持实际有用的存储相关应用程序和功能的快速生态系统开发。

See [discussion #241](https://github.com/filecoin-project/FIPs/discussions/241) for discussion of the
more general motivations behind the innovation-enabling motivation for this and related upcoming proposals.

参见[讨论#241](https://github.com/filecoin-project/FIPs/discussions/241)讨论本提案
和相关即将提出的提案的创新激励背后的更多一般的动机。

### Scale and security 规模和安全
The concept of quality-adjusted power incurs blockchain state storage and processing costs
linear in the number of sectors proven.
Such linear data will eventually limit the capacity of the chain to account for proven storage.
Removal of quality-adjusted power opens up scalable representations of homogeneous sectors as aggregates.
While the scalability of committed storage is not a pressing problem today, reducing coupling will
set us on stronger footing for solving it in the future.

质量调整过的存力的概念导致区块链状态存储和处理成本与已验证的扇区数量呈线性关系。
这种线性数据最终将限制链的容量，以说明经验证的存储。
去除经质量调整过的存力会打开同质扇区作为集合的可扩展表示。
虽然承诺存储的可扩展性在今天不是一个紧迫的问题，但减少耦合将为我们在未来解决这一问题奠定更坚实的基础。

QA power also slightly undermines economic-cost arguments for network security. 
Decoupling Filecoin Plus rewards from consensus will restore the property that consensus power requires 
a linear investment in storage hardware, and prevent collusion with FIL+ notaries from potentially
subverting consensus (though this is not a practical concern today).

QA存力也略微削弱了网络安全的经济成本论证。
将Filecoin Plus奖励与共识分离将恢复共识力量需要对存储硬件进行线性投资的特性，并防止与FIL+公证人的勾结可能破坏共识（尽管这在今天不是一个实际问题）。

## Specification 规范

### Uniform sector power 统一扇区存力
Every sector has uniform power corresponding to its raw committed storage size, regardless of the presence of deals. 
This **decouples the concept of sector quality from consensus**, 
and makes all bytes equal in the eyes of the power table. 
Network power and committed storage are now the same concept and need not be tracked separately by the power actor.

无论是否存在交易，每个扇区都具有与其原始承诺存储大小相对应的统一存力。
这**将扇区质量的概念与共识**分离，并使所有字节在功率表中都相等。
网络存力和承诺存储现在是同一概念，不需要由存力参与者单独跟踪。

- Remove `DealWeight` and `VerifiedDealWeight` fields from `miner.SectorOnChainInfo`.
- 从`miner.SectorOnChainInfo`中删除`DealWeight`和`VerifiedDealWeight`字段。
- Remove call from miner to storage market actor `VerifyDealsForActivation` during sector pre-commitment
  or replica update (the method is misnamed: it actually computes deal weight, and real verification happens later).
  Remove the method itself from the storage market actor.
- 在扇区预承诺或副本更新期间，删除miner对存储市场参与者`VerifyDealsForActivation`的调用（该方法命名错误：它实际上计算交易权重，实际验证稍后进行）。
从存储市场参与者中删除方法本身。
- Remove `TotalQualityAdjPower`, `TotalQABytesCommitted`, `ThisEpochQualityAdjPower`, `ThisEpochQAPowerSmoothed` from `power.State`.
- 从`power.State`移除`TotalQualityAdjPower`, `TotalQABytesCommitted`, `ThisEpochQualityAdjPower`, `ThisEpochQAPowerSmoothed`.
- Add `ThisEpochRawBytePowerSmoothed` to power.State and maintain it in a similar fashion to `ThisEpochQAPowerSmoothed`.
- 将`ThisEpochRawBytePowerSmoothed`添加到power.State和以类似于`ThisEpochQaPowerMooted`的方式维护。
- Remove `QualityAdjPower` from `power.Claim`.
- 从`power.Claim`移除`QualityAdjPower`.
- Remove all calculations leading to these values, and replace their use as inputs to calculations 
  with the equivalent raw byte size/power values.
- 删除导致这些值的所有计算，并用等效原始字节大小/存力值替换它们作为计算输入。

With uniform sector power, the power of groups of sectors may be trivially calculated by multiplication.
Window PoSt deadline and partition metadata no longer need to maintain values for live, unproven, 
faulty and recovering power, but the sector number bitfields remain.
The complexity of code and scope for error in these various derived data is reduced.

对于均匀扇区存力，可以通过乘法简单地计算扇区组的存力。
窗口发布截止日期和分区元数据不再需要维护活动、未经验证、故障和恢复存力的值，但扇区号位字段仍保留。
减少了代码的复杂性和这些不同派生数据中的错误范围。

- Remove `LivePower`, `UnprovenPower`, `FaultyPower` and `RecoveringPower` from `miner.Partition`, 
  along with the code paths that calculate these values.
- 从`miner.Partition`删除`LivePower`, `UnprovenPower`, `FaultyPower` and `RecoveringPower`,
 沿着这些代码路径计算这些值。 
- Re-code accesses of these memoized power values to instead multiply a sector count by sector size.
- 重新编码这些存储存力值的访问，以扇区大小来替换扇区计数乘数。

_Note:_ this proposal does leave per-sector metadata on chain, including activation/expiration epochs and deal IDs. 
It will still be necessary to load sector metadata when processing faults in order to schedule sector expiration. 
This might be removed by future changes toward fungible sectors.

_注：_本方案确实将每个扇区的元数据保留在链上，包括激活/到期时间和交易ID。
在处理故障时，仍然需要加载扇区元数据，以便安排扇区过期。这可能会被未来转向可替代扇区的变化所移除。

_Note:_ Uniform power also means that all similarly-sized sectors committed at the same epoch would require the same initial pledge. 
Similarly, the expected day reward and storage pledge metadata values depend only on activation epoch. 
The per-sector values maintained to track penalties for replaced CC sectors (from pre-SnapDeals) become technically redundant. 
We could maintain historical pledge/reward values for the network just in the power and reward actors 
(instead of storing each of these numbers in chain state hundreds of times per epoch).
Removing these per-sector values would improve scalability of sector accounting, but this proposal excludes those changes
in the name of making a smaller total change while still achieving the decoupling of deals from consensus.

_注：_统一存力也意味着在同一时期承诺的所有类似规模的扇区将需要相同的初始承诺。
类似地，预期的日奖励和存储承诺元数据值仅取决于激活时间。
为跟踪被替换CC扇区（来自pre-SnapDeals之前）的惩罚而保持的每个扇区值在技术上变得多余。
我们可以仅在存力和奖励参与者中维护网络的历史承诺/奖励值（而不是在每个历元中以链状态存储这些数字数百次）。
删除这些每个扇区的价值将提高扇区帐号的可伸缩性，但本提案以较小的总变化的名义排除了这些变化，同时仍实现了交易与共识的脱钩。

### Storage market actor accounts for verified deal space-time 存储市场参与者负责验证交易时空
The storage market actor becomes responsible for accounting for providers' storage of verified deals, 
and providing metrics to the reward actor to guide reward distribution.

存储市场参与者负责计算供应商对已验证交易的存储，并向奖励参与者提供指标以指导奖励分配。

To the storage market actor:  
对于存储市场参与者：
- Add `TotalVerifiedSpace`, a total of active verified deal space at the end of the last epoch. 
  Maintain this value during processing of deal activation and termination, 
  and when the market actor is notified of temporary faults of the sectors hosting verified deals.
- 添加`TotalVerifiedSpace`，即上一个历元结束时活动验证交易空间的总和。
  在交易激活和终止处理过程中，以及当市场参与者收到托管验证交易的扇区的临时故障通知时，保持该值。
- Add `TotalVerifiedSpaceDeltas`, a queue (AMT of epoch -> BigInt) to deltas to 
  the total verified space indexed by the current or future epoch at which they take effect.
- 将`TotalVerifiedSpaceDeltas`添加到由当前或未来生效的历元索引的总已验证空间的增量中，这是一个队列（历元的AMT -> BigInt）。
- Add `VerifiedRewardsHistory` (AMT of epoch -> pair), a fixed-size queue of pairs of `TotalVerifiedSpace` 
  and the verified deal reward amount paid to the market actor (see below) from the end of each epoch.
  The window may be quite large, say 30-60 days. 
  TODO: consider whether a different structure might be more efficient for fixed-size, dense array
- 添加`VerifiedRewardsHistory`(历元ATM -> 对)，一个固定大小的`TotalVerifiedSpace`对队列，
  以及从每个历元结束时支付给市场参与者的已验证交易奖励金额（见下文）。
  窗口可能相当大，例如30-60天。
  TODO:考虑不同的结构是否对固定大小的密集阵列更有效
- Add `ProviderVerifiedClaims` a map (HAMT) of per-provider verified deal metadata. For each provider, maintain:
  - `LastClaimEpoch`: the epoch up until which rewards were last processed for the provider.
  - `VerifiedSpace`: the provider's active verified deal space at the end of LastClaimEpoch.
  - `VerifiedSpaceDeltas`: a queue of deltas since LastClaimEpoch, the provider's version of TotalVerifiedSpaceDeltas.
- 添加`ProviderVerifiedClaims`每个提供商验证的交易元数据的映射（HAMT）。对于每个提供商，维护：
  - `LastClaimEpoch`：供应商最后一次处理奖励的时间。
  - `VerifiedSpace`：在LastClaimEpoch结束时提供商的活动验证交易空间。
  - `VerifiedSpaceDeltas`：自LastClaimEpoch以来的deltas队列，提供程序的TotalVerifiedSpace deltas版本。
- Add `MigratedVerifiedDeals`, a map of deal ID to the size and start/end epoch as computed by the 
  (deprecated) quality-adjusted power mechanism for each existing verified deal.
  This is populated once at migration and will eventually become redundant.
- 添加“MigratedVerifiedDeals”，这是一个交易ID到大小和开始/结束时间的映射，
  由每个现有已验证交易的（不推荐的）质量调整过的存力机制计算。
  这在迁移时填充一次，最终将成为冗余。

When a verified deal is activated (on or before its start epoch), 
the deal's size is added to the `TotalVerifiedSpaceDeltas` queue entry for the start epoch, 
and subtracted from the entry for the end epoch. 
Similarly, the deltas are recorded in the provider's `VerifiedSpaceDeltas` queue. 
During cron at the end of each epoch, the queue entry for that epoch is removed and the value (which may be negative) 
added to `TotalVerifiedSpace`. The sum of `TotalVerifiedSpace` plus all deltas should always equal zero, 
the rolling sum never dipping below zero.

当激活已验证的交易时（在其开始时期时或之前），该交易的大小将添加到开始时期的`TotalVerifiedSpaceDelta`队列条目中，并从结束时期的条目中减去。
类似地，增量记录在提供者的`VerifiedSpaceDelta`队列中。
在每个历元结束时的cron期间，删除该历元的队列条目，并将该值（可能为负值）添加到`TotalVerifiedSpace`。`TotalVerifiedSpace`加上所有增量的总和应始终等于零，滚动总和不得低于零。

Note that it is important that the `TotalVerifiedSpaceDeltas` queue entries are fixed size (a single big integer). 
This allows us to schedule the deltas exactly on the deal start/end epoch, 
retaining an exact calculation for `TotalVerifiedSpace`, without exposing the potential for some party to
overload cron processing at a single epoch by stacking deals with the same start or end epoch.

请注意，`TotalVerifiedSpaceDeltas`队列条目的大小是固定的（单个大整数），这一点很重要。
这使我们能够在交易开始/结束时间段精确地调度增量，保留对`TotalVerifiedSpace`的精确计算，
而不会暴露出某一方通过在同一开始或结束时间段堆叠交易而在单个时间段过载cron处理的可能性。

See the section on faults for an explanation of `MigratedVerifiedDeals`.

有关`MigratedVerifiedDeals`的解释，请参阅故障部分。

To the storage market actor:  
对于存储市场参与者：
- Add a `ClaimRewards` method which traverses a provider's verified space deltas since last claim,
  along with the `VerifiedRewardsHistory`, and accumulates the reward due to a specific miner.
  It then invokes `miner.ApplyRewards` with that amount to transfer it to the miner's locked funds.
- 添加一个`ClaimRewards`方法，该方法遍历自上次索赔以来提供商的验证空间增量，以及`VerifiedRewardsStory`，并累积特定矿工的奖励。
  然后调用`miner.ApplyRewards`使用该金额将其转移到矿工的锁定资金中。

Each provider's claim of their share of verified space, and hence rewards, 
is processed manually by an invocation from the provider, rather than by cron. 
For each epoch since `LastClaimEpoch`, pop and add the corresponding delta to the provider's verified space,
then compute the fraction of total verified space at that epoch which the provider provided, and thus their share of the epoch's reward.
Accumulate these rewards up to the penultimate epoch, and pay to the provider via `miner.ApplyRewards`.

每个提供者对其已验证空间份额的声明，以及由此产生的奖励，都是通过提供者的调用而不是通过cron手动处理的。
对于自`LastClaimEpoch`以来的每个历元，弹出并将相应的增量添加到提供商的验证空间，然后计算提供商在该历元提供的总验证空间的分数，从而计算他们在历元奖励中的份额。
将这些奖励累积到倒数第二个纪元，并通过`miner.ApplyWards`支付给提供商。

Note that since the `VerifiedRewardsHistory` window is fixed, a provider which does not claim rewards within that window will forfeit them.

请注意，由于“VerifiedRewardsHistory”窗口是固定的，因此在该窗口内未申请奖励的提供商将被没收。

Parameter: verified reward history length: 60 days?

参数：验证奖励历史长度：60天？

### Reward distribution 奖励分配
The block reward at each epoch is divided into two shares:  
每个历元的区块奖励分为两部分：
- Consensus reward, proportional to raw byte power; and
- 共识奖励，与原始字节功率成比例；以及
- Verified deal premium, proportional to verified deal space.
- 验证交易溢价，与验证交易空间成比例。

A miner's chance of winning the election at each epoch is proportional to their share of the raw byte power 
(no quality-adjusted power) committed to the network and non-faulty.
Winning the consensus reward remains a lottery, and is paid directly to the miner actor.

矿工在每个时期赢得选举的机会与他们在网络和无故障中所占的原始字节存力（无质量调整存力）成比例。
赢得共识奖励仍然是一种抽奖，直接支付给矿工参与者。

The verified deal premium is paid to market actor, for subsequent distribution to providers. 
This reward is not a lottery, and is claimable by providers whether or not they produce blockchain blocks.

经核实的交易溢价将支付给市场参与者，以便随后分配给供应商。
这种奖励不是彩票，无论供应商是否生产区块链区块，供应商都可以申请。

To the reward actor:  
奖励参与者：
- Add `ClaimVerifiedDealReward`, to be invoked by the market actor, 
  which records the current total verified space for the next epoch, and remits the reward for the previous one.
- 添加`ClaimVerifiedDealReward`，由市场参与者调用，
  其记录下一个历元的当前总验证空间并汇出前一个历次的奖励。
- Rework the `UpdateNetworkKPI` implementation to compute the share of the next epoch's reward to be reserved for verified deal premium.
- 修改`UpdateNetworkKPI`实现，以计算下一个历元的奖励份额，该份额将保留给已验证的交易溢价。

In cron at the end of each epoch, the market actor calls `ClaimVerifiedDealReward`. 
This serves two purposes. 
在cron中，在每个时代结束时，市场参与者称为“ClaimVerifiedDealReward”。
这有两个目的。
(1) It provides the reward actor with the total verified space value for this epoch,
which will be used to compute the share of reward for the next epoch (similar to the current storage power from the power actor). 
(1) 它向奖励参与者提供该历元的总验证空间值，该值将用于计算下一历元的奖励份额（类似于存力参与者的当前存储能力）。
(2) In response, the reward actor pays to the market actor the share of total mining reward due to verified deals from the epoch just passed.
(2) 作为回应，奖励行为人向市场行为人支付因刚刚过去的时代验证的交易而获得的总采矿奖励份额。


After receiving both the storage power and verified space values, 
the reward actor splits the total block reward to be paid in the next epoch into 
the consensus reward and the verified deal premium.
The block reward is paid as usual, and the deal premium retained for the market actor to claim.
This mechanism will generalise to multiple market actors in the future (on the assumption they can use cron).

在接收到存储功率和验证的空间值之后，奖励参与者将下一个历元中要支付的总区块奖励分成共识奖励和验证的交易溢价。
区块奖励照常支付，交易溢价保留给市场参与者索赔。
该机制将在未来推广到多个市场参与者（假设他们可以使用cron）。

The reward split function is given by:  
奖励分割函数由下式给出：
```
ConsensusReward = EpochReward * NetworkRawPower / (NetworkRawPower + 9 * TotalVerifiedSpace)
VerifiedDealReward = EpochReward - ConsensusReward = 9 * NetworkRawPower * TotalVerifiedSpace / (NetworkRawPower + 9 * TotalVerifiedSpace)
```

On the assumption of smoothly changing verified space as a proportion of network power,
this reward distribution results in the *same total reward* profile as the current implementation,
both for the system as a whole and for individual miners, with the exception of a change in the timing
of verified deal rewards. This proposal has no intention of altering the ultimate incentives to storage providers,
only of providing them by a more transparent mechanism.

假设验证空间与网络存力成比例平稳变化，
这种在**相同总奖励**配置中的奖励分配结果作为当前实现，
无论是整体还是单个矿工系统，除了验证交易奖励的时间发生变化外。
本提案无意改变对存储提供商的最终激励，只是通过更透明的机制为其提供激励。

### Sector pledge 扇区抵押
The current sector initial pledge held by each miner actor comprises two parts:  
各矿工参与者持有的当前扇区初始抵押包括两部分：
- Sector storage pledge: a multiple of the reward expected to be earned by newly-committed power
- 扇区存储承诺：新承诺的存力预计将获得的奖励的倍数
- Sector consensus pledge: a pro-rata (by QA power) fraction of the circulating money supply when the sector is committed
- 扇区共识承诺：当扇区承诺时，按比例（通过QA存力）分配流通货币供应量的一部分

For each of these parts, reduce the sector pledge proportionally with the ratio of consensus reward to total reward,
so that the sector pledge is aligned with the sector's rewards. 
Pledge corresponding to the verified deal reward will be held by the market actor (see below) 
so that the total pledge amount remains consistent with the QA-power approach.

对于这些部分中的每一部分，按照共识奖励与总奖励的比率按比例减少扇区承诺，以便扇区承诺与扇区奖励保持一致。
与经验证的交易奖励相对应的质押将由市场参与者持有（见下文），以使总质押金额与QA存力方法保持一致。

The sector storage pledge becomes a multiple of the projected consensus block reward for the sector (i.e. excluding verified deal reward). 
The smaller network power denominator is balanced by a reduced share of total reward allocated to consensus rewards.

扇区存储承诺成为该扇区预计共识区块奖励的倍数（即，不包括验证交易奖励）。
较小的网络存力分母由分配给共识奖励的总奖励的减少份额来平衡。

For a sector with no verified deals, this should yield approximately the same storage pledge amount as before this proposal 
(and continue to function if the consensus reward ratio changes smoothly).
对于没有验证交易的扇区，这将产生与本提案之前大致相同的存储质押金额（如果共识奖励比率平稳变化，则继续发挥作用）。

Approximately (the real version uses an alpha/beta smoothing function):  
近似（实际版本使用alpha/beta平滑函数）：
```
SectorStoragePledge := ProjectionPeriod * ExpectedReward
where
ExpectedReward := ConsensusRewardEstimate * SectorPower / NetworkPower
ConsensusRewardEstimate := TotalRewardEstimate * ConsensusRewardRatio
```

TODO: show demonstration of the equivalence in behaviour using real network values.

TODO：使用真实网络值演示行为的等效性。

For sector consensus pledge, the “lock target” of which the sector takes a pro-rata share is multiplied (i.e. reduced)
by ratio of consensus reward to total block reward. The calculation must include as input the current reward split function, 
and set as pledge the pro-rata share of money supply according to the instantaneous share of total reward to be earned by the raw byte power.
The remainder of the total consensus pledge will be met by a verified deal pledge.

对于扇区共识抵押，该扇区按比例获得份额的“锁定目标”乘以（即减少）共识奖励与总区块奖励的比率。计算必须包括当前奖励分割函数作为输入，并根据原始字节幂将获得的总奖励的瞬时份额，将按比例的货币供应份额设置为质押。全部共识承诺的其余部分将通过经验证的交易承诺来实现。

Approximately:  
大约：
```
SectorConsensusPledge := LockTarget * PledgeShare
where
LockTarget := 0.3 * ConsensusRewardRatio
PledgeShare := SectorPower / NetworkPower
```

As at present, the sector pledge is forfeit when a sector terminates ahead of schedule.

目前，如果某个扇区提前终止，则该扇区质押将被没收。

### Verified deal pledge 验证过的交易抵押
In order to maintain the current money supply relationships, storage providers must pledge collateral
with the storage market as a function of their expected deal premium rewards, as well as sector rewards.
This pledge secures verified deals, rather than sectors.

为了维持当前的货币供应关系，存储提供商必须根据其预期的交易溢价回报以及扇区回报向存储市场抵押抵押品。
这一抵押保证了经过验证的交易，而不是扇区。

We wish to maintain approximately the same total pledge requirement for a sector+deals,
meaning the deal pledge should match the reduction in sector pledge caused by removing QA power.
Thus, it takes a very similar form:  
我们希望保持一个扇区+交易的总质押要求大致相同，这意味着交易质押应与取消QA存力导致的行业质押减少相匹配。
因此，它采用非常相似的形式：
- Deal storage pledge: a multiple of rewards expected to be earned by the deal premium
- 交易存储抵押：一个交易溢价预期获得的奖励的倍数
- Deal consensus pledge: a pro-rata fraction of the circulating supply when the deal is activated
- 交易共识抵押：交易启动时按比例分配的流通供应量

For the deal storage pledge, a projection function similar to that used to project sector rewards may be used
to project a multiple of the expected reward to be earned by a verified deal.

对于交易存储质押，类似于用于预测扇区奖励的预测函数，可用于预测验证交易将获得的预期奖励的倍数。

Approximately  
近似地
```
DealStoragePledge := ProjectionPeriod * ExpectedReward
where
ExpectedReward := DealRewardEstimate * DealSpace / TotalDealSpace
DealRewardEstimate := TotalRewardEstimate * (1 - ConsensusRewardRatio)
```

Similarly, the deal consensus pledge is a pro-rata share of the money supply, moderated by the 
instantaneous share of total reward emissions that the deal can claim.

类似地，交易共识抵押是按比例分配的货币供应量份额，由交易可以声称的总回报排放的瞬时份额调节。

Approximately:  
近似地: 
```
DealConsensusPledge := LockTarget * PledgeShare
where
LockTarget := 0.3 * (1 - ConsensusRewardRatio)
PledgeShare := DealSpace / TotalDealSpace
```

TODO: describe market state changes for tracking pledge

TODO:描述跟踪质押的市场状态变化

The deal pledge is forfeit (burned) if a provider fails to carry a deal to completion.

如果提供商未能完成交易，则交易质押将被没收（烧毁）。

### Faults and penalties 过失和处罚
(Coming soon)

Notify the market of deal faults.
Add penalties to match the existing penalty scheme.

## Design Rationale 设计原理

### Separation of security from reward distribution 安全与奖励分配的分离
The current mechanism of quality-adjusted power is a means of incentivising useful storage, 
delegating the definition of “useful” to an off-chain notary.
The incentive for storing verified deals is the increased block reward that may be earned from the increased power of those sectors,
despite constant physical storage and infrastructure costs. 
In essence, **the network subsidises useful storage by taxing the other power**.

当前的质量调整过的存力机制是一种激励有用存储的手段，将“有用”的定义委托给链外公证人。
存储经验证的交易的动机是，尽管物理存储和基础设施成本不变，但这些扇区的电力增加可能会带来更多的区块奖励。
本质上，**网络通过向其他存力征税来补贴有用的存储**。

Storage power thus has two roles, currently coupled:
因此，存储能力具有两个角色，目前耦合：
- to secure the network through raising the economic cost of some party maintaining a significant fraction of the total power, and
- 通过提高某一方维护总存力的一大部分的经济成本来保护网络，以及
- to determine the distribution of block rewards.
- 以确定区块奖励的分布。

The block reward currently both rewards security and subsidises useful storage. 
This coupling theoretically means the verified deal power boost reduces the hardware cost to attack the network by a factor of 10,
if colluding notaries would bless all of a malicious provider's storage (this is an impractical attack today, though).

区块奖励目前既奖励安全，又补贴有用的存储。
这种耦合在理论上意味着，如果合谋的公证人能够保护恶意提供商的所有存储，则经验证的交易存力提升将攻击网络的硬件成本降低10倍（尽管这在今天是一种不切实际的攻击）。

This proposal **makes the reward direct**, reducing complexity and coupling between the storage market and individual sectors,
and clearly separating security from reward distribution policy.

本提案**使用奖励直达**，降低了存储市场和各个扇区之间的复杂性和耦合，并将安全性与奖励分配政策明确分开。

This means that verified deals no longer contribute to consensus power, though they continue to earn the 
same (in fact, more reliable and sooner) rewards as the QA-power mechanism. This could constitute a reduction
in incentive to maintain verified deals to the extent that consensus power is desirable for reasons _other
than the reward_. Such reasons might include 
(b) control, if providers manipulate the blockchain outside the "honest" protocol, for self-interested reasons. 
To the extent that this proposal reduces the ability to stack consensus power through 
defeating the Filecoin Plus notary process, it confers a chain security benefit.

这意味着，经过验证的交易不再有助于达成共识，尽管它们继续获得与QA存力机制相同的（事实上，更可靠、更快）回报。如果出于_奖励以外_的原因需要共识，这可能会降低维持经验证交易的动机。这些原因可能包括：
(a) 声誉，如果供应商在存储上依附于共识存力而不是有用数据的存储，或
(b) 控制，如果提供商出于自身利益的原因在“诚实”协议之外操纵区块链。
在某种程度上，本提案通过挫败Filecoin Plus公证程序而降低了堆叠共识存力的能力，从而带来了链安全优势。 

### Change in reward smoothness 奖励平滑度的变化
A storage provider's consensus reward is earned via a repeated lottery, so it is not smooth.
In the long run, the expected value is quite stable, but in the short run may vary a lot.

存储提供商的共识奖励是通过重复抽奖获得的，因此这并不顺利。从长期来看，预期值相当稳定，但从短期来看，可能会有很大变化。

This proposal makes the deal reward smooth.
A provider earns deal reward consistently, without regard to the consensus lottery.
A provider need not mine any blockchain blocks to earn the deal reward, even if they do win the lottery.
这一提议使交易奖励顺利进行。
提供商始终获得交易奖励，而不考虑共识彩票。
提供商不需要挖掘任何区块链区块来获得交易奖励，即使他们确实中了彩票。

This is an improvement in the economics for small providers, 
who account for only a tiny fraction of network power but tend to disproportionately focus on providing
high quality storage for verified deals. 
Such providers very rarely win the consensus lottery in any case,
and their smaller scale permits less sophisticated blockchain node infrastructure and connectivity to capitalize on those infrequent wins.
A provider focussing on sectors with high verified deal content can thus earn 90% of their total reward smoothly and predictably.

这对于小型供应商来说是一种经济上的进步，它们只占网络存力的一小部分，但往往不成比例地专注于为经验证的交易提供高质量存储。
这些提供商在任何情况下都很少赢得共识抽奖，其规模较小，允许不太复杂的区块链节点基础设施和连接利用这些不常见的赢家。
因此，专注于具有高验证交易内容的扇区的提供商可以顺利且可预测地获得其总回报的90%。

### Change in reward timing 奖励时机的变化
This proposal effects a change in the timing of deal rewards.
In the quality-adjusted power model, the reward attributable to an individual verified deal is 
spread out over the whole lifetime of the sector supporting that deal. 
If a deal occupies a small fraction of a sector's lifetime, much of the reward is delayed well past the deal's expiration.

该提议改变了交易奖励的时间安排。
在经质量调整过的存力模型中，可归因于单个经验证交易的奖励分布在支持该交易的扇区的整个生命周期内。
如果一笔交易占用了一个扇区生命周期的一小部分，那么大部分回报都会延迟到交易到期之后。

The deal reward is also payable prior to the initial Window PoSt proof at which a newly-committed sector
first gains power.

交易奖励也应在新承诺扇区首次获得权力的初始窗口后证明之前支付。

This is a positive change in aligning providers' cash flow with value provided to the network.
The rewards for a verified deal are earned during its lifetime.
After a deal expires, the sector continues to earn storage power rewards only, like a committed-capacity sector.
This is a necessary step towards enabling more flexible assignment of deals to sectors,
permitting deal extension and the transfer of deal data to new sectors, including those that previously hosted other deals.

这是一个积极的变化，使供应商的现金流与提供给网络的价值保持一致。
已验证交易的奖励将在其有效期内获得。
交易到期后，该行业将继续只获得存储功率奖励，就像承诺容量行业一样。
这是实现更灵活地将交易分配给扇区、允许交易扩展和将交易数据转移到新扇区（包括以前托管其他交易的扇区）的必要步骤。

## Backwards Compatibility 向后兼容性

This proposal introduces changes to actor state and method parameters, 
and also directly changes consensus election rules to use raw byte power instead of QA power.
It thus requires a network-wide consensus upgrade.

本方案引入了对参与者状态和方法参数的更改，并直接更改共识选择规则，以使用原始字节存力而不是QA存力。
因此，它需要网络范围的一致升级。

Tools which inspect on-chain state directly may need to introduce a new schema in order to operate smoothly across the upgrade epoch.

直接检查链上状态的工具可能需要引入新的模式，以便在整个升级周期中顺利运行。

### Migration 迁移
This proposal requires a point-in-time migration of chain state involving the miner, storage market, power, and reward actors.

本提案要求在时间点上迁移涉及采矿者、存储市场、存力和奖励参与者的链状态。

The primary challenge is moving the pledge requirements and rewards associated with quality-adjusted power from 
miner per-sector information into the market per-deal accounting records.

主要挑战是将与质量调整后的电力相关的质押要求和奖励从采矿者的每个扇区信息转移到市场的每个交易帐号记录中。

#### Market actor 市场参与者
Initialise total verified space and per-miner verified space.
For each migrated deal, multiply space by the fraction (<=1) of sector lifetime the deal occupied 
in order to replicate exactly the same reward schedule as QA-power did.
Old deals pay out over sector lifetime, new ones over deal lifetime.

初始化总验证空间和每个矿工验证空间。
对于每个迁移的交易，将空间乘以交易占用的扇区寿命的分数（<=1），以便复制与QA存力完全相同的奖励计划。
旧交易在整个扇区周期内支付，新交易在整个交易周期内支付。

Initialise the MigratedVerifiedDeals metadata with these values.

使用这些值初始化MigratedVerifiedDeals元数据。

Initialise the total and per-provider verified space delta queues with the expirations of the migrated deals at the expiration of their sectors.

使用迁移交易在其扇区到期时的到期时间，初始化总的和每个提供商验证的空间增量队列。

TODO during draft phase: Spec out the migration more fully

草稿阶段的TODO：更全面地规范迁移

#### Miner actor 矿工参与者
Migrate the sectors!

迁移扇区！

#### Reward actor 奖励参与者
Changes to baseline and realised cumsum if necessary. 
Initialise verified deal space for the reward calculation immediately subsequent to migration.

如有必要，对基线和已实现累积和进行更改。
在迁移后立即为奖励计算初始化验证的交易空间。

## Test Cases 测试用例
TODO during draft phase:  
起草阶段的TODO：
- Equivalence of total reward for CC sectors, sectors with deals, varying deal/sector expirations etc
- CC扇区、有交易的扇区、不同的交易/扇区到期日等的总回报等值
- Equivalence of total reward through migration
- 通过迁移获得的总回报的等价性
- Equivalence of pledge functions and penalties
- 质押功能和处罚的等价性

## Security Considerations 安全考虑
TODO during draft phase. Mostly about incentives (below).

草拟阶段的待办事项。主要是关于激励（见下文）。

## Incentive Considerations 激励因素

### Incentive to block producers 阻止生产者的激励
Earning the verified deal premium doesn't depend on winning blocks.
This is a deviation from the current protocol that requires a miner to win a block 
(with a Winning PoSt) in order to claim any rewards. 
This may reduce the incentive for providers with a very high proportion of verified deals to mine blocks.

获得验证交易溢价并不取决于赢得区块。
这是对当前协议的一种偏离，该协议要求矿工赢得一个区块（有一个获胜的帖子）才能获得任何奖励。
这可能会降低具有非常高比例的已验证交易的供应商开采区块的动机。

When network-wide storage utilisation is low, such providers will account for a very small proportion of consensus power.
Most network reward will be earned by block producers.

当网络范围内的存储利用率较低时，此类提供商将占共识存力的很小比例。
大部分网络奖励将由区块产生者获得。

When network-wide storage utilisation is high (>10%), the network may pay more reward to deal providers than block producers.
One mitigation for this could be to cap the fraction of network total reward that the deal premium may draw.
An asymptotic upper bound may be implemented in the reward split function by changing a constant. 
For example, to impose an upper bound of 30% of reward for deal subsidies:  
当网络范围的存储利用率较高（>10%）时，网络可能会向交易提供商支付比区块生产商更多的报酬。
一种缓解措施是限制交易溢价可能获得的网络总回报的一部分。
通过改变常数，可以在奖励分割函数中实现渐近上界。
例如，对交易补贴征收30%的奖励上限：
```
VerifiedDealReward = 9 * NetworkRawPower * TotalVerifiedSpace / (NetworkRawPower + 29 * TotalVerifiedSpace)
ConsensusReward = EpochReward - VerifiedDealReward
```

### Incentive to maintain deals 维持交易的激励
No change: the same total collateral is at risk for reneging on a deal.

没有变化：相同的总抵押品有违约的风险。

### Incentive to maintain healthy storage 保持健康存储的激励措施
No change: the same total penalties are applied for reneging on a deal.

无变化：对违反协议的行为适用相同的总处罚。

Maybe a change after the deal expires, and sector rewards revert to CC.

交易到期后可能会发生变化，行业奖励将恢复为CC。

### Incentive to select sector lifetimes and maintain CC sectors 选择扇区寿命和维护CC扇区的激励
Reward timings are changed, deal reward earned during deal instead of spread out over sector.

奖励时间发生变化，在交易期间获得的交易奖励，而不是分散在整个行业。

Current incentive to select minimal sector life, since longer delays rewards.
Longer lifespan no longer delays rewards, so may see marginal incentive to longer commitments.

当前选择最小扇区寿命的激励，因为延迟时间越长。
更长的寿命不再延迟奖励，因此可能会看到延长承诺的边际激励。

## Product Considerations 产品考虑
TODO during draft phase: better reward profile for small, deal-focussed SPs

起草阶段的待办事项：为小型、以交易为重点的SP提供更好的奖励

## Implementation 实现
A draft implementation is in progress at https://github.com/filecoin-project/specs-actors/pull/1560. 
As a technical FIP, the proposal is expected to be accepted before implementation is completed, 
in order to motivate and direct that implementation.

实施草案正在以下位置进行：https://github.com/filecoin-project/specs-actors/pull/1560.
作为一项技术FIP，该建议预计将在实施完成前被接受，以激励和指导实施。

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).







