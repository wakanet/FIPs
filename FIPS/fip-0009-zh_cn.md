---
fip: "0009"
title: Exempt Window PoSts from BaseFee burn
author: Steven Allen (@stebalien), Molly Mackinlay (@momack2), Łukasz Magiera (@magik6k), Zixuan Zhang (@zixuanzh)
discussions-to: https://github.com/filecoin-project/FIPs/issues/52
status: Final
type: Technical
category: Core
created: 2020-12-15
spec-sections:
  - section-systems.filecoin_vm.gas_fee
---

## Simple Summary 简述

Exempt direct `SubmitWindowedPoSt` messages that execute successfully from base-fee burn (i.e., don't burn `baseFee*gasUsed`).

将成功执行的直接`SubmitWidowedPost`消息从基本费用燃烧中免除（即，不燃烧`baseFee*gasUsed`）。

## Abstract

If a miner sends a direct on-chain message to a miner actor's "SubmitWindowedPoSt" method, and the message executes successfully, immediately "refund" all the burned gas, instead of just refunding the overestimation. However, overestimation penalties, gas premium, etc. still apply.

## Change Motivation

This proposal is a short-term stopgap to reduce the impact of rising base fee on continuously proving existing storage. In principle, BaseFee*GasUsage is burned to compensate the network for the resources consumed by messages and ensure incentive alignment. However, SubmitWindowPoSt is a required message to continue mining operations in Filecoin. 

* Long-term solutions for reducing the cost of Window PoSt include https://github.com/filecoin-project/FIPs/issues/42.
* Long-term solutions for reducing chain congestion include https://github.com/filecoin-project/FIPs/issues/49, https://github.com/filecoin-project/FIPs/issues/50.

Unfortunately, such long-term solutions are complex and cannot be properly implemented and tested on the required timescale (before the holidays). This proposal is a short-term mitigation until those solutions are ready.

## Specification

If an _account_ (not a multisig, payment channel, etc) sends a direct on-chain message to a miner actor's `SubmitWindowedPoSt` method and the message succeeds, immediately "refund" all the burned gas, instead of just refunding the overestimation. DO NOT refund, overestimation penalties, gas premium, etc.

## Design Rationale

This design was chosen as the least-invasive stopgap solution to the problem of expensive window posts. It directly removes the majority of the fees related to Window PoSt messages without affecting block validation times.

## Backwards Compatibility

This FIP requires a network upgrade at a specific epoch to ensure that all node operators abide by the new pricing rules after that epoch.

## Test Cases

After the change, the "GasCost" of a successful, direct `SubmitWindowedPoSt` message must change from:

```json
{
  "Message": {
    "/": "bafy2bzacedhftxcnozkaeleau4szu2vpvksbjs2xkpracp6xvdog6gucnzxdc"
  },
  "GasUsed": "125122238",
  "BaseFeeBurn": "12512223800",
  "OverEstimationBurn": "466093200",
  "MinerPenalty": "0",
  "MinerTip": "15543374565710",
  "Refund": "151788019038",
  "TotalCost": "15556352882710"
}
```

To:

```json
{
  "Message": {
    "/": "bafy2bzacedfyp5mz6le43iifaviw2g7hgiqr3eet2bfdzffbitphyubjzzgpy"
  },
  "GasUsed": "125122238",
  "BaseFeeBurn": "0",
  "OverEstimationBurn": "466093200",
  "MinerPenalty": "0",
  "MinerTip": "15723304407057",
  "Refund": "164300242838",
  "TotalCost": "15723770500257"
}
```

Specifically, the `BaseFeeBurn` is 0.

All other messages, _including_ messages that indirectly invoke `SubmitWindowedPoSt` (e.g., from a multisig) or contain an unsuccessful Window PoSt, must retain the prior gas fees.

## Security Considerations

This FIP effectively removes the base-fee "auction" from Window PoSt messages given Window PoSt congestion. This means the base-fee could rise unchecked because Window PoSt messages are not sensitive to the base-fee. However, miners may only submit one windowed post per partition, per day. This puts an upper bound on the amount of congestion we expect from Window PoSt messages.

Furthermore, any chain congestion due to Window PoSt messages will increase the base-fee, pricing out other messages that are sensitive to base-fee hikes. This will leave room for more Window PoSt messages, clearing them out faster and allowing the base-fee to drop. This only works because Window PoSt messages are rate-limited.

For these reasons, we consider this solution to be, at most, an acceptable _stopgap_. However:

1. It is not a final solution: It does not improve chain throughput or reduce the cost of verifying Window PoSt messages on the network as a whole.
2. It is not an acceptable solution for any other message type: Window PoSts are special, rate-limited, "system" messages.

## Incentive Considerations

Base Fee serves as a signal to message senders on what the market clearing price is. Exempting BaseFee burn for SubmitWindowPoSt is effectively removing the posted price mechanism (Base Fee) of EIP1559 for `SubmitWindowedPoSt`. 
Per latest analysis by [Tim Roughgarden](https://timroughgarden.org/), Base Fee tend to be under the market clearing price when demand for the network rises sharply. However, SubmitWindowedPoSt is a required message of the protocol and it can be argued that it deserves special treatment or below market rate relative to onboarding new storage or other “optional” network messages.

Given that we are not removing Gas Burn entirely, message selection works as before from the perspective of the block producing miner. Miners are selecting messages based on FeeCap/GasLimit, where FeeCap includes BaseFee and GasPremium.

However, from the perspective of message senders, priority within WindowPoSt messages dissolves into submitting bids for the first price auction of GasPremium: block producing miners may include their messages for free, while smaller miners may have to pay a higher GasPremium to get their messages included. However, this is also the case in the current protocol. While there is no guarantee that this proposal will disproportionately help smaller miners, this proposal at least alleviates the burden of fee burn for all miners for a message that is crucial to the healthy functioning of the economy.

In the event of network demand rising sharply, BaseFee will spike and likely price out other messages that are sensitive to BaseFee hikes. Given that  SubmitWindowPoSt is exempt from BaseFee burn, default message selection logic will still pick these messages, thereby allocating more chain bandwidth for SubmitWindowPoSt and allowing them to go through. If a high BaseFee persists, messages other than SubmitWindowPoSt will be priced out and SubmitWindowPoSt can be cleared more quickly, leading to a drop in BaseFee.


## Product Considerations

After the network upgrade, miners may need to raise their fee-cap for Window PoSt messages to account for changes to the network's base-fee. However, given this change, this will not _significantly_ impact the cost of a Window PoSt message.

NOTE: The base-fee will still affect the overestimation burn and miner penalties for including Window PoSt messages with fee-caps under the base-fee.

As this stopgap does not address the rising base-fee, prove-committing sectors will continue to be expensive unless the base-fee drops.

## Implementation

PENDING PR

## Related Work

* https://github.com/filecoin-project/FIPs/issues/24#issuecomment-736923109 - Steven Li (@steven004)

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
