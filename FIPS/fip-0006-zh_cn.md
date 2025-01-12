---
fip: "0006"
title: No repay debt requirement for DeclareFaultsRecovered
author:  Nicola (@nicola), Irene (@irenegia)
discussions-to: https://github.com/filecoin-project/FIPs/issues/21
status: Deferred
type: Technical
created: 2020-10-23
---


## Simple Summary 简述
Remove the requirement to repay debt from DeclareFaultsRecovered.

取消定义错误恢复范围内的债务偿还要求。

## Abstract
Filecoin miners are subject to pay storage fault and consensus fault fees.  These are paid using locked funds (ie, unvested block rewards) and then the miner’s available balance. If these two are not enough to cover the total fee amount, the remaining part to pay is recorded in a variable (FeeDebt) in the miner’s state and the miner is declared “in fee debt”.
For a miner in debt, the call to the methods PreCommit, DeclareFaultsRecovered and WithdrawBalance fails. Moreover this miner can not mine blocks.
The fact that a miner in debt can not recover faults creates an undesired loop situation, in which a miner with faulted partitions can not recover, keep increasing its debt (note that continued faults cause more fees), eventually leading to sector termination if the miner does not repay their debt after 14 days.  This is especially bad for small miners with little balance. To avoid this we propose to allow recovery of faults for miners in debt.

## Problem Motivation
In the current protocol, recovering faults is not allowed if a miner is in debt (ie, the call to the method “DeclareFaultsRecovered” fails when feeDebts > 0). This can create the following problem:
A miner misses the windowPoSt deadline for some partition for 2 days;
This causes paying storage fault fee and the miner could go in debt;
If it does, the miner can not recover the faulted partitions without obtaining the funds to pay back the debt first and the debt will continue to increase.


## Specification
Remove the call to RepayDebtOrAbort in the method “DeclareFaultsRecovered”;
In other words, no check about the FeeDebt variable is required in the method “DeclareFaultsRecovered”.
Diff from current version: remove the following line of code in the miner actor
https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go#L1322.


## Design Rationale
The main motivation for this change is introducing the minimal change that allows miners in debt to recover storage faults in order to stop paying for penalties.

## Backwards Compatibility
This change requires a network version increment to distinguish the old behaviour from the new. It does not change any state schemas or exported method signatures.

## Test Cases
N/A

## Implementation
Remove the following line of code in the miner actor
https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go#L1322.


## Security Considerations
After this change, a miner in fee debt continues to not be able to produce new blocks or withdraw tokens from its available balance, so the possible actions and strategies for malicious miners are the same as in the current version of the protocol. In particular, a miner that does not recover from debt (ie, a miner that does not pay its fees) cannot make profit from the power registered in the power table because it is not eligible to mine blocks. 
Miners that recover their faults are still not eligible until they recover from fee debt, this means that their power will still count towards the total power, and the average WinCount is reduced. Although this is not a new problem, this change could worsen the potential power inflation. We do not believe this is a risk for the protocol and this problem should be addressed separately with a new FIP (e.g., possible options for this are (1) allowing miners in debt to mine blocks or (2) remove the power of miners in debt from the power table).

## Incentive Considerations
After this change, a miner in fee debt continues to be incentivized to repay its debt asap since otherwise it can not win block reward. It allows miners to stop paying for fault penalties if they can recover their sectors early without having the urgent need to acquire tokens or incur more penalties.

## Product Considerations
In the event of a miner going into debt but being able to recover quickly, debt does not increase based on the miner's ability to repay their debt quickly but only depends on storage maintenance. This FIP maintains a strong incentive to recover from faults (no storage growth or block rewards) but also avoid putting miners in a situation where their debt keeps increasing due to their inability to access liquidity in a timely fashion.


## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
