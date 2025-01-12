---
fip: "0012"
title: DataCap Top up for FIL+ Client Addresses
author: DS (@dshoy), JV (@jnthnvctr), ZX (@zx)
discussions-to: https://github.com/filecoin-project/FIPs/issues/70
status: Final
type: Technical
category: Core
created: 2021-01-29
spec-sections: 
  - https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go
---

<!--You can leave these HTML comments in your merged FIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new FIPs. Note that a FIP number will be assigned by an editor. When opening a pull request to submit your FIP, please use an abbreviated title in the filename, `fip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
This FIP proposes a change to the way DataCap is managed with regards to Client addresses within Filecoin Plus. Currently, Client addresses may receive a one-time allocation of DataCap and any future allocations must be sent to a new address. This FIP proposes enabling subsequent allocations to a Client address that has previously received DataCap

本FIP建议改变Filecoin Plus中有关客户端地址的DataCap管理方式。目前，客户端地址可能会收到一次性的DataCap分配，任何未来的分配都必须发送到新地址。本FIP建议启用对先前已接收DataCap的客户端地址的后续分配

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->


Currently, Clients may only receive a single DataCap allocation to any given address with subsequent allocations requiring a new address if there is DataCap remaining, unless they are able to spend the entirety of their previous allocation. However, given market dynamics it may not always be possible to liquidate small enough amounts of Datacap in order to be removed as a verified Client.

The original motivation was to err on the side of cautiousness as the FIL+ mechanism was developed and ensure this new mechanism was not abused. However, now that this mechanism has been running for a period of time it is clear that the constraint is no longer necessary. Removing this constraint will improve the user experience for developers and end-users while having minimal impact on security. 

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->


The motivation of this change is to reduce a security constraint that introduces an unnecessary amount of friction for users in Filecoin. By specifying each allocation requires a new address, Clients must constantly rotate through the addresses they intend to use for deals - introducing significantly more complexity for developers who are building applications for end-users to manage their own storage. Removing this constraint will simplify the process for developing applications on Filecoin that intend to use DataCap with no meaningful impact on security as dispute resolution and community governance now happen in the notary-governance repository.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

Client addresses should be able to receive additional DataCap allocations to a given address. Remove [checks](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go#L190-L192) on whether `AddVerifiedClientParams.Address` is already a VerifiedClient. 

If the on-chain address has no DataCap: add new `AddVerifiedClientParams.Allowance` to this address.
If the on-chain address is currently a Verified Client: add `AddVerifiedClientParams.Allowance` to its current DataCap balance.

Client addresses should be able to receive additional DataCap allocations to a given address. Cases to consider: 
On-chain address, never received DataCap
No change from current mechanism - Client should be able to request DataCap to this address.
On-chain address, received DataCap previously
Client should be able to request DataCap to this address again
DataCap should be treated as an addition to existing balance
Existing DataCap Balance = Existing DataCap Balance + New Allocation


## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->


## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->


## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->
The major security consideration is whether enabling top ups to an existing address might potentially introduce additional risk into this mechanism. Below are a few scenarios which are worth considering: 

- Malicious actor attempting to acquire large amounts of DataCap
  - In the existing implementation, a malicious actor could generate many addresses and individually apply to multiple notaries to acquire DataCap. A malicious client could selectively disclose existing addresses, to make it seem like they had less DataCap than they actually do. 
  - In the new implementation, a malicious actor might apply to a notary with the same address to multiple Notaries. So long as every Notary is checking the addresses existing allocation before approving a further allocation (and presuming there isn’t a race condition in the approvals), this malicious actor would be discovered. 
- Client applying to multiple Notaries with the same address
  - A legitimate client may apply to multiple Notaries aiming to acquire DataCap. Given different response times, said Client might not actually require all the DataCap requested - instead they may just be looking to receive DataCap as quickly as possible.
  - One fix here is to improve the tooling, for Notaries to be notified of the last allocation an address has received prior to approval. Making it visually obvious that a Client has recently received an allocation can prompt the Notary to confirm whether the prospective allocation is still required. 
  - This leaves the possibility of a race condition still open - where two Notaries independently confirm the same transaction at the same time. However, this can be mitigated in two ways. First, given the geographic distribution and overall limited number of Notaries, the likelihood of two Notaries approving at the same time is quite rare. Second, the tooling for the Notaries can be used to prompt the Notary to confirm with the client that there are no other pending allocations prior to making the allocation.

In the new implementation, there is actually a slight security benefit in that a Notary is able to see a transaction history for a given address (and for honest actors the norm would likely be to use a set of addresses), which might better distinguish legitimate users from malicious ones.

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->


This should have no impact on existing incentives - Clients who wish to acquire DataCap today can spin up multiple addresses to receive an equal amount of DataCap. The primary change here would be to minimize the amount of address rotation that clients and developers will need to engage in during prolonged operation.

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->


From a product perspective implementing this change will greatly enhance the user experience for both developers and for end users. Rather than requiring developers to rotate addresses behind the scenes on behalf of users to make deals, developers can choose whether they’d like to use a static address per user or not. Similarly, Clients who are attempting to do large scale data storage would be able to use a static address for their storage and simply focus on requesting additional top ups as deals are made, rather than being required to also rotate addresses. For Notaries, this change also introduces the added benefit of creating a norm around re-using addresses, which could be used in establishing a Client’s track record of previous allocation decisions and potentially better inform future allocation amounts.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
