---
fip: "0036"
title: Introducing a Sector Duration Multiple for Longer Term Sector Commitment 
status: Draft
type: Technical
author: Axel C (@AxCortesCubero), jbenet (@jbenet), Maria S (@misilva73), Molly M. (@momack2), Tom M. (@tmellan), Vik K. (@vkalghatgi), ZX @zixuanzh)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/421 (prev https://github.com/filecoin-project/FIPs/discussions/386)
created: 2022-07-29
---

## Simple Summary 简述
- A Sector Duration Multiplier is introduced for all sectors, including Committed Capacity sectors and sectors containing storage deals.
- 一个扇区持续时间乘数被引入到了所有扇区，包括承诺存力的扇区和包含存储交易的扇区。
- A longer sector will have a higher Quality Adjusted Power than a shorter sector, all things equal.
- 在所有条件相同的情况下，更长的扇区将比更短的扇区具有更高的质量校正存力。
- The Duration Multiplier is multiplicative on the existing Quality Multiplier incentive (Filecoin Plus incentive).
- 持续时间乘数与现有质量乘数激励（Filecoin额外激励）相乘。
- Sectors with higher Quality Adjusted Power as a result of Duration Multiplier and Quality Multiplier will require higher initial pledge collateral.
- 由于持续时间乘数和质量乘数，具有更高质量校正存力的扇区将需要更高的初始质押抵押。
- The minimum sector duration time will increase to 1 year and the maximum sector duration to 5 years.
- 最小扇区持续时间将增加到1年，最大扇区持续时间增加到5年。
- The SectorInitialConsensusPledge multiplier will increase from 30% to 40%.
- 扇区初始共识抵押乘数将从30%增加到40%。
- This policy will apply at Sector Upgrade and Extension.
- 该策略将适用于扇区升级和扩展。

## Problem Motivation 问题动机
Currently, storage providers do not receive any additional compensation or incentive for committing longer term sectors (whether that be CC or storage deals) to the network. The protocol places equal value on 180 to 540 day sectors in terms of storage mining rewards. However, in making an upfront commitment to longer term sectors, storage providers take on additional operational risks (more can go wrong in a longer time period), and lock rewards for longer. Furthermore, in committing longer term sectors/deals, storage providers demonstrate their long-term commitment to the mission and growth of the Filecoin Network, and are more aligned with client preference for persistent storage. Therefore, the added value of longer-term sector commitments, coupled with the compounded operational/liquidity risks storage providers incur for longer term sectors should be compensated for in the form of increased rewards. 

目前，存储提供商不会因向网络承诺长期扇区（无论是CC还是存储交易）而获得任何额外补偿或奖励。在存储挖掘奖励方面，该协议对180至540天的扇区给予同等价值。然而，在对长期扇区做出前期承诺时，存储提供商会承担额外的运营风险（在较长的时间内可能会出现更多错误），并将奖励锁定更长时间。此外，在承诺较长期的扇区/交易时，存储提供商展示了他们对Filecoin网络的使命和发展的长期承诺，并且更符合客户对持久存储的偏好。因此，长期扇区承诺的附加值，加上存储提供商为长期扇区承担的复合运营/流动性风险，应以增加奖励的形式予以补偿。

From a macroeconomic perspective, incentives to seal for longer durations affect the circulating supply dynamics of the network, since collateral is locked for longer. As the network exists in its current state, the percentage of FIL locked on the network is more likely to decline. This FIP looks to introduce more favorable percentage locked value dynamics, while simultaneously ensuring that this increase in locking contributes to network utility, stable circulating supply dynamics, and SP profitability and optionality. 

从宏观经济的角度来看，由于抵押品被锁定的时间更长，密封时间更长的激励会影响网络的循环供应动态。由于网络处于它的当前状态，网络上锁定的FIL的百分比更有可能下降。该FIP旨在引入更有利的百分比锁定价值动态，同时确保锁定的增加有助于网络效用、稳定的循环供应动态以及SP盈利能力和可选性。

Per the economic preference to increase the percentage value locked, we also propose adjusting the [Initial Consensus Pledge Mutliplier](https://spec.filecoin.io/systems/filecoin_mining/miner_collaterals/) to 40%. The intention is to create long-term aligned total value locked (TVL) dynamics to support stable and predictable conditions for storage provider (SP) returns. We further discuss the problem/change motivation, and explore impacts on SP profitability and network macroeconomics in the CEL analysis brief [here](https://pl-strflt.notion.site/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa). 

根据增加锁定百分比价值的经济偏好，我们还建议调整[初始共识质押乘数](https://spec.filecoin.io/systems/filecoin_mining/miner_collaterals/)至40%。其目的是创建长期对齐的总值锁定（TVL）动态，以支持存储提供商（SP）回报的稳定和可预测条件。我们在CEL分析简报中进一步讨论了问题/变化动机，并探讨了对SP盈利能力和网络宏观经济的影响[此处](https://pl-strflt.notion.site/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa).

The motivation to increase the minimum and maximum sector durations is decreased sector turnover and improved network stability. Given that the network expects to grow, having sectors that expire every 6 months means that the network potentially needs to find new sealing throughput to compensate for the loss of power from expiration. This hinders the network as it continues to scale. 

增加最小和最大扇区持续时间的动机是减少扇区更替和提高网络稳定性。考虑到网络预计会增长，每6个月到期一次的扇区意味着网络可能需要找到新的密封吞吐量，以补偿到期造成的电力损失。这阻碍了网络继续扩展。

Another perspective is that block rewards are high now but exponentially decreasing, and these high early rewards should be used to incentivize participation that’s long-term aligned. 6 months is not long-term. In particular it's short over the scale we need stability in the supply dynamics. Increasing the minimum duration from 6 months to 1 year doubles the minimum level of commitment, and smooths out locking dynamics by stretching inflow-outflow over a longer time period, all while impacting relatively few storage providers as sector durations for CC and FIL+ are both substantially above the minimum on average. 

另一个观点是，区块奖励现在很高，但呈指数级下降，这些早期的高奖励应该用于激励长期一致的参与。6个月不是长期的。特别是，它的规模不够，我们需要供应动态的稳定。将最小持续时间从6个月增加到1年，使最低承诺水平翻了一番，并通过在更长的时间段内延长流入和流出来平滑锁定动态，同时影响相对较少的存储提供商，因为CC和FIL+的扇区持续时间均大大高于平均最小值。

Finally, the new maximum limit of 5 years gives SPs the option to express a long-term commitment to the network which was previously not possible with the maximum sector duration length of 540 days, and also receive commensurate rewards. 

最后，新的最长期限为5年，使SP们有权表达对网络的长期承诺，这在以前的最大扇区持续时间为540天的情况下是不可能的，并且还可以获得相应的奖励。

## Specification 详述

### Sector Duration Multiplier 扇区持续时间乘数
The current sector quality multiplier follows from the spec [here](https://github.com/filecoin-project/specs/blob/ad8af4cd3d56890504cbfd23e5766a279cbfa014/content/systems/filecoin_mining/sector/sector-quality/_index.md). The notion of Sector Quality distinguishes between sectors with heuristics indicating the presence of valuable data.

当前扇区质量乘数源自规范[此处](https://github.com/filecoin-project/specs/blob/ad8af4cd3d56890504cbfd23e5766a279cbfa014/content/systems/filecoin_mining/sector/sector-quality/_index.md). 扇区质量的概念区分了具有启发性的扇区，以标示存在有价值的数据。

Sector Quality Adjusted Power is a weighted average of the quality of its space and it is based on the size, duration and quality of its deals.

扇区质量校正存力是其空间质量的加权平均值，基于其交易的规模、持续时间和质量。

 Name                                | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| QualityBaseMultiplier (QBM)         | Multiplier for power for storage without deals.       |
| DealWeightMultiplier (DWM)          | Multiplier for power for storage with deals.          |
| VerifiedDealWeightMultiplier (VDWM) | Multiplier for power for storage with verified deals. |

The formula for calculating Sector Quality Adjusted Power (or QAP, often referred to as power) makes use of the following factors:

计算扇区质量校正存力（或QAP，通常称为存力）的公式使用了以下因素:

- `dealSpaceTime`: sum of the `duration*size` of each deal
- `dealSpaceTime`: 每笔交易的`duration*size`之和
- `verifiedSpaceTime`: sum of the `duration*size` of each verified deal
- `verifiedSpaceTime`: 每个已验证交易的`duration*size`之和
- `baseSpaceTime` (spacetime without deals): `sectorSize*sectorDuration - dealSpaceTime - verifiedSpaceTime`
- `baseSpaceTime` (无交易的时空): `sectorSize*sectorDuration - dealSpaceTime - verifiedSpaceTime`

Based on these the average quality of a sector is:

$$avgQuality = \frac{baseSpaceTime \cdot QBM + dealSpaceTime \cdot DWM + verifiedSpaceTime \cdot VDWM}{sectorSize \cdot sectorDuration \cdot QBM}$$

The _Sector Quality Adjusted Power_ is:

$sectorQuality = avgQuality \cdot sectorSize$

**Proposed Protocol Change**: 

Introduce a multiplier based on sector duration
 Name                                | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| QualityBaseMultiplier (QBM)         | Multiplier for power for storage without deals.       |
| DealWeightMultiplier (DWM)          | Multiplier for power for storage with deals.          |
| VerifiedDealWeightMultiplier (VDWM) | Multiplier for power for storage with verified deals. |
| **SectorDurationMultiplier (SDM)** | **Multiplier for power for storage based on promised sector duration** |

**This SectorDurationMultiplier function proposed is linear with slope 2**. See below for the function proposed. 
![alt text](https://lh3.googleusercontent.com/IB_Xn5JBcFPQvc_eu-bwcnA3pDdY6mcRER68ThWkI2cGxK3K3c1wyjVF6zf7tQbQn-HqGGn8u7Ct2jX_wp1gzv0TxLuvGqN4gMV21-q4wU2cygemqrVEhTAneVwtPoePgZjK9X_dT3L26Ycsabk)

The rationale to select this linear slope 2 function is based on a principle that the selected parameters should maximize the effectiveness of the duration incentive, subject to SP’s collateral availability constraints, while taking into account micro and macroeconomic consequences with minimal added implementation complexity. Further analysis/simulation is shown in the analysis brief CEL prepared linked above. 

选择该线性斜率2函数的基本原理是，根据SP的抵押品可用性约束，所选参数应最大限度地提高持续时间激励的有效性，同时考虑微观和宏观经济后果，并将实施复杂性的增加降至最低。进一步的分析/模拟显示在上面链接的分析概要CEL中。

Therefore, the new suggested *Sector Quality Adjusted Power* is: 

因此，新建议的*扇区QAP*为：

$sectorQuality = (avgQuality \cdot sectorSize) \cdot SDM$

### Change to Minimum Sector Commitment 更改为最低扇区承诺
We propose a minimum sector commitment of 1 year (360 days). This is a 180-day increase from the current minimum of 6 months (180 days). This will not change the mechanics of sector pre-commit and proving, it will just adjust the minimum sector commitment lifetime to 1-year.

我们建议最低扇区承诺为1年（360天）。这比目前的最低6个月（180天）增加了180天。这不会改变扇区pre-commit和证明的机制，只会将扇区承诺的最低期限调整为1年。

### Change to Maximum Sector Commitment 更改为最大扇区承诺
We propose a maximum sector commitment of 5 years. This is an increase from the current maximum sector commitment of 540 days. Note, the protocol currently sets a maximum sector **lifetime** to 5 years (i.e sectors can be extended up to 5 years). This FIP would not adjust that. So, the maximum sector commitment of 5 years would now equal the maximum sector lifetime. Therefore, upon sector-extension, the maximum sector-extension of up to 5-years remains the same. 

我们建议扇区承诺的最大期限为5年。这比目前540天的最大扇区承诺有所增加。注意，该协议目前将最大扇区**寿命**设置为5年（即，扇区可延长至5年）。本FIP不会对此进行调整。因此，5年的最大扇区承诺现在将等于最大扇区寿命。因此，在扇区延期后，最长5年的扇区延期保持不变。

### Change to PreCommitDeposit  更改为预提交保证金
With this FIP, sectors can get higher quality and expect higher expected rewards than currently possible. This has an impact on the value of the PreCommit Deposit (PCD). From the security point of view, PCD has to be large enough in order to consume the expected gain of a provider that is able to pass the PoRep phase with an invalid replica (i.e. gaining block rewards without storing). The recent FIP0034 sets the PCD to 20 days of expected reward for a sector of quality 10 (max quality). We now need to increase this to 20 days of expected reward for a sector of quality 100 (the new max quality) to maintain the status quo about PoRep security.

有了这一FIP，各扇区可以获得比目前更高的质量和预期回报。这会影响预承诺保证金（PCD）的价值。从安全的角度来看，PCD必须足够大，以消耗能够使用无效副本通过PoRep阶段的提供商的预期收益（即，在不存储的情况下获得块奖励）。最近的FIP0034将质量10（最高质量）扇区的PCD设置为20天的预期奖励。我们现在需要将这一预期奖励增加到20天，用于质量100（新的最高质量）的扇区，以维持PoRep安全的现状。

### Initial Pledge Calculation 初始质押计算
The change we propose to status quo Initial Pledge Calculations is the change to the SectorInitialConsensusPledge calculation as detailed below. 

我们提议对现状初始质押计算进行的变更是对扇区初始共识计算的变更，详情如下。

The protocol defines Sector Initial Pledge as:

$SectorInitialPledge = SectorInitialStoragePledge + SectorInitialConsensusPledge$

Currently, 

$SectorInitialConsensusPledge = 0.3 \cdot FILCirculatingSupply \cdot \frac{SectorQAP}{max(BaselineTarget, NetworkQAP)}$

We propose changing the calculation to a multiplier of  40%: 

$SectorInitialConsensusPledge = 0.4 \cdot FILCirculatingSupply \cdot \frac{SectorQAP}{max(BaselineTarget, NetworkQAP)}$

### Impact on Fault and Termination Fees 对故障和终止费用的影响
We currently propose no change to status quo Fault and Termination Fee calculations. Fees continue to be based on expected daily block rewards. In the future it may be valuable to re-examine the 90 day duration for the termination fee. 

我们目前建议不改变当前的故障和终止费计算。费用继续基于预期的每日区块奖励。将来，重新审查90天的终止费期限可能很有价值。

## Design Rationale 设计原理

### Supporting Longer-Term Commitments 支持长期承诺
The current maximum commitment of 1.5 years limits the ability for SPs to make a long-term commitment to the network (or get rewarded for it). We expect that increasing the maximum allowable commitment of 5 years, while also introducing thoughtful incentives to seal sectors for longer, can increase the stability of storage and predictability of rewards for SP’s. This is further discussed in the sections below.

当前1.5年的最大承诺限制了SP对网络做出长期承诺（或获得奖励）的能力。我们预计，增加5年的最大允许承诺，同时引入深思熟虑的激励措施，以更长时间密封扇区，可以提高存储的稳定性和SP奖励的可预测性。这将在以下章节中进一步讨论。

### Incentivizing Longer Term Commitments 激励长期承诺
Longer term commitments are incentivized by a rewards multiplier. The multiplier increases the amount of FIL expected to be won per sector per unit time based on the duration the sector is committed for. 

长期承诺由奖励乘数激励。乘数根据扇区承诺的持续时间，增加每个扇区每单位时间预计赢得的FIL金额。

The proposed rewards multiplier is linear in duration. This means sectors recieve rewards at a rate linearly proportional to duration. 

提出的奖励乘数在持续时间上是线性的。这意味着各扇区的奖励率与持续时间成线性比例。

Example:
- **Storage Provider A** commits sectors for 1 year that generate on aggregate **2 FIL/day** on average. By the end of the commitment, Storage Provider A expects to have received **730 FIL**.
- **存储提供商A**提交了一年的扇区，平均生成**2 FIL/天**。在承诺结束时，存储提供商A预计已收到**730 FIL**。
- **Storage Provider B** agrees to store the same data, but makes a commitment to store it for 3 years. Since their commitment is three times as long, they receive **6 FIL/day** on average. At the end of the three year commitment they expected to have received **2,190 FIL**.
- **存储提供商B**同意存储相同的数据，但承诺存储3年。由于他们的承诺是原来的三倍，他们平均每天收到**6 FIL**。在三年承诺期结束时，他们预计将收到**2190 FIL**。

The rationale is that operational burden and risk to storage providers increases with duration, and it is fair that they’re commensurately rewarded for this.

其基本原理是，存储提供商的运营负担和风险随着时间的推移而增加，因此，他们得到相应的回报是公平的。

To maintain protocol incentives that are robust to consistent storage, the amount of collateral temporarily locked for the duration of the sector must also increase. Sector sealing gas costs do not increase with the multiplier. This means longer durations have higher capital efficiency, which further incentives longer commitments.

为了保持对一致性存储具有鲁棒性的协议激励，在整个扇区期间临时锁定的抵押品数量也必须增加。扇区密封气体成本不会随着乘数的增加而增加。这意味着更长的持续时间具有更高的资本效率，从而进一步激励更长的承诺。

The form of the duration incentive multiplier is linear with slope 2. The factors behind this specific design choice to incentivize longer commitments are:  
持续时间激励乘数的形式与斜率2呈线性关系。激励长期承诺的具体设计选择背后的因素是：
- **Simplicity**. Rewards proportionate to risk, with an understandable incentive mechanism that’s easy to reason about.
- **简单**。回报与风险成比例，具有易于推理的可理解的激励机制。
- **Sufficiency**. Sublinear may be inadequate to incentivize storage providers to accept the burden of risk longer commitments entail. 
- **充分性**。次线性可能不足以激励存储提供商接受长期承诺带来的风险负担。
- **Supply**. In terms of percentage of available supply, a slope of 2 is identified in simulations as the lowest level that sustains circa 50% percentage available supply locked (two-thirds higher than current value). This is important to incentivize long-term commitments as it supports a stable business environment..
- **供应**。就可用供应百分比而言，模拟中确定的斜率为2，是维持约50%可用供应锁定百分比（比当前值高三分之二）的最低水平。这对于激励长期承诺非常重要，因为它支持稳定的商业环境..
- **Capital**. A more aggressive slope was not chosen on the basis that capital may be less available in the future, and long sector durations with high rewards multipliers, which require increased collateral, should be widely accessible.
- **资本**。选择更激进的斜率并不是基于未来资本可能较少，而且需要增加抵押品的具有高回报乘数的长扇区持续时间应该可以广泛获得。

### Refusing Shorter-Term Commitments 拒绝短期承诺

Currently the minimum sector duration is six months. A new minimum duration of one year is proposed. The rationale is based on three factors:   
目前，扇区的最短期限为六个月。提议新的最低期限为一年。基本原理基于三个因素：
- **Stability**. A one year minimum smooths out locking dynamics by stretching inflow-outflow over a longer time period. 
- **稳定性**。一年的最小值通过在更长的时间段内拉伸流入和流出来平滑锁定动态。
- **Efficiency**. Waves of expiration on a six month basis have the potential to waste resources resealing sectors at twice the rate of one year minimum sectors.
- **效率**。六个月到期的浪潮有可能浪费资源，重新密封扇区的速度是一年最低扇区的两倍。
- **Ethos**. Filecoin has an exponential rewards emission schedule, with high rewards that have been useful to bootstrap the network. As the network matures this perspective should be refined to better incentivize storage providers who are aligned with the long-term goals of the network. One year is only six months longer than the current minimum duration, but it shows much stronger commitment to the long-term principles and long-term success of the network. 
- **精神**。Filecoin有一个指数奖励排放计划，高奖励对引导网络非常有用。随着网络的成熟，应完善这一观点，以更好地激励与网络长期目标保持一致的存储提供商。一年只比目前的最短期限长六个月，但它显示出对网络长期原则和长期成功的更坚定承诺。

Furthermore, there is empirical evidence from the duration of sectors sealed that most storage providers support sectors greater than one year. Increasing the minimum from six months to one year will discourage only the most short-term-aligned storage providers.

此外，从行业封闭的持续时间来看，有经验证据表明，大多数存储供应商支持的行业超过一年。将最低期限从六个月增加到一年，只会让最短期的存储提供商望而却步。

### Improving Stability of Rewards 提高奖励的稳定性
A stable investing environment is needed to support long-term storage businesses. A high double-digit percentage return on pledge locked is not sufficient alone. To this end a sustained and substantial amount of locked supply is also needed. 

需要稳定的投资环境来支持长期存储业务。单靠高达两位数的质押锁定回报率是不够的。为此，还需要持续和大量的锁定供应。

In reality the percentage of available supply locked has been decreasing since September 2021. While current token emission rate is exponentially decreasing with time, the percentage of available supply locked is expected to continue to decline, at least until the linear vesting schedule completes, based on current locking inflow-outflows and network transaction fees.

事实上，自2021年9月以来，锁定的可用供应百分比一直在下降。虽然当前代币排放率随着时间呈指数下降，但基于当前锁定流入流出和网络交易费用，锁定的可供供应百分比预计将继续下降，至少直到线性归属时间表完成。

This environment can be improved however. The first way to improve it is by increasing the 30% multiplier in the InitialConsensusPledge to 40%. This is a moderate increase that provides a solid long-term improvement in percentage of available supply locked. The second way is a corollary of the duration multiplier incentive. Longer sectors mean collateral is locked for longer. All else equal, at equilibrium this means the total amount of locked collateral is consistently higher. Simulations confirm both effects together can target a percentage of available supply locked that is sufficiently high and sustained to substantially improve the long-term storage business environment. See [Supplementary Information](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe) for a detailed summary of the supporting simulation analysis

然而，可以改善这种环境。第一种改进方法是将初始共识边缘中的30%乘数增加到40%。这是一个温和的增长，在可用供应锁定的百分比方面提供了一个稳固的长期改善。第二种方式是持续时间乘数激励的必然结果。更长的扇区意味着抵押品锁定的时间更长。在所有其他条件相同的情况下，在平衡状态下，这意味着锁定抵押品的总量始终较高。模拟结果证实，这两种效应一起可以将锁定的可用供应的百分比定为目标，该百分比足够高且持续，以显著改善长期存储业务环境。见[补充信息](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe)详细总结了支持性仿真分析

### Rebalancing SP Profitability 重新平衡SP盈利能力
Return on investment from pledged collateral provided by the storage rewards are currently substantial, with Filecoin-denominated returns in high double digits. Yet Filecoin-denominated returns are only part of what is needed to support successful long-term storage. 

存储奖励提供的质押抵押品的投资回报目前相当可观，以Filecoin计价的回报率高达两位数。然而，以Filecoin计价的回报只是支持成功长期存储所需的一部分。

Simulations indicate a better balance between current and future rewards can be achieved through the proposed changes. The proposals adjust the Filecoin-denominated minting-based returns to a more sustainable level in the immediate term, while the long-term trajectory is unchanged. This enables improving the percentage locked supply to stabilize the business environment for long-term network success. See [Supplementary Information](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe) for further details for percentage return on invested collaterals from mining reward.

仿真表明，通过提出的变化可以实现当前和未来奖励之间的更好平衡。这些建议在短期内将以Filecoin命名的基于铸币的回报调整到更可持续的水平，而长期轨迹不变。这使得能够提高锁定供应的百分比，以稳定业务环境，实现长期网络成功。见[补充信息](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe)有关采矿奖励投资抵押品的百分比回报率的更多详细信息。

### Impact on Initial Pledge 对初始质押的影响
The initial pledge per raw byte power will increase. This is by design. It intends to increase the percentage of available supply locked. 

每个原始字节的初始保证功率将增加。这是设计的。它打算增加可用供应锁定的百分比。

The initial pledge per quality adjusted power, which is the relevant measure for storage provider’s return on pledge invested, may be marginally higher than current to begin with, but will decrease with time. See [Supplementary Information](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe) for plausible trajectories across different new average duration scenarios.

每质量校正存力的初始质押是存储供应商投资质押回报的相关衡量标准，从一开始可能略高于当前水平，但会随着时间的推移而降低。见[补充信息](https://www.notion.so/pl-strflt/Duration-Changes-FIP-discussion-Analysis-Summary-735ce6685b7946f0a03fc13c3fe271fa?d=711694bed594481eb26c1aa46a8e51ec#4c47f8071e7c4d35bfd242580b43efbe)不同新的平均持续时间场景的合理轨迹。

### Impact on Pre-Commit Deposit  对预承诺保证金的影响
FIP-0034 sets the pre-commit deposit to a fixed value regardless of sector content. From the security point of view, PCD has to be large enough in order to cover the expected gain of a provider that is able to pass the PoRep phase with an invalid replica (i.e. gaining block rewards without storing). The recent FIP0034 sets the PCD to 20 days of expected reward for a sector of quality 10 (max quality). We now need to increase this to 20 days of expected reward for a sector of quality 100 (the new max quality) to maintain the status quo about PoRep security.

FIP-0034将pre-commit保证金(译注:PCD)设置为固定值，而与扇区内容无关。从安全角度来看，PCD必须足够大，以覆盖能够使用无效副本通过PoRep阶段的提供商的预期收益（即，在不存储的情况下获得区块奖励）。最近的FIP0034将质量10（最高质量）扇区的PCD设置为20天的预期奖励。我们现在需要将这一预期奖励增加到20天，用于质量100（新的最高质量）的主扇区，以维持PoRep安全的现状。

As of end of July 2022, the calculations are roughly: 

截至2022年7月底，计算结果大致如下：

```
EpochReward := 20.53 * 5 = 102.7 FIL
NetworkPower := 17.98 * 2^60 Bytes
CirculatingSupply := 330.7 * 10^6 FIL

// Sector quality = 2
StoragePledge := 0.0196 FIL
ConsensusPledge := 0.4386 FIL
PreCommitDeposit := 50 * StoragePledge = 0.9802 FIL
InitialPledge := StoragePledge + ConsensusPledge = 0.4582 FIL

// Sector quality = 100 (values are approx 50x greater)
StoragePledge := 0.9802 FIL
ConsensusPledge := 21.93 FIL
PreCommitDeposit := StoragePledge = 0.9802 FIL
InitialPledge := StoragePledge + ConsensusPledge = 22.9102 FIL 
```
Note:
- per the change proposed by the FIP, the ConsensusPledge is calculated as:
- 根据FIP提出的变更，协商一致的边缘计算如下：
- $SectorInitialConsensusPledge = 0.4 \cdot FILCirculatingSupply \cdot \frac{SectorQAP}{max(BaselineTarget, NetworkQAP)}$
- The **minimum** Sector Quality is now 2 per the duration multiplier function outlined in the Design Specification. 
- 根据设计规范中概述的持续时间乘数函数，**最小**扇区质量现在为2。

## Backwards Compatibility 向后兼容性
This policy would apply at Sector Extension and Upgrade for existing sectors. 

该政策将适用于现有行业的行业扩展和升级。

## Test Cases 测试用例
N/A

## Security Considerations 安全考虑

### Risks of Faulty Proof-of-Replication 复制错误证明的风险
The existing 1.5 year sector duration limit in effect provides a built-in rotation mechanism that can be used to turn over power in the event we discover a flaw in PoRep. Increasing the maximum commitment to 5 years weakens this mechanism. [Alternative](https://github.com/filecoin-project/FIPs/discussions/415) policies have been suggested. This is something the community must be aware of and agree on.

现有的1.5年扇区持续时间限制实际上提供了一种内置的旋转机制，在我们发现PoRep中存在缺陷时，可以使用该机制来切换功率。将最高承诺增加到5年会削弱这一机制。[备选案文](https://github.com/filecoin-project/FIPs/discussions/415)已经提出了政策建议。这是社区必须意识到并同意的事情。

### Risks to Consensus 达成共识的风险
The proposed rewards multiplier increases potential risk to consensus. The main consideration is how long it would take for a colluding consortium of storage providers to exceed threshold values of consensus power.

拟议的奖励乘数增加了达成共识的潜在风险。主要考虑的是，串通的存储提供商财团需要多长时间才能超过共识存力的阈值。

Analysis indicates a malicious consortium would need consistent access to high levels of FIL+, and near-exclusive access to the maximum rewards multiplier, for a substantial period of time, for a viable attack. 

分析表明，恶意联盟需要在相当长的一段时间内持续访问高级别的FIL+，并接近独占地访问最大奖励乘数，才能进行可行的攻击。

*Example:*
The network currently has 18 EiB of quality adjusted power.  
Consider the scenario of 50 PiB/day onboarding, with 5% attributed to FIL+, and that this is sustained for several months. 

该网络目前具有18个质量调整功率的EiB。
考虑50 PiB/天的入职情况，其中5%归因于FIL+，并持续数月。

Now if the malicious consortium can acquire 50% of FIL+ deals and commit sectors for 5 years to gain the maximum duration multiplier, and all other storage power maintain the lowest possible duration sectors of 1 year, then in a single day, the adversarial colluding group is expected to gain 0.7% of consensus power. This follows from:

现在，如果恶意财团可以收购50%的FIL+交易，并在5年内承诺扇区，以获得最大持续时间乘数，而所有其他存储能力保持1年的最低持续时间扇区，那么在一天内，对抗性串通集团预计将获得0.7%的共识权力。这是由于：

```
1. advPower = advFILplusPct * FILplusMultiplier * durationMultiplier * powerOnboarding * FILplusPct
2. advPower =  0.5 * 10 * (2 * 5) * 50 * 0.05 = 125
```
where `advFILplusPct` is the fraction of FILplus deals available that are acquired by the adversary, `FILplusMultiplier` is the 10x FIL+ power multiplier, `durationMultiplier` is the maximum 5 year duration multiplier (5 * 2), `powerOnboarding` is the byte power onboarded, and `FILplusPct` is the fraction of the power that is FIL+.

其中，`advFILplusPct`是对手获得的可用FILplus交易的分数，`FILplusMultiplier`是10倍FIL+功率乘数，`duration乘数`是最大5年持续时间乘数（5 * 2），`powerOnboarding`是加载的字节存力，而`FILplusPct`是FIL+的存力分数。

```
3. honestPower = 0.5 * 10 * (2 * 1) * 50 * 0.05 + 1 * (2 * 1) * 50 * 0.95 = 120
4. advPowerDailyPctGain = advPower/(18*1024 + honestPower) 
5. advPowerDailyPctGain = 0.7%
```
If this scenario is maintained, the adversarial group is expected to exceed 33% of consensus power within 140 days. 

如果维持这种情况，预计敌对集团将在140天内超过33%的共识力量。

Factors that mitigate this risk are that it’s unlikely a single group could achieve 50% of FIL+ power consistently, and unlikely that the adversarial group exclusively takes up the longer duration sectors with enhanced power multipliers. 

缓解这一风险的因素是，单个集团不太可能持续获得50%的FIL+功率，敌对集团也不太可能通过增强的存力乘数专门占据更长持续时间的扇区。

A limitation is that the above calculation assumes the malicious party is starting from 0% of consensus power. If they already control 10%, time to 33% is reduced to approximately 100 days.  

一个限制是，上述计算假设恶意方从共识功率的0%开始。如果它们已经控制了10%，则达到33%的时间减少到大约100天。

### Rollout Shock 滚动冲击
Rollout shock could occur if SPs race to extend their commitments and gain a further 10x multiplier. This could be mitigated by gradually increasing the maximum multiplier 1x to 5x during an initial period (e.g. first three months) 

如果SP争先恐后地延长其承诺并进一步获得10倍的乘数，则可能会出现推出冲击。这可以通过在初始阶段（如前三个月）逐渐将最大乘数从1倍增加到5倍来缓解

## Product & Incentive Considerations 产品和激励因素
As discussed in the problem motivation section, this FIP introduces incentives to further align the cryptoeconomic schema of the Filecoin Network with intended goals of the network to provide useful and reliable storage. We introduce the idea that longer term sectors represent a long-term investment and commitment to the Filecoin ecosystem, and therefore should be rewarded proportionally with greater block reward shares.

如问题动机部分所述，本FIP引入了激励措施，以进一步使Filecoin网络的加密经济模式与网络的预期目标保持一致，从而提供有用和可靠的存储。我们提出了这样一种观点，即长期扇区代表着对Filecoin生态系统的长期投资和承诺，因此应按比例获得更大的区块奖励份额。

Note, we also introduce the possibility for storage providers to receive additional multipliers from committing CC for longer. Even this has added value insofar as it represents a commitment to the ecosystem long term that should be rewarded. 

注意，我们还引入了存储提供商在更长时间内从提交CC中接收额外乘数的可能性。即使如此，它也具有附加值，因为它代表了对生态系统的长期承诺，应该得到奖励。

From a product perspective, we see strong support for a network more aligned with longer-term stable storage. From a recent (< 3 week old) snapshot of all LDN applications, the responses fall into the buckets below. Almost half (47%) of all applicants want long-term or "permanent" storage.

从产品的角度来看，我们看到了对与长期稳定存储更加一致的网络的强大支持。根据所有LDN应用程序的最新（<3周）快照，响应分为以下几类。几乎一半（47%）的申请者希望长期或“永久”储存。

| Period | Count | Percentage |
| :---: | :---: | :---: |
| 1 to 1.5+ years | 56 | 22% |
| 2+ years | 22 | 9% |
| 3+ years | 40 | 16% |
| 5+ years | 15 | 6% |
| Long-term/Permanent | 120 | 47% |

We recognize that this proposal may not align with a small fraction of SP’s who exclusively prefer shorter commitments to the network, but contend that from an ecosystem perspective, this policy on aggregate makes most participants better off. Note, regular deals can still be accepted for less than sector duration, so there should be minimal loss to flexibility for onboarding clients. 

我们认识到，这一提议可能与一小部分SP不一致，他们只喜欢对网络做出较短的承诺，但认为从生态系统的角度来看，这一总体政策使大多数参与者受益。注意，常规交易仍可以在少于扇区持续时间的情况下接受，因此，入职客户的灵活性损失应该最小。

For smaller SP’s, introducing this policy could help improve their competitiveness and ability to capture network block rewards, Under this proposal, returns on tokens put up as collateral scale linearly for all SP’s (regardless of size), whereas only larger ones are able to take advantage of economies of scale for hardware. This proposal, if anything, should benefit smaller SP’s because they can still get rewards boost/multipliers without prohibitively expensive hardware costs, and termination risks associated with FIL+ data.

对于较小的SP，引入这一政策有助于提高其竞争力和获取网络块奖励的能力。根据这一提议，作为抵押品的代币的回报对于所有SP（无论大小）都呈线性增长，而只有较大的SP能够利用硬件的规模经济。如果有什么不同的话，这项建议应该会使较小的SP受益，因为他们仍然可以获得奖励提升/乘数，而不会产生昂贵的硬件成本，也不会产生与FIL+数据相关的终止风险。

## Implementation
TBD 
