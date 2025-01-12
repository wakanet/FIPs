---
fip: "0028"
title: Remove DataCap and verified client status from client address
author: Jiaying Wang (@jennijuju), Deep (@dkkapur), Fil+ Governance Community
discussions-to: https://github.com/filecoin-project/FIPs/issues/204
status: Final
type: Technical
created: 2021-12-13
---

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
In the case of inactivity from a verified client or abuse of the [Filecoin Plus](https://docs.filecoin.io/store/filecoin-plus/) system and incentives, notaries can remove partial or all DataCap from a client address. When clients's datacap is less or equal to zero, it means the verified client status for that address is revoked as well. 

如果验证客户不活动或滥用[Filecoin Plus](https://docs.filecoin.io/store/filecoin-plus/)系统和激励措施，公证人可以从客户地址中删除部分或全部数据帽。当客户端的数据上限小于或等于零时，这意味着该地址的已验证客户端状态也被撤销。

## Abstract 摘要
<!--A short (~200 word) description of the technical issue being addressed.-->
Today, DataCap is granted to notaries (verifiers on chain) which they can then allocate it to client addresses as a one-time-use credit. As clients make verified deals on chain, DataCap is used. Clients will need additional DataCap from notaries if they used up all DataCap that was allocated to them. Notaries will have to re-allocated more DataCap for these clients to re-earn verified client status and make verified deals. This flow is the only way in which DataCap is used and deducted from the network. DataCap is a valuable resource and the Fil+ system is still evolving quite a bit with regards to better identification of trustworthy clients. Recently, the program has been trending towards more rapid iteration with higher risk taking appetite in an effort to improve the client UX and get more data on ways in which DataCap can be used as a useful level. The option to remove DataCap would increase the odds of "success" of the program in making Filecoin as productive as possible and introduce an additional point of leverage to continue growing the Fil+ ecosystem without less risk of the system getting abused.

今天，DataCap被授予公证人（链上的验证者），然后他们可以将其作为一次性信用分配给客户地址。当客户在链上进行验证交易时，使用DataCap。如果客户用完了分配给他们的所有DataCap，他们将需要公证人提供额外的DataCap。公证人将不得不为这些客户重新分配更多的数据上限，以重新获得经验证的客户身份并进行经验证的交易。该流是使用DataCap并从网络中扣除DataCap的唯一方式。DataCap是一种有价值的资源，Fil+系统在更好地识别可信客户方面仍在不断发展。最近，该计划一直趋向于更快速的迭代，具有更高的风险承受能力，以努力改善客户UX，并获得更多数据，说明如何将DataCap用作有用的级别。删除DataCap的选项将增加该计划“成功”的几率，使Filecoin尽可能具有生产力，并引入额外的杠杆点，以继续发展Fil+生态系统，同时降低系统被滥用的风险。

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

_Note: this FIP is created based on the current state (Dec 2021) of Filecoin and the Fil+ program. It is under the assumption of the high level principles and the mechanism of the program will stay unchanged for the next 4-6 months, and does not account for changes that may occur due to the introduction of the FVM._

The program has recently hit the milestone of having 1000+ unique client addresses receive DataCap. DataCap utilization by clients historically has hovered between 30-40%. There are a lot of client addresses with DataCap on the network, many of which are not using the allocation they have recevied. The top level goal of Fil+ is to make Filecoin more productive, and adding an option to remove DataCap is useful in several ways:

- increasing the risk-taking ability for the program to achieve the next order of magnitude of scale
- provides a lever to ensure clients have consequences for violating trust / notaries have a mechanism to enforce dispute and audit results
- removing latent DataCap which could be used for future storage market manipulation or DataCap selling/buying


## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

### Add `RemoveVerifiedClientDatacap` to the verified_registry_actor

<TO BE REVIEWED>

```go

  func (a Actor) RemoveVerifiedClientDatacap(rt runtime.Runtime, params * RemoveVerifiedClientDatacapParams) * RemoveVerifiedClientDatacapReturn {
  ...

  }
  
  type RemoveVerifiedClientDatacapParams struct {
	  Address   addr.Address
	  Allowance DataCap
	  InitiatorSignature crypto.Signature
          ApprovalSignature crypto.Signature
  }
	
```
	
V1 of `RemoveVerifiedClientDatacap` will need 4 total signatures, two from any of the verifiers (notaries) and a root key holder multisig (`f080`) approval (threshold of 2). Every DataCap removal from a client must therefore have:

- 2 notaries approving
- 2 RKH approving

For the V1 design, neither of the notaries needs to have been the original verifier / granter of DataCap to the verified client.

If the requested amount of DataCap for removal is greater or equal than the remaining DataCap the client has, the client address' DataCap balance will be set to 0, and the client will be removed from the `VerifiedClients` map in verified_registry_state.


## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

Having 2 notaries sign ensures notaries looking to remove a client's DataCap are communicating and sharing information with the community and other notaries. 

Introducing the RKH signature in the process creates a need for documentation and public audit trail so that a root key holder can sign this proposal. This also provides a security check in case of malicious notary action. 


## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

Not applicable here - past DataCap allocations are eligible for removal in the future unless the DataCap has already been used in deals. Altering deal state is outside the scope of this proposal.

## Test Cases
<!--Test cases
 for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

Testing is not blocking for this FIP, but could be good to ensure the combination of 2 notary + 1 RKH signer is safe.

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

This FIP introduces a new behavior in the network whereby unused DataCap can be removed from a client address. This increases the overall risk surface area for Fil+, whereby there may be an incentive for notary addresses + RKH addresses have another desirable power in the network. However, this is not really as lucrative or abusable as granting DataCap to a malicious client could be, and has the additional stopgap of requiring 3 signers, so overall, this FIP should not create any significant new threats/risks for the network. 

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->

As mentioned above, DataCap is a lucrative resource in the network, and having the ability to control / influence it is definitely desireable. This FIP does not meaningfully change the incentives for stakeholders in the Fil+ ecosystem since removing DataCap is a reversible action, i.e., if incorrectly removed, DataCap can be granted again to the client entity.

Based on how the Fil+ community supports DataCap removal  for client inactivity, there is an additional incentive to use DataCap when clients receive it. However, this is in line with the programs goals to make the network more productive, and reduces risk of clients stockpiling unused DataCap.

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->

For most trustworthy clients, this FIP does not directly change the product experience of the network. 

However, this gives notaries, and the Fil+ community in general, leverage for further experimentation and risk taking that will have hopefully have a positive impact on future client experience.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
TODO

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
