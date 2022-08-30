---
fip: "0037"
title: Gas model adjustment for for user programmability
author: Raúl Kripalani (@raulk), Steven Allen (@stebalien)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/316
status: Draft
type: Technical
category: Core
created: 2022-03-09
spec-sections:
  - TBD
requires:
  - FIP-0030
  - FIP-0031
  - FIP-0032
replaces: N/A
---

# FIP-0037: Gas model adjustment for user programmability

## Simple Summary 简述

This FIP expands on FIP-0032 (Gas model adjustment for non-programmable FVM). It
introduces new gas fees to cover the costs of functionalities and system tasks
related to the deployment and execution of user-deployed on-chain actors. It
also prices syscalls that were previously free for built-in actors.

本FIP扩展了FIP-0032（不可编程FVM的气体模型调整）。
它引入了新的gas费用，以支付与部署和执行用户部署在链参与者上相关的功能和系统任务的成本。
它还为以前对内置参与者免费的系统调用定价。

## Abstract 摘要

This FIP introduces gas fees for actor deployment, actor loading, and actor
memory expansion. It modifies existing fees, such as the hashing fee, which is
now made dynamic. Finally, it prices syscalls that are currently unpriced, and
adjusts the permissions on syscalls that are now considered privileged.

该FIP为参与者部署、参与者加载和参与者内存扩展引入了gas费用。
它修改了现有的费用，例如散列费，现在它是动态的。
最后，它为当前未定价的系统调用定价，并调整现在被视为特权的系统调用的权限。

## Change Motivation 变更动机

The ability to deploy user-defined actors results in new workloads and tasks for
the network to execute. These introduce new resource consumption pressures in
terms of compute, memory, chain bandwidth, state footprint, and other axes. User
transactions triggering these workloads need to be subjected to gas fees in
order to compensate the network for this new work, as well as to size blocks
appropriate to sustain the 30s epoch time.

部署用户定义的参与者的能力为网络执行新的工作负载和任务。
这些在计算、内存、链带宽、状态占用和其他轴方面引入了新的资源消耗压力。
触发这些工作负载的用户事务需要缴纳天然气费用，以补偿网络的这项新工作，并确定适合维持30秒历元时间的区块大小。

## Specification 规范

Here’s an overview of proposed changes to the existing gas model. We explain
each one in detail in the following sections.

以下是对现有天然气模型的拟议变更概述。我们将在下面的章节中详细解释每一个。

- Introduce a fee for *actor installation,* the act of deploying of new actor
  code to the chain.
- 为*参与者安装*引入一项费用，将新参与者代码部署到链中的行为。
- Introduce a fee for *actor loading,* the act of loading an actor’s Wasm module
  to handle a call.
- 为*参与者加载*引入一项费用，加载参与者的Wasm模块来处理调用的行为。
- Introduce memory expansion gas.
- 引入内存扩展gas。
- Adjust the hashing fee to vary depending on digest function and length of
  plaintext.
- 根据摘要函数和明文长度调整哈希费。
- Price currently unpriced syscalls.
- 当前未定价的系统调用的价格。
- Restrict the charge_gas syscall.
- 限制charge_gas系统调用。

### Actor installation fee 参与者安装费

We specify an actor installation fee. This fee covers the cost of accepting Wasm
bytecode submitted by a user, persisting it, and preparing it for execution.

我们指定演员安装费。该费用包括接受用户提交的Wasm字节码、保存该字节码以及准备执行该字节码的费用。

This includes:

这包括：

- Storing the Wasm bytecode in the state tree.
- 将Wasm字节码存储在状态树中。
- Validating the Wasm bytecode: includes the cost of performing static code
  analysis to validate its structure, integrity, and conformance with system
  policies (e.g. number of functions, maximum stack length, usage of Wasm
  extensions, etc.)
- 验证Wasm字节码：包括执行静态代码分析以验证其结构、完整性和与系统策略的一致性（
  例如，函数数量、最大堆栈长度、Wasm扩展的使用等）的成本
- Potentially optimizing the Wasm bytecode.
- 潜在地优化Wasm字节码。
- Weaving in the gas metering logic.
- 在gas计量逻辑中交织。
- Compiling the Wasm bytecode into machine code.
- 将Wasm字节码编译成机器代码。
- Potentially progressively optimizing the machine code.
- 潜在地逐步优化机器代码。

**Pricing formula**

`<<TODO>>`

### Actor loading fee 参与者加载费用

We introduce an actor loading fee. In the FVM, all Wasm bytecode is compiled and
cached as machine code on disk (notwithstanding that clients may apply caching
policies to retain the hottest actors in memory).

我们引入参与者加载费。在FVM中，所有Wasm字节码都被编译并缓存为磁盘上的机器代码
（尽管客户端可能会应用缓存策略将最热门的参与者保留在内存中）。

For every non-value transfer message[^1], the actor’s precompiled module must be
loaded and the actor’s entrypoint must be invoked. The actor loading fee covers
that cost, and is computed over the length of the original Wasm bytecode[^2] and
complexity factors. These factors are determined at deployment time through
static code analysis, and include:

- number of imported syscalls
- number of globals
- number of exported entrypoints[^3]

The loading fee is additive to the extant *call fee*: loading bytecode and
invoking the loaded bytecode are two separate acts. In fact, an actor loaded
once can be invoked multiple times during a single call stack. This FIP proposes
no fee discount/exemption on deduplicated actor loading (such as on
recursion/reentrancy), but we may consider it in the future.

**Pricing formula**

`<<TODO>>`

### Memory expansion gas

FVM actors are allowed to allocate memory within predefined bounds. Wasm memory
is measured and allocated in pages. A page equates to 64KiB of memory. Page
usage is metered and limited per call; reentrant or recursive calls are
independent of one another for purposes of memory consumption.

Starting with this FIP, actor invocations are granted 1 page for free (64KiB),
and the cost of that page is accounted for in the call fee. This avoids an
immediate memory expansion operation on every invocation, as most actors need
*some* memory to do useful work.

An actor can use any number of pages, but the total page count for the entire
call stack cannot exceed 8192 pages, i.e. 512MiB. The next allocation must fail
with an `SysErrOutOfMemory`.

We introduce a memory expansion fee. Today, we charge a fee for the memory
expansion operation, dependent on the number of pages requested at a time. This
is to account for the memory zeroing cost on allocation.

This fee does not aim to model the cost of memory *usage*, because the presence
of a hard limit in a sequential execution environment means that all
implementations must preallocate and reserve the maximum amount of allocatable
memory (512MiB) anyway. So, in this context, memory is not a shared or scarce
resource to tax[^4].

`<<TODO: consider moving the memory policy to a FIP of its own>>`

**Pricing formula**

`<<TODO>>`

### Multihash-dependent hashing fee

Today’s syscall for hashing assumes a Blake2b-256 digest function. This syscall
will be generalized, so it can hash over a predetermined set of multihash digest
functions. Because the computational effort varies per digest function, so does
the updated hashing fee.

`<<TODO: specify the syscall generalization in a FIP of its own, or bundle with
memory policy>>`

**Pricing formula**

`<<TODO>>`

### Price currently unpriced syscalls

In Filecoin network version 15 and below, these syscalls (or equivalent
operations) are unpriced. With this FIP, they will acquire a price:

- `network::epoch, network::version`, `network::base_fee`, `message::caller`,
  `message::receiver`, `message::method_number`, `message::value_received` ,
  `debug::debug_enabled`
    - These syscalls retrieve static, contextual information. They are charged a
      flat syscall fee of <<TODO: value>>.
- `actor::get_actor_code_cid`, `actor::new_actor_address` ,
  `self::current_balance()`
    - These syscalls query the actor’s node in the state tree. They are charged
      a flat syscall fee of <<TODO: value>>.
- `rand::get_randomness_from_tickets` , `rand::get_randomness_from_beacon`:
    - These syscalls access randomness. They do so traversing the FVM<>node
      boundary. The cost of these syscalls is proportional to the length of
      their input, but not to the lookback because nodes are expected to cache
      randomness up to the maximum lookback in a ring buffer like data
      structure.
    - `<<TODO: limit the lookback>>`
    - The cost of these syscalls is: <<TODO: formula>>.

### Restrict usage of `gas_charge` syscall

This syscall is available to all actors, but it will be restricted such that
only built-in actors can effectively make use of it. User-deployed actors will
have their transactions aborted with exit code `SysErrForbidden` when attempting
to invoke this syscall.

## Design Rationale

`<<TODO>>`

## Backwards Compatibility

`<<TODO>>`

## Test Cases

`<<TODO>>`

## Security Considerations

`<<TODO>>`

## Incentive Considerations

`<<TODO>>`

## Product Considerations

`<<TODO>>`

## Implementation

For clients relying on the reference FVM implementation
([filecoin-project/ref-fvm](https://github.com/filecoin-project/ref-fvm)), the
implementation of this FIP will be transparent, as it is self-contained inside
the FVM.

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Footnotes

[^1]: Ideally, it would depend on the length of the compiled *machine code* (as
    this is what’s effectively loaded at invocation time). But because machine
    code is compiler- and platform-dependent, that approach is not feasible.

[^2]: Today, value transfers are handled entirely inside the VM. In the future,
    we may want to delegate to the actor. This could be achieved by introducing
    the notion of a *value transfer entrypoint:* a publicly exported function
    invoked to handle messages with no payload. At deployment time, the FVM
    would need to memorize if a *value transfer entrypoint* exists, to determine
    when actor code should be loaded and when not.

[^3]: Currently, actors only support one entrypoint: the `invoke` function.
    Support for multiple entrypoints may be added in the future to handle value
    transfers, code upgrades, and lazy state migrations.

[^4]: This will change once Filecoin switches to parallel execution, and memory
    becomes a shared resource that determines scheduling decisions and task
    bin-packing. But even under that model, it is unlikely for memory usage
    costs to be folded into gas.
