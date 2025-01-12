---
fip: "0026"
title: Extend sector fault cutoff period from 2 weeks to 6 weeks
author: IPFSUnion(@IPFSUnion)
discussions-to: https://github.com/filecoin-project/FIPs/issues/189
status: Final
type: Technical (Core)
created: 2021-10-01
---

<!--You can leave these HTML comments in your merged FIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new FIPs. Note that a FIP number will be assigned by an editor. When opening a pull request to submit your FIP, please use an abbreviated title in the filename, `fip-draft_title_abbrev.md`. The title should be 44 characters or less.-->


## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->

Due to force majeure factors such as major natural disasters, storage providers may not be able to restore services in a short period. According to the current implementation of the protocal, the sector will be forcibly terminated after two consecutive weeks of faults. Two weeks is not enough to complete the EiB-level data migration and recovery, so we hereby propose this FIP.

由于重大自然灾害等不可抗力因素，存储供应商可能无法在短期内恢复服务。根据目前协议的执行情况，该扇区将在连续两周出现故障后被强制终止。两周时间不足以完成EiB级数据迁移和恢复，因此我们在此提出此FIP。

## Abstract 摘要
<!--A short (~200 word) description of the technical issue being addressed.-->

Filecoin needs to extend the fault period so that large storage providers have enough time to complete the data migration.  Six weeks is generally enough for overseas migration, including overall planning, customs application, sea or air transport, etc.  

Filecoin需要延长故障期，以便大型存储提供商有足够的时间完成数据迁移。海外迁移一般需要六周时间，包括总体规划、海关申请、海运或空运等。

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

Any country in the world is likely to face force majeure factors such as major natural disasters or social abnormal events, causing storage providers to be unable to provide services normally for a long period of time. To this end, we must plan ahead.  
Up to the current V13 network, the sector will be forcibly terminated if there are two consecutive weeks of faults. However, two weeks is not enough for a large storage provider to complete the data migration and restart the service. If appropriate measures are not taken, it will not only cause huge economic losses to the storage provider, but also cause large fluctuations in the storage power of the entire Filecoin network.  
Therefore, it is necessary to make some adjustments to the sector fault period.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

Extend fault period of the sector before sector termination from 2 consecutive weeks to 6 consecutive weeks.  

## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

Extend the sector fault period to buy time for storage providers to migrate data

## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

The proposal extends the fault period of the sector, so such changes must be completed through version upgrades.


## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

Test PR: https://github.com/filecoin-project/specs-actors/pull/1506

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

Strong incentives remain for Storage Providers to keep proving all their sectors reliably, which should prevent any increased variability / unreliability in network storage power.

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->

The maintained FaultFee structure provides strong incentives for storage providers to maintain great quality of service and keep any downtime to a bare minimum.  
So this proposal seeks to extend the fault window without recalculating the fault fees schedule. it is thus fundamentally irrational for storage providers to choose to fault on sectors for an extended period of time. This reinforces the idea that this extended fault period ought to be viewed as a worst-case-scenario provision, rather than a benchmark for how long someone could reasonably take data offline. The current punishment is quite exspensive so that it is unlikely to incentive unreliable storage behavior.


## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->

Increasing the sector forced termination window increases the potential time between when a storage provider could stop storing/proving data to the network, and when storage clients would have their payment refunded. This could be annoying/frustrating from a sector repair perspective, since there is a longer window before it is clear whether a storage provider is coming back online, which the client isn't compensated for.



## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

Specs-actors PR: https://github.com/filecoin-project/specs-actors/pull/1504  

Note:  
This proposal will not go into effect until v14 Chocolate network upgrade.  
If a storage provider were to fault on their Window PoSt submission before the upgrade (scheduled for the end of October), they would still be bound to the 14 day fault sector period. On day 15, for example, they could experience sector termination even if this FIP had been approved.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
