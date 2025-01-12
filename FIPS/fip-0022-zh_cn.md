---
fip: "0022"
title: Bad deals don't fail PublishStorageDeals
author: ZenGround0 (@ZenGround0)
discussions-to: https://github.com/filecoin-project/FIPs/issues/142
status: Final
type: Core
category (*only required for Standard Track*): Core
created: 2021-09-06
review-period-end: 2021-10-11
spec-sections: 
  - specs-actors
---

<!--You can leave these HTML comments in your merged FIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new FIPs. Note that a FIP number will be assigned by an editor. When opening a pull request to submit your FIP, please use an abbreviated title in the filename, `fip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
Change the PublishStorageDeals implementation so that a single bad deal in the parameters does not fail the message and prevent publishing all the valid deals in the parameters. Instead the message will succeed, only publish only the valid deals and include failure information in the return value.

更改PublishStorageDeals实现，以便参数中的单个错误交易不会使消息失败，并阻止发布参数中的所有有效交易。相反，消息将成功，只发布有效交易，并在返回值中包含失败信息。

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
The PublishStorageDeals message drops deals that error and publishes valid ones. To maintain complete information about published deals in the receipt the return value contains a bitfield of the indexes of all validly deals that were published.

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->
Because to some extent deal errors are unavoidable this issue is frequently brought up by storage providers as a problem. For example filecoin-project/specs-actors#1466. This change will make deal publishing errors less expensive for storage providers and less disruptive for clients.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

PublishStorageDeals deal validation, duplicate deal filtering and balance locking errors do not cause message failure but instead these errors are logged and the deal causing the problem is be dropped from the publish set. VerifiedRegistry actor UseBytes calls are now moved before publishing state changes so that these errors can also be logged and deals dropped instead of triggering message failure.

PublishStorageDeals logic is changed to first iterate over deals and apply checks to filter out invalid deals without mutating the market actor state. Only after filtering invalid deals will the method iterate through valid deals and modify state for all valid deals atomically.

The return value of PublishStorageDeals is changed from

```golang
PublishStorageDealsReturn struct {
    IDs []abi.DealID
}
```
to
```golang
PublishStorageDealsReturn struct {
    IDs []abi.DealID
    Valid bitfield.Bitfield
}
```

We use a bitfield to keep the return bytes compact. The bitfield is over the index of the input array of client deal proposals. For example if three client deal proposals are input and the first two are invalid the `Valid` bitfield will be 001. With this new return format consumers of the PublishStorageDeals receipt can maintain exact information on which proposals are matched to which deal ids. All valid client deal proposals are assigned a deal ID and added in order to the return array.

If all proposals fail validation PublishStorageDeals will return an error. If an ErrIllegalState error or other internal error is encountered PublishStorageDeals will return an error.

## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
The design of the main error semantics change is trivial given the goal to improve user experience.

The `Valid` bitfield in the return value is over input index because deal ids are only assigned after publish storage deals runs so an index of deal ids does not work.  A bitfield is used to enable return information to be compact.  While marking failed deals in the output would also work this proposal chooses valid deals because the valid deal the successes are the most directly relevant to consumers which saves an extra bitfield XOR during processing.

## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This proposal requires a breaking change in an actor method signature and therefore a new actors version.

## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

* No errors and all deals pass
Correct return values and successful publishing of valid deals in the precense of the following errors:
* Deal validation errors
* Insufficient balance errors
* VerifiedRegistry UseBytes errors

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

There are no security implications.

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->

There are no incentive implications.

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->

Miners publishing deals will have an improved user experience.  Clients and others waiting on and parsing PublishStorageDeals return values will need to update software to use the new error handling semantics.  This will require being able to read filecoin RLE+ bitfields which might take some developement work.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

specs-actors PR: https://github.com/filecoin-project/specs-actors/pull/1487
## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
