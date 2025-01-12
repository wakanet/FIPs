---
fip: "0004"
title: Liquidity Improvement for Storage Miners
author: davidad (@davidad), jbenet (@jbenet), zenground0 (@zenground0), ZX (@zixuanzh), danilo (@danlessa)
discussions-to: https://github.com/filecoin-project/FIPs/issues/14
status: Final
type: Technical
---

Liquidity Improvement for Storage Miners

提高存储矿工的流动性

## Simple Summary 简述

* Make 25% of storage-mining rewards immediately available with no vesting.
* 立即提供25%的存储挖掘奖励，无需授予。

## Abstract
This proposal allows a substantial fraction of block rewards to be immediately available for withdrawal to allow for greater liquidity and decision freedom for storage miners. The majority of mining rewards continue to vest over 180 days as collateral to reduce initial pledge, to align long-term incentives, and to guarantee storage reliability.

## Problem Motivation
All rewards on Filecoin go through a linear vesting schedule of 180 days to encourage miners to consider utility creation besides speculation, to provide space and time for onboarding real use cases onto the network, and to be incentive-aligned with the network long-term. However, a 6-month vesting schedule could be a heavy cashflow burden. 

## Specification
In ApplyRewards of MinerActor,
* Currently: all rewards go through RewardVestingSpec and all rewards get added to pledgeDeltaTotal.
* Proposed: 75% of rewards go through RewardVestingSpec and only 75% rewards get added to pledgeDeltaTotal.

## Design Rationale
The rationale for the specification is to achieve the goals and desired behavior with minimal code changes (conditional checks and parameter updates). Note that 25% of reward will become immediately withdrawable unless the miner is in FeeDebt. In that case, a portion of the 25% of unlocked reward will automatically be applied to recovering the FeeDebt - so the miner’s balance will increase by the amount unclaimed by FeeDebt.

## Backwards Compatibility
This proposal does not require changes to the state schema of any actor, nor method parameter or return values . Thus it can take effect as a behaviour change triggered after a specific upgrade epoch.

## Test Cases
N/A

## Implementation
- Dependency PR: https://github.com/filecoin-project/go-state-types/pull/16
- Actors PR (WIP): https://github.com/filecoin-project/specs-actors/pull/1249 

## Security Considerations
There were security concerns that block reward should only start vesting after one day to mitigate certain malicious strategies. However, this cost would be shouldered by all storage miners in the network. Given the security concern level of the attack and the significant UX improvement of making rewards immediately available, 25% block rewards will still be immediately available and the remaining will vest over 6 months. It is possible that this will be addressed by another FIP in the future.

## Incentive Considerations
Allowing some portion of the block reward to be immediately available increases the risk that miners break away from the lifetime promise of their sectors due to volatile price trajectory or other opportunities. However, this is a tradeoff that the protocol can make to improve cash flow and profitability for miners. It also provides more degree of freedom for storage miners to reinvest into the network. The majority of earned block rewards are still vesting over 6 months to ensure that incentives are aligned.

## Product Considerations
Despite an increased risk of miners not fulfilling their sector lifetime promise and dropping sectors early, penalties for dropping sectors have not decreased. Overall, improved profitability and liquidity should lead to better outcomes for the network and allow for greater storage miners diversity.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
