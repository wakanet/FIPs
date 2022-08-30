---
fip: "0044"
title: Standard Authentication Method for Actors
author: Aayush Rajasekaran (@arajasek), Alex North (@anorth)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/413
status: Draft
type: Technical (Interface)
created: 2022-08-02
replaces: [FIP-0035](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0035.md)
---

## Simple Summary 简述

The Filecoin protocol has entities called "actors" that can be involved in computation on the Filecoin blockchain. 
There are various different kinds of actors; among the most common are accounts and multisigs. 
With [the planned introduction of the Filecoin Virtual Machine](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md), there will be many new kinds of user-defined actors.
Unfortunately, today, the only kind of actors that can "authenticate" a piece of data are account actors -- they do so through a signature validation.
This FIP is the starting point to evolve a convention by which any actor can verify such authenticity. It simply proposes adding a special "authenticate" message that actors can choose to implement. 
If they do implement such a method, it can be used by other actors to authenticate pieces of data.

Filecoin协议具有称为"参与者"的实体，可以参与Filecoin区块链上的计算。
有各种各样的参与者；其中最常见的是账户和多签。
随着[计划引入Filecoin虚拟机](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md)，将有许多新类型的用户定义参与者。
不幸的是，今天，唯一能够"验证"数据的参与者是帐户参与者——他们通过签名验证来验证。
本FIP是一个起点，可以发展一种任何参与者都可以验证这种真实性的约定。它只是建议添加一个特殊的"身份验证"消息，参与者可以选择实现。
如果他们确实实现了这样的方法，其他参与者可以使用它来验证数据片段。

## Abstract 摘要

There is a need for arbitrary actors to be able to verify (to other actors) that they have approved of a piece of data. This need will become more pressing with the Filecoin Virtual Machine.
This FIP proposes adding an `Authenticate` method that any actor can choose to implement. 
This method can be called by any other actor to verify the authenticity of a piece of data. This is a small change to the existing actors, and allows for user-defined actors to flexibly implement this method according to their needs.
Concretely, we add this method to the built-in `Account` actor, and have the `Storage Market` actor invoke this method,
in lieu of validating signatures directly.

任意参与者需要能够（向其他参与者）验证他们已经批准了一段数据。随着Filecoin虚拟机的出现，这种需求将变得更加迫切。
本FIP建议添加任何参与者都可以选择实现的`身份验证`方法。
任何其他参与者都可以调用此方法来验证数据的真实性。这是对现有参与者的一个小改动，允许用户定义的参与者根据其需求灵活地实现此方法。
具体来说，我们将此方法添加到内置的`Account`参与者中，并让`Storage Market`参与者调用此方法，
代替直接验证签名。

## Change Motivation 变更动机

We want any actors, including user-defined actors, to be able to authenticate arbitrary data.
If we don't have this ability, we will need to engineer around it in painful ways. 

我们希望任何参与者，包括用户定义的参与者，都能够验证任意数据。
如果我们没有这种能力，我们将需要以痛苦的方式围绕它进行设计。

The proposed [FIP-0035](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0035.md)
is a concrete example. It seeks to allow user-defined actors to serve as storage clients in the built-in `Storage Market` actor.
In order to enable this, it proposes adding several new methods and data types in order to work around the 
inability of user-defined actors to authenticate data.

拟定的[FIP-0035](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0035.md)
是一个具体的例子。它试图允许用户定义的参与者在内置的`存储市场`参与者中充当存储客户端。
为了实现这一点，它建议添加几个新方法和数据类型，以解决用户定义的参与者无法验证数据的问题。

It will be infeasible to modify the builtin-actors themselves every time we want to be able to authenticate some new kind of information, and doing so will require more total messages 
to be landing on the Filecoin blockchain. Instead, a flexible, universal approach for actors to authenicate data would be preferred.

每次我们希望能够验证某些新类型的信息时，修改内置参与者本身是不可行的，这样做将需要更多的总消息登录到Filecoin区块链。
相反，更倾向于采用灵活、通用的方法，让参与者验证数据。

## Specification 规范

We propose adding the following method to the built-in `Account` Actor. 
This method can then be the blueprint for other builtin actors (eg. the multisig actor), 
as well as a template for user-defined actors.

我们建议将以下方法添加到内置的`帐户`参与者。
然后，这个方法可以是其他内置参与者（例如，multisig参与者）的蓝图，
以及用户定义角色的模板。

```
    /// Authenticates whether the provided input is valid for the provided message.
    /// Errors with USR_ILLEGAL_ARGUMENT if the authentication is invalid.
    pub fn authenticate_message(params: AuthenticateMessageParams) -> Result<(), ActorError>
```

The params to this method is a wrapper over the message and signature to validate:

此方法的参数是消息和签名的包装，用于验证：

```
pub struct AuthenticateMessageParams {
    pub authorization: Vec<u8>,
    pub message: Vec<u8>,
}
```

Further, we propose modifying the StorageMarketActor to call this method when validating `ClientDealProposal`s instead of validating signatures directly.

此外，我们建议修改StorageMarketActor，以便在验证`ClientDealProposal`时调用此方法，而不是直接验证签名。

## Design Rationale 设计原理

The idea is to create a flexible, lightweight endpoint that actors can expose to callers that might want to
validate its authorization. Some actors will have very obvious implementations: the account actor simply 
performs a signature validation, while an `m/n` multisig need only perform `m` signature validations.
Other actors might opt for creative implementations of such authorization based on their needs. Yet 
others can choose not to support this endpoint at all -- the Filecoin protocol's singleton builtin-actors
such as the Storage Power Actor, the System Actor, the Cron Actor, etc. will likely choose to omit this method.

其目的是创建一个灵活、轻量级的端点，参与者可以向可能希望验证其授权的调用方公开该端点。
一些参与者将有非常明显的实现：帐户参与者只是
执行签名验证，而“m/n”多签名只需执行“m”签名验证。
其他参与者可能会根据自己的需要选择这种授权的创造性实现。
然而，其他人可以选择根本不支持这个端点——Filecoin协议的单例内置参与者，
如存储能力参与者、系统参与者、Cron参与者等，可能会选择省略这个方法。

In order for this method to function as a standard for future actors, it needs a standardized invocation pattern. 
In the current dispatch model based on method numbers, that would be a standardized method number for `authenticate_message` methods. This FIP does not propose adopting a standard method number, preferring to rely on a new standardized calling convention, such as [FRC-0042](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0042.md) instead.

为了使该方法作为未来参与者的标准，它需要一个标准化的调用模式。
在基于方法编号的当前调度模型中，这将是`authenticate_message`方法的标准化方法编号。
本FIP不建议采用标准方法编号，更倾向于依赖新的标准化调用约定，
如[FRC-0042](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0042.md)代替。

## Backwards Compatibility 向后兼容性

Introducing new methods to the builtin-actors needs a network upgrade, but otherwise breaks no backwards compatibility.

向内置参与者引入新方法需要网络升级，否则不会破坏向后兼容性。

## Test Cases 测试用例

Proposed test cases include:

建议的测试用例包括：

- that the `authenticate_message` method on account actors can be called and succeeds with valid input
- 可以调用帐户参与者的`authenticate_message`方法，并在有效输入的情况下成功
- that the `authenticate_message` method on account actors can be called, but fails with invalid input
- 可以调用帐户参与者上的`authenticate_message`方法，但由于输入无效而失败
- that storage deals can be made with the changes to the `StorageMarketActor`
- 通过对`StorageMarketActor`的更改，可以进行存储交易

## Security Considerations 安全考虑

We need to be careful with the security of the `authenticate_message` implementations of the
builtin-actors. User-defined contracts are free to implement these methods as they see fit.

我们需要注意内置参与者的`authenticate_message`实现的安全性。
用户定义的契约可以在其认为合适时自由实现这些方法。

This FIP only proposes adding the `authenticate_message` method to the account actor, which 
simply invokes the signature validation syscall.

此FIP仅建议向帐户参与者添加`authenticate_message`方法，该方法仅调用签名验证系统调用。

## Incentive Considerations 激励因素

This proposal is generally well-aligned with the incentives of the network. It can be considered
as a simpler replacement for FIP-0035, with the only drawback being that this proposal as currently written does not allow for the existing built-in multisig actor to be a storage client. 

这项建议与网络的激励措施大体一致。
它可以被视为FIP-0035的一个更简单的替代品，
唯一的缺点是，目前编写的该提案不允许现有的内置多西格参与者成为存储客户端。

## Product Considerations 产品考虑

This proposal is the first step towards enabling non-account actors to function as clients for
storage deals, which is a major product improvement. Furthermore, it enables future innovation 
that relies on the ability of non-account actors to authenticate data as approved.

该提案是使非账户参与者能够作为存储交易的客户的第一步，这是一项重大的产品改进。
此外，它还支持未来的创新，这种创新依赖于非账户参与者验证批准数据的能力。

## Implementation 实现

Partial implementation [here](https://github.com/filecoin-project/builtin-actors/pull/502)

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
