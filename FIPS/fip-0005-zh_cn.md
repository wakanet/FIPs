---
fip: "0005"
title: Remove ineffective reward vesting 
author: Alex North (@anorth), @Zenground
discussions-to: https://github.com/filecoin-project/FIPs/issues/18
status: Final
type: Technical
category: Core
created: 2020-10-22
---

## Simple Summary 简述
Remove the processing of miner reward vesting from `PreCommitSector` and `ConfirmSectorProofsValid`, because it's expensive and doesn't achieve much.

从`PreCommitSector`和`ConfirmSectorProofsValid`中删除矿工奖励授予的处理，因为这很昂贵，而且效果不大。

## Abstract
Remove expensive calculation of miner reward vesting from the `PreCommitSector` and `ConfirmSectorProofsValid` methods, leaving it to the deadline cron. 
This will reduce the gas consumption of these methods substantially, freeing up chain bandwidth and reducing costs.

## Change Motivation
Chain bandwidth is a valuable resource, necessary for both critical network operations and product use.
Avoiding frequent and high-gas-cost operations reduces contention, costs, and chain validation latency.

Processing reward vesting is quite expensive as it loads and stores a sizeable array for the vesting table. 
For high-scale miners, committing many sectors per deadline (or even per-epoch), this represents a sizeable portion of total gas cost and chain bandwidth consumption.

## Specification
Remove the invocations of `State.UnlockVestedFunds` from `PreCommitSector` and `ConfirmSectorProofsValid`.

## Design Rationale
Reward vesting is computed via `State.UnlockVestedFunds`, which is called from `PreCommitSector`, `ConfirmSectorProofsValid`, `WithdrawBalance` and the deadline cron handler.
The original motivation for computing vesting on sector commitment is so that any vested funds could be used for the pre-commit deposit or pledge.

The vesting table is quantized to 12-hr increments. 
Since vesting is processed in the deadline cron every 30 minutes regardless, a call during sector commitment almost always achieves nothing. 
The most it could achieve is to accelerate a release by 30 minutes.

A miner operator wishing to process vesting manually, ahead of the per-deadline cron call, could do so by calling `WithdrawFunds` with an amount of zero.
Such a call would require use of the miner's Owner address.

## Backwards Compatibility
This change must be implemented along with an increment to the runtime's network version number. It will only take effect after a designated upgrade epoch.

This proposal does not require changes to the state schema of any actor, nor method parameter or return values. 

## Test Cases
N/A

## Security Considerations
None.

## Incentive Considerations
This proposal changes incentives only marginally. 

It may delay the use of recently-vested rewards for new pre-commit deposits or pledges, leading to incentive to hold slightly more working balance in a miner actor.
Similarly, it may incentivize use of the `WithdrawFunds` method in order to accelerate the release of vested funds.

## Product Considerations
This proposal will decrease contention for chain transaction throughput, generally making use of the Filecoin blockchain cheaper.

## Implementation

Implemented in specs-actors in https://github.com/filecoin-project/specs-actors/pull/1276, introduced to the network in [Lotus v1.2.0](https://github.com/filecoin-project/lotus/releases/tag/v1.2.0) at epoch 265200.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
