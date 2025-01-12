---
fip: "0002"
title: Free Faults on Newly Faulted Sectors of a Missed WindowPoSt
author: anorth (@anorth), davidad (@davidad), Evan (@miyazono), Irene (@irenegia), Luca (@lucaniz), Nicola (@nicola), ZX (@zixuanzh)
discussions-to: https://github.com/filecoin-project/FIPs/issues/6
status: Final
type: Technical
created: 2020-09-24
---

## Simple Summary 简述

* Remove fee incurred by a successful recovery.
* 删除成功恢复所产生的费用。
* Remove fee incurred by a missed WindowPoSt.
* 删除丢失的WindowPoSt所产生的费用。
* Update Sector Fault Fee (FF) to 3.51 days of expected block reward.
* 将扇区故障费（FF）更新为预期区块奖励的3.51天。

## Abstract
Given the state of the network, honest miners are sometimes disproportionately penalized for operational failures by the SectorFaultDetectionFee (SP), especially when missing a WindowPoSt. This proposal reduces fees on detected faults, without sacrificing much of the security or the incentive to provide reliable storage. 
Specifically:
1. Detected faults (missing a WindowPost deadline) on 100% healthy partitions incur no fee; 
2. In partitions where some sectors are either already faulty or have declared recovery but a WindowPoSt is missed, a SectorFaultFee (FF) is incurred on those sectors;
3. Faulty sectors detected by missing a WindowPoSt incur an FF per proving period starting on the first proving deadline after they are detected faulty. Skipped Faults thus still require an FF starting on the first proving deadline.

## Problem Motivation
Operational failures are common and sometimes honest miners may miss a WindowPoSt deadline for reasons beyond their control. In the early days of the network, disruption and bugs are expected to be more frequent. The protocol should recognize operational realities and allow for some degree of flexibility, while maintaining the security and incentive to provide reliable storage. Currently, these events will lead to disproportionate faults and penalties incurred by honest miners. Specifically, missing a WindowPoSt within the relevant time window, or failing it because of a single sector, incurs penalties for the full partition (2349 sectors). This FIP intends to decrease the acuteness and magnitude of the penalties for honest intermittent downtime while still maintaining the penalties for very low storage reliability to encourage healthy storage.

## Specification
On `SubmitWindowedPoSt`for a partition, which is a superset of active sectors (skipped faults may be declared on some active sectors), recovering sectors, and faulty sectors:
* For every recovering sector whose proof passes,
  * Currently: Restore power, deduct an FF penalty.
  * Proposed: Restore power, incur no penalty.

* For every active sector declared as skipped,
  * Currently: Remove power, mark the sector as faulty, deduct a penalty of SP-FF now, and an FF is later incurred at `handleProvingDeadline`. 
  * Proposed: Remove power, mark the sector as faulty, incur no penalty now, and an FF is later incurred at `handleProvingDeadline`.

On `handleProvingDeadline` as a Cron event at the deadline:

* For each partition for which the proof is not submitted (detected faults):
  * Currently: Penalize every faulty sector FF; Penalize every sector in the partition that was either recovering or non-faulty SP, and remove power for all non-faulty sectors in the partition. 
  * Proposed:
    * Penalize every faulty and recovering sector FF;
    * Remove power for all non-faulty sectors in the partition, mark them as faulty, but incur no penalty. 

In miner actor constants:
* DeclaredFaultProjectionPeriod
  * Currently: 2.14 days.
  * Proposed: 3.51 days.

## Design Rationale
The rationale for the specification is to achieve the goals and desired behavior with minimal code changes (conditional checks and parameter updates). Many other proposals were considered but reducing penalties for newly faulted sectors and increasing penalty for continued faults effectively reduces the penalty risk with minimal changes to the implementation and incentive structure.  

## Backwards Compatibility
This proposal does not require changes to the state schema of any actor, nor method parameter or return values . Thus it can take effect as a behaviour change triggered after a specific upgrade epoch.

## Test Cases
N/A

## Implementation
https://github.com/filecoin-project/specs-actors/pull/1181

## Security Considerations
The rational strategy is still to store everything and submit WindowPoSt at every deadline; the fact that the protocol allows a miner to not prove an entire partition without paying fees opens to the possibility that miners with many faults in a single partition may prefer to miss the deadline instead of declaring the faults. Nevertheless, for example, it is still rational for a miner who has a partition with < 30% sectors faulty to declare all the faults. Moreover, even with a few faulty sectors partially missing, honest fault declaration could incur a higher penalty than waiting and declaring the fault only when the WindowPoSt challenges detect the missing parts. 

As such, one could make the argument that DeclareFault should now be removed. However, in the spirit of managing implementation complexity, this is left out of scope for this FIP. DeclareFault is now “economically” disabled since it can be an irrational decision compared to missing a WindowPoSt or declaring skipped faults depending on the number of faulty sectors a miner has. There might be other use cases of DeclareFault such as data migration, server maintenance, etc but that can be considered and evaluated in a separate FIP. Client node implementation could disable advance fault declaration to avoid paying higher fees.

## Incentive Considerations
In the past, miners who missed the deadline to submit their WindowPoSt due to normal operational realities were disproportionately penalized by the network because of how the Sector Fault Detection Fee is administered. Operational downtime is common and the protocol should be more tolerant of these effects, while still maintaining a strong incentive to provide reliable storage throughout the lifetime of a sector. Having a more gradual penalty structure allows the protocol to forgive intermittent failures, while imposing higher fees for continued faults, eventually incurring a heavy penalty at termination to incentivize healthy storage. A Markov Decision Process corresponding to a pairing of a partition’s actual availability (up or down, as a daily Bernoulli event) with the chain state (active or faulty) was evaluated. Rewards are emitted by this process corresponding to expected block rewards, fees, and the costs of storage and recovery. In this model, a break-even uptime of 60% was targeted; 3.51 is the lowest fault-fee multiple that will result in positive expected values only for uptimes greater than 60%.

This proposal removes the penalty for the first proving period when a healthy sector is first faulted, however the Sector Fault Fee will be paid in the next proving period if the sector still remains faulty. Note that this proposal also removes penalties for the proving period during which a sector is recovered. This incentive structure accommodates honest operational failures while maintaining a strong incentive to recover quickly and provide reliable storage. A speculative miner may opt for a dishonest strategy of alternating between missing a WindowPoSt and declaring recovery every day, but that will not be a profitable storage strategy compared to storing honestly (due to loss of storage power throughout the faulted period), and will expose the miner to a greater risk of heavy penalties if they miss two consecutive WindowPoSt deadlines due to actual downtime.

## Product Considerations
By making the penalty more tolerant of moderate operational downtime, storage mining will be a more pleasant experience, making Filecoin as a market more attractive for storage providers. At the same time, in view of product concerns from the client side about storage reliability, the fees for continued faults have been increased to maintain a disincentive to supply very unreliable storage.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
