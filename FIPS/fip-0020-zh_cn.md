---
fip: "0020"
title: Add Return Value to WithdrawBalance
author: Steven Li (@steven004), Zenground0 (@Zenground0)
discussions-to: https://github.com/filecoin-project/FIPs/issues/26
status: Final
type: Technical Core
category: Core
created: 2021-06-06
review-period-end: 2021-10-11
spec-sections: 
  - specs-actors
---

<!--You can leave these HTML comments in your merged FIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new FIPs. Note that a FIP number will be assigned by an editor. When opening a pull request to submit your FIP, please use an abbreviated title in the filename, `fip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
Add a return value to market and miner withdraw methods to indicate the actual withdrawn amount

将返回值添加到市场和矿工提取方法，以指示实际提取金额

## Abstract 摘要
<!--A short (~200 word) description of the technical issue being addressed.-->
WithdrawBalance methods can succeed even when withdrawing less than the specified amount in message arguments. To resolve this ambiguity the return value should specify the amount withdrawn.

即使提取的金额小于消息参数中的指定金额，也可以成功提取余额方法。为了解决这种模糊性，返回值应指出提取了的金额。

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

Both miner and market actors have WithdrawBalance methods.  Both methods are an attempt to withdraw a specified amount, but return success even when the available balance is less than the withdrawal amount specified.  So the actual amount withdrawn is equal to or less than the amount specified in the method parameters, and the method always return nil. Therefore there is no way to know how much FIL is actually withdrawn by only checking the chain status and message info e.g. from node CLI or an explorer.

Adding return values specifying the withdrawn amount will improve the visibility and traceability of FIL flow. This is important especially for miners who need to have a very clear balance sheet and financial report.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->
Add `abi.TokenAmount` return values to market and miner `WithdrawBalance` methods and return the amount withdrawn. 

## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
The design is trivial.

## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This proposal requires a breaking change in actor method signatures and therefore a new actors version.

## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->
For both miner and market actors
* WithdrawBalance with an amount that can be withdrawn => the specified amount is returned
* WithdrawBalance with an amount greater than that which can be withdrawn => the actual amount withdrawn is returned

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->
This only changes return value behavior so has no security implications.

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->
This only changes return value behavior so has no incentive implications.

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->
This has a positive product consideration for miners and storage clients.  Most existing tools should still work since it is unlikely tools depend on the empty return value of withdraw balance.  Tools can be improved and adapted to make use of the return value.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
specs-actors PR: https://github.com/filecoin-project/specs-actors/pull/1476

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
