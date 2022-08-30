---
fip: 0039
title: Filecoin Message Replay Protection
author: Afri Schoedon (@q9f)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/301
status: Draft
type: Technical
category: Core
created: 2022-02-16
---

## Simple Summary 简述
Defines Filecoin chain identifiers and adds them to unsigned messages and signatures to prevent transactions to be replayed across different Filecoin networks.

定义Filecoin链标识符，并将其添加到未签名的消息和签名中，以防止事务在不同的Filecoin网络上重播。

## Abstract 摘要
The proposal adds a field to unsigned messages indicating which chain a transaction is intended for. In addition, signatures encode the chain they are targeted for to prevent clients from accepting signed messages to be replayed across all Filecoin networks.

本方案在未签名消息中添加了一个字段，指示事务用于哪个链。此外，签名对其所针对的链进行编码，以防止客户端接受要在所有Filecoin网络上重播的签名消息。

## Change Motivation 更改动机
Currently, transactions signed for testnets are also valid on mainnet and could be _replayed_. The motivation of this specification is to prevent that by specifically introducing network identifiers for all Filecoin main and test networks and adding them explicitly to the unsigned messages and signatures.

目前，为测试网签名的事务在主网上也有效，可以_回放_。本规范的目的是通过专门为所有Filecoin主网络和测试网络引入网络标识符，并将其显式添加到未签名消息和签名中来防止这种情况。

## Specification 规范
### Chain Identifiers 链ID
The following chain IDs **MUST** be defined for the Filecoin networks.
- Filecoin: `CHAIN_ID := 1` (TBD)
- Interopnet: `CHAIN_ID := 10` (TBD)
- Butterlfynet: `CHAIN_ID := 20` (TBD)
- Calibnet: `CHAIN_ID := 30` (TBD)
- Devnet: `CHAIN_ID := 4269` (TBD)

### Unsigned Messages 未签名的消息
The unsigned message **SHOULD** be appended by an additional field `chain_id`. Legacy messages remain valid but **SHOULD** be deprecated.

未签名消息**应**被附加一个字段`chain_id`。旧消息仍然有效，但**应**被弃用。

```pyhton
from dataclasses import dataclass

@dataclass
class UnsignedMessageLegacy:
  version: int = 0
  from: int = 0
  to: int = 0
  sequence: int = 0
  value: int = 0
  method_num: int = 0
  params: bytes()
  gas_limit: int = 0
  gas_fee_cap: int = 0
  gas_premium: int = 0

@dataclass
class UnsignedMessageFip9999:
  version: int = 0
  from: int = 0
  to: int = 0
  sequence: int = 0
  value: int = 0
  method_num: int = 0
  params: bytes()
  gas_limit: int = 0
  gas_fee_cap: int = 0
  gas_premium: int = 0
  chain_id: int = 0
```

### Signatures 签名
The signature of a message **SHOULD NOT** contain a recovery ID (or _"y-parity"_); instead it **MUST** contain a value `v` that encodes both the `chain_id` and the `recovery_id`.

消息的签名**不应**包含恢复ID（或_"y-parity"_）；相反，它**必须**包含一个值`v`，该值同时编码`chain_id`和`recovery_id`。

```python
v = recovery_id + CHAIN_ID * 2 + 35
```

A signature is a serialization of the fields `r | s | v`. Note that `v` value is appended not prepended.

签名是字段`r | s | v`的序列化。请注意，`v'值是附加的，而不是前缀。

## Design Rationale 设计原理
Both Bitcoin and Ethereum use `v` instead of a recovery ID. Bitcoin uses a value of `v` in `{27,28,29,30,31,32,33,34}` to allow recovering public keys on all four possible points on the elliptic curve for both uncompressed and compressed public keys. This does not apply to Filecoin, however, note that the `+ 35` offset in the `v` specifically allows to distinguish between Bitcoin and Filecoin signatures.

比特币和以太坊都使用`v`而不是恢复ID。比特币在`{27,28,29,30,31,32,33,34}`中使用`v`，以允许在椭圆曲线上的所有四个可能点上恢复未压缩和压缩公钥。这不适用于Filecoin，然而，请注意`v`中的`+ 35`偏移量特别允许区分比特币和Filecoin签名。

Ethereum uses a similar replay protection in [EIP-155](https://eips.ethereum.org/EIPS/eip-155). It **COULD** be considered to choose a set of `CHAIN_ID`s that do not conflict with EVM chains. (TBD)

以太坊在[EIP-155](https://eips.ethereum.org/EIPS/eip-155)中使用了类似的重放保护. 可以**考虑**选择一组与EVM链不冲突的`CHAIN_ID`。（待定）

The `v` value will be encoded at the end of the signature to allow for the easy determination of `v` values for `v > ff`.

`v`值将在签名的末尾进行编码，以便于确定`v > ff`的`v`值。

## Backwards Compatibility 向后兼容性
FIP9999 Messages and Signatures are not backward compatible and there will always be the need to maintain both, legacy transactions and FIP9999 transactions.

FIP9999消息和签名不向后兼容，因此始终需要同时维护遗留事务和FIP999事务。

The main breaking changes are:  
主要中断变化为：
* one additional field of `chain_id` in the unsigned message
* 未签名消息中的一个附加字段`chain_id`
* using `v` instead of `recovery_id` in the signatures
* 在签名中使用`v`代替`recovery_id`
* moving the `v` value to the end of the signature.
* 将`v`值移动到签名的末尾。

## Test Cases
_Test Cases can be provided after further discussion of the proposal._

## Security Considerations 安全考虑
This proposal supposedly enhances chain and user security by preventing replay protection across different Filecoin networks.

本方案通过防止不同Filecoin网络之间的重放保护，增强了链和用户安全。

Notably, EIP-155 has been in use by Ethereum for more than five years and has proven robust.

值得注意的是，EIP-155已经被以太坊使用了五年多，并且已经证明是稳健的。

## Incentive Considerations 激励因素
This FIP does not affect incentives in itself and is of pure technological nature without any impact on economic factors.

本FIP本身并不影响激励措施，属于纯技术性质，对经济因素没有任何影响。

## Product Considerations 产品考虑
Applications and products should be encouraged to adapt this FIP as it enhances end-user security; at the same time legacy transactions should be discouraged.

应鼓励应用程序和产品采用本FIP，因为它增强了最终用户的安全性；同时，应阻止遗留事务。

However, since legacy transactions are not invalid for the time being, the impact on applications is fairly low.

但是，由于遗留事务暂时不是无效的，因此对应用程序的影响相当小。

## Implementation 实施
_An implementation can be provided after further discussion of the proposal._

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
