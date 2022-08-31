---
fip: FIP-0035
title: Support actors as built-in storage market clients
author: Alex North (@anorth)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/271
status: Withdrawn
type: Technical
category: Core
created: 2022-02-17
---

## Simple Summary 简述
Extend the built-in storage market actor to support deal proposal by clients 
without the need to submit a signed message.
This allows non-account actors (which have no signing key) to act as deal clients.

扩展内置存储市场参与者，以支持客户的交易提案，而无需提交签名消息。
这允许非账户参与者（没有签名密钥）充当交易客户。

## Abstract 概要
The built-in storage market actor does not support other actors as deal clients.
When the FVM enables user-programmable actors, this would be a big limitation on their functionality.
This limitation arises from the market actor's requirement for a client signature on a deal proposal,
but only externally-owned accounts can make signatures.

内置存储市场参与者不支持其他参与者作为交易客户。
当FVM启用用户可编程角色时，这将对其功能造成很大限制。
这一限制源于市场参与者要求客户在交易提案上签名，但只有外部拥有的账户才能签名。

This proposal adds new methods to the storage market actor for a client to propose a deal on-chain,
and then a provider to confirm that deal. This complements the existing one-step publishing of storage
deals that is suitable for externally-owned accounts.

该提案为存储市场参与者添加了新的方法，让客户提出链上交易，然后由供应商确认该交易。
这补充了适用于外部拥有的帐户的存储交易的现有一步发布。

## Change Motivation 变更动机
Interacting with storage markets is likely to be a significant use case for FVM actors.
Examples of contracts that might act as deal clients include actors representing a group (multisig), 
more complex DAOs, decentralised replication/repair services, 
or brokering actors such as auction or bounty contracts built on top of the built-in market.

与存储市场的互动可能是FVM参与者的一个重要用例。
可能充当交易客户的契约示例包括代表一组（multisig）的参与者、更复杂的DAO、分散的复制/修复服务或中介参与者，
如基于内置市场的拍卖或赏金契约。

Without making a change like this, storage deals will remain restricted to external account owners, 
which is a significant barrier to automation and trust-minimised systems.

如果不做出这样的改变，存储交易将仍然限于外部账户所有者，这是自动化和信任最小化系统的一个重大障碍。

## Specification 规范
Add new state and methods to the built-in storage market actor.

为内置存储市场参与者添加新的状态和方法。

The existing code already uses the term "proposals" to describe the collection of deals
that both parties have already committed to,
and "pending" to describe any deal that has not yet completed.
To avoid confusion, the on-chain proposals are termed "offers".

现行法规已经使用“提案”一词来描述双方已经承诺的交易集合，“待定”一词用于描述尚未完成的任何交易。
为避免混淆，链上提案被称为“报价”。

### State 状态
```go
type State struct {
    // Existing state remains as-is.
    
    // Offers by clients for specific deal terms, not yet accepted by the provider.
    // The first key is the client actor ID.
    // Offers are grouped by client so the client bears costs of traversing the 
    // deals collection if it gets large.
    Offers cid.Cid // HAMT[ActorID]HAMT[abi.DealID]DealProposal
}
```

Offers are identified by integers in the same namespace as other deals, so that the identifier
remains constant across the phases of offer and acceptance by the storage provider.

要约由与其他交易相同名称空间中的整数标识，因此标识符在要约和存储提供商接受的各个阶段保持不变。

### Methods 方法

```go
 type OfferStorageDealsParams struct {
 	Offers []DealProposal
 	AllowPartialSuccess bool
 }
type OfferStorageDealsReturn struct {
	Success []bool // Array of success indicators, parallel to parameters
	IDs []abi.DealID // Array of IDs for the proposals, parallel to parameters
}

// Offers a set of storage deals from a client to specific providers.
// Each proposal offered must specify the client as the calling actor, but may specify different providers.
// A proposal fails if it is internally invalid or expired.
// A proposal fails if an identical proposal already exists as a pending proposal (submitted here or via PublishStorageDeals).
// The call aborts if AllowPartialSuccess is false and any proposal is invalid.
// Client collateral and payment are locked immediately; the call aborts if 
// insufficient client funds are available for all otherwise-valid offers.
func (Actor) OfferStorageDeals(params *OfferStorageDealsParams) *OfferStorageDealsReturn {
    // Validate proposals.
    // Allocates IDs by incrementing state.NextID.
    // Transfers client collateral and prepayment from escrow to locked.
    // Deducts data cap for verified deals.
    // Records each proposal in state.Offers.
    // Records the CID of each proposal in state.PendingProposals.
    // Schedules a deal op for the deal start epoch (for expiration), with jitter.
}

type AcceptStorageDealsParams struct {
    DealIDs []abi.DealID
}
type AcceptStorageDealsReturn struct {
    Accepted []abi.DealID
    Failed []abi.DealID
}

// Accepts a set of storage deals by a provider, committing both parties.
// Each ID must reference a proposal currently offered and specifying as the provider a 
// miner actor for which the caller is the owner, worker, or control address.
// All offers must specify the same provider.
// Offers that specify a different provider or have already expired fail individually.
// Provider collateral is locked immediately; the call aborts if insufficient provider funds are
// available to accept all offers that would otherwise succeed. 
func (Actor) AcceptStorageDeal(params *AcceptStorageDealsParams) *AcceptStorageDealsReturn {
    // Validate offers match the provider.
    // Transfers provider collateral from escrow to locked.
    // Removes proposals from state.Offers, adds to state.Proposals
    // CID remains in state.PendingProposals, and there's already a deal op scheduled.
}

type CleanupOffersParams struct {
    Client ActorID
    Offers Bitfield
}
type CleanupOffersReturn struct {
}

// Deletes offers for a client that have passed their start epoch without being accepted by the provider.
// If the `Offers` bitfield is not empty, loads only proposals with IDs specified by that parameter.
// If `Offers` is empty, loads all proposals for the client.
// A proposal that is loaded but not yet expired is left intact.
// Cleaning up expired offers will reduce the gas cost of future operations by the client.
func (Actor) CleanupOffers(params *CleanupOffersParams) *CleanupOffersReturn {
    // If offers is not empty, load each proposal specified by it in turn, and remove if expired.
    // If offers is empty, traverse the full collection.
}
```

A client cannot retract an offer, but must wait for its start epoch to pass.

客户不能撤回报价，但必须等待其开始时间过去。

## Design Rationale 设计原理
This proposal demonstrates a pattern of ad-hoc account abstraction, 
providing a means for an actor to do something without providing a signed message.
We will likely see this kind of pattern repeated in other actors until Filecoin offers
a first-class account abstraction primitive.

该方案演示了一种特殊帐户抽象模式，为参与者提供了一种无需提供签名消息就可以做某事的方法。
我们可能会在其他参与者中看到这种模式的重复，直到Filecoin提供了一流的帐户抽象原语。

An alternative to this proposal is implementing account abstraction, 
and then altering the built-in market actor to delegate signature checking to the client actor.

该方案的另一个替代方案是实现帐户抽象，然后改变内置的市场参与者，将签名检查委托给客户参与者。

### Partial failure 局部故障
The client can choose whether to offer a collection of deals atomically, or allow only some offers
to succeed.
This route is chosen to provide flexibility to client contracts.

客户可以选择是以原子方式提供一系列交易，还是只允许一些交易成功。
选择此路线是为了为客户合同提供灵活性。

The provider's call to accept deals can partially succeed.
The reasons for this are similar to the reasons partial success was added to `PublishStorageDeals`.
Publishing or accepting deals is often part of a complex sector commitment workflow that will
roll forward regardless of individual deal failures.

提供商接受交易的呼叫可能部分成功。其原因与`PublishStorageDeals`中添加部分成功的原因类似。
发布或接受交易通常是复杂扇区承诺工作流程的一部分，无论个别交易失败，该工作流程都会向前推进。

Calls always fail completely if insufficient collateral is available, so that the caller can
choose to either select some offers to make/accept or to top up collateral to make progress.
Any offer selection policy implemented on-chain would be unsuitable to some providers,
or unnecessary complexity.

如果没有足够的抵押品，呼叫总是完全失败，因此呼叫者可以选择提供/接受一些报价，或者补充抵押品以取得进展。
链上实施的任何报价选择策略都不适用于某些供应商，或不必要的复杂性。

### Future extensions 未来扩展
This mechanism can be extended in the future to allow clients to authorize other addresses to
offer deals on their behalf (similar to Ethereum ERC-20/721 token authorizations).
This would change the behaviour but not the type signature of `OfferStorageDeals`.
(First-class account abstraction primitives would make such an extension unnecessary).

该机制可以在未来扩展，以允许客户授权其他地址代表其提供交易（类似于以太坊ERC-20/721令牌授权）。
这将改变行为，但不会改变`OfferStorageDeals`的类型签名。（第一类帐户抽象原语会使这种扩展变得不必要）。

A similar extension could be made to allow providers to nominate addresses to accept deals on their behalf.

可以做出类似的扩展，允许供应商指定地址代表其接受交易。

This mechanism may be easily extended to support offers that do not specify a provider.
Such an offer would be available to any storage provider to accept (like a storage bounty).

该机制可以很容易地扩展到支持不指定提供者的报价。
任何存储提供商都可以接受这样的提议（如存储奖励）。

## Backwards Compatibility 向后兼容性
This proposal leaves all existing methods intact. 
Clients and storage providers that do not wish to use the new functionality need not make any changes.

该提案保留了所有现有方法。不希望使用新功能的客户端和存储提供商无需进行任何更改。

This proposal changes the state schema for the storage market actor, 
and so requires migration as part of a network upgrade.

此建议更改了存储市场参与者的状态模式，因此需要将迁移作为网络升级的一部分。

## Test Cases 测试用例
To be provided during implementation.

将在实施期间提供。

## Security Considerations 安全考虑
None.

## Incentive Considerations 激励因素
None.

## Product Considerations 产品考虑
The primary motivation for this proposal is the product benefit of supporting contracts acting as deal clients.

该提案的主要动机是作为交易客户支持合同的产品利益。

Non-contract deal clients can use these new methods too, if they wish. 
As compared with `PublishStorageDeals`, the `OfferStorageDeals` flow:  
非契约交易客户也可以使用这些新方法，如果他们愿意的话。
与`PublishStorageDeals`相比，`OfferStorageDeals`流：
- involves the client to posting a message on chain, rather than sending it to the provider out of band;
- 涉及客户端在链上发布消息，而不是将其发送到带外提供商；
- uses slightly more total gas, split between the client and the provider, 
rather than provider paying all gas fees;
- 使用稍多的总天然气，由客户和供应商分摊，而不是供应商支付所有gas费用；
- is cheaper in gas for providers;
- gas对供应商而言是否更便宜；
- uses significantly _less_ total gas if the client is a BLS address.
- 如果客户端是BLS地址，则使用的总gas显著_更少_。

In both flows, an offer cannot be retracted by a client, and thus may be relied upon by the provider.

在这两种流程中，客户不能撤回报价，因此供应商可能会依赖报价。

## Implementation 实现
To be provided before "Final" status.

在“最终”状态之前提供。

Implementation of this proposal will likely target the WASM actors running in the FVM.

该提案的实施可能会针对在FVM中运行的WASM参与者。

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
