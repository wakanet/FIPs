---
fip: "0032"
title: Gas accounting model adjustment for non-programmable FVM
author: Raúl Kripalani (@raulkr), Steven Allen (@stebalien), Jakub Sztandera (@Kubuxu)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/316
status: Draft
type: Technical
category: Core
created: 2022-03-08
spec-sections:
  - TBD
requires:
  - FIP-0030
  - FIP-0031
replaces: N/A
---

# FIP-0032: Gas accounting model adjustment for non-programmable FVM

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Simple Summary 简述](#simple-summary 简述)
- [Abstract 概要](#abstract 概要)
- [Change Motivation 变更动机](#change-motivation 变更动机)
- [Specification 规范](#specification 规范)
  - [Execution gas 执行的gas费](#execution-gas 执行的gas费)
  - [Syscall gas 系统调用的gas费](#syscall-gas 系统调用的gas费)
  - [Extern gas 外部的gas费](#extern-gas 外部的gas)
  - [IPLD state management gas IPLD状态管理的gas费](#ipld-state-management-gas IPLD状态管理的gas费)
  - [Milligas precision and under/overflow handling Milligas精度和欠/溢出处理](#milligas-precision-and-underoverflow-handling Milligas精度和欠/溢出处理)
- [Design Rationale 设计原理](#design-rationale 设计原理)
  - [Gas sizing baseline Gas分级基线](#gas-sizing-baseline Gas分级基线)
  - [Benchmark methodology 基准方法](#benchmark-methodology 基准方法)
- [Backwards Compatibility 向后兼容](#backwards-compatibility 向后兼容)
- [Test Cases 测试用例](#test-cases 测试用例)
- [Security Considerations 安全考虑](#security-considerations 安全考虑)
- [Incentive Considerations 激励因素](#incentive-considerations 激励因素)
- [Product Considerations 产品考虑](#product-considerations 产品考虑)
- [Implementation 实现](#implementation 实现)
- [Assets 资产](#assets 资产)
- [Annex A: Table of gas utilization increase per message type 附录A：每种消息类型的gas利用率增加表](#annex-a-table-of-gas-utilization-increase-per-message-type 附录A：每种消息类型的gas利用率增加表)
- [Annex B: Table of total final gas used per message type 附录B：每种消息类型使用的最终gas总量表](#annex-b-table-of-total-final-gas-used-per-message-type 附录B：每种消息类型使用的最终gas总量表)
- [Acknowledgements 致谢](#acknowledgements 致谢)
- [Copyright 版权](#copyright 版权)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Simple Summary 简述

This FIP is part of a two-FIP series that adjust the gas accounting model of the
Filecoin blockchain in preparation of user-programmable actors. This FIP focuses
on achieving high-fidelity gas accounting under the new Wasm runtime that
built-in system actors will migrate to as part of [FIP-0030] and [FIP-0031]. It
does not cover upcoming system features like actor deployment, actor dispatch,
actor loading, memory management, and more. Those will be covered in a future
FIP.

要FIP是两个FIP系列的一部分，这个系列调整了Filecoin区块链的gas核算模型，
为用户可编程参与者做准备。本FIP侧重于在新Wasm运行时下实现高保真gas记帐，
内置系统参与者将作为[FIP-0030]和[FIP-0031]的一部分迁移到该运行时。
它不包括即将出现的系统特性，如角色部署、角色调度、角色加载、内存管理等。
这些将在未来的FIP中涵盖。

## Abstract 概要

The current gas accounting model was designed to cover the execution of
predefined, trusted code developed by the Filecoin Core Devs.

当前的gas记账模型旨在涵盖Filecoin核心开发人员开发的预定义可信代码的执行。

With the arrival of the Filecoin Virtual Machine, the Filecoin network will gain
the capability to run untrusted code in the form of user-defined actors. Thus,
the gas accounting model needs to evolve to price arbitrary code.

随着Filecoin虚拟机的到来，Filecoin网络将获得以用户定义的参与者的形式运行不受信任代码的能力。
因此，gas记账模型需要发展到为任意代码定价。

This FIP evolves the current gas accounting model by introducing the concepts of
_execution gas_, _syscall gas_, and _extern gas_. Furthermore, this FIP revises
the IPLD state management fees to adapt to the new design.

本FIP通过引入_execution gas_、_syscall gas和_extern gas_。此外，本FIP修订了IPLD状态管理费，以适应新设计。

This FIP is designed to accompany [FIP-0030] and [FIP-0031] under the same network
upgrade (Milestone 1 of the FVM roadmap).

本FIP旨在在同一网络升级（FVM路线图里程碑1）下伴随[FIP-0030]和[FIP-0031]。

## Change Motivation 变更动机

Gas is an essential mechanism to (a) compensate the network for the computation
performed during the execution of transactions, (b) allocate chain bandwidth,
and to (c) restrict the compute capacity of blocks so that the designated epoch
time (30 seconds) can be sustained with contemporary hardware.

Gas是一种基本机制，用于
（a）补偿网络在事务执行过程中执行的计算，
（b）分配链带宽，以及
（c）限制块的计算容量，以便用当代硬件维持指定的历元时间（30秒）。

Today, transactions on the Filecoin network are charged gas on these operations:

今天，Filecoin网络上的交易在以下操作中收取费用：

- On including a chain message (variable gas units, based on length).
- 包括链消息（基于长度的可变gas单位）。
- On returning a value (variable gas units, based on length of returned
  payload).
- 返回值时（可变gas单位，基于返回有效载荷的长度）。
- On performing a call (variable gas units, depending on whether it's a simple
  value transfer, or an invocation to actor logic).
- 在执行调用时（可变gas单位，取决于它是简单的值传递还是对参与者逻辑的调用）。
- On performing an IPLD store get (fixed gas units).
- 在执行IPLD存储获取（固定gas单元）时。
- On performing an IPLD store put (variable gas units, based on block payload
  size).
- 在执行IPLD存储存入时（基于块有效载荷大小的可变gas单位）。
- On actor creation (fixed gas units).
- 在参与者创建时（固定gas单位）。
- On actor deletion (fixed gas units).
- 在参与者删除时（固定gas单位）。
- On cryptographic operations:
- 关于加密操作：
    - On signature verification (variable gas units, depending on signature type
      and plaintext length).
    - 签名验证（可变gas单位，取决于签名类型和明文长度）。
    - On hashing (fixed gas units).
    - 关于散列（固定gas单位）。
    - On computing the CID of an unsealed sector (variable gas units, based on
      proof type and piece properties).
    - 计算未密封扇区的CID（基于证明类型和件属性的可变gas单位）。
    - On verifying a seal (variable gas units, based on seal properties).
    - 验证密封时（基于密封特性的可变gas单位）。
    - On verifying a window PoSt (variable gas units, based on PoSt properties).
    - 验证PoSt窗口时（可变gas单位，基于PoSt属性）。
    - On verifying a consensus fault.
    - 在验证一致性错误时。

None of these gas charges accounts for the actual costs of the execution of the
actor's compute logic itself. This results in an imprecise resource accounting
system. So far, the network has coped without it because:

所有这些gas费用都没有考虑到执行参与者计算逻辑本身的实际成本。
这导致资源会计系统不精确。到目前为止，该网络在没有它的情况下应对，因为：

1. With the exception of some methods, the cost of most built-in actor logic is
   proportional to the cost of the above operations, most of which are triggered
   through syscalls.
1. 除了一些方法之外，大多数内置的参与者逻辑的成本与上述操作的成本成正比，其中大多数操作是通过系统调用触发的。
2. Only pre-determined, trusted logic runs on chain in the form of built-in
   actors.
2. 只有预先确定的可信逻辑以内置参与者的形式在链上运行。
3. Technical limitations. Prior to the FVM ([FIP-0030], [FIP-0031]), there was no
   portable actor bytecode whose execution could be metered identically across
   all implementations.
3. 技术限制。在FVM（[FIP-0030]、[FIP.0031]）之前，没有可移植的参与者字节码，其执行可以在所有实现中进行相同的计量。

With the ability to deploy user-defined actors on the FVM, these assumptions no
longer stand and it's vital to start tracking real execution costs with the
highest fidelity possible in order to preserve security guarantees.

由于能够在FVM上部署用户定义的参与者，这些假设不再成立，因此必须开始以尽可能高的保真度跟踪实际执行成本，以保持安全保障。

## Specification 规范

Here's an overview of proposed changes to the existing gas accounting model. We
explain each one in detail in the following sections.

以下是对现有gas记账模型的拟议变更的概述。我们将在以下章节中详细解释每一个。

- Introduce execution gas, metered by Wasm bytecode.
- 引入执行gas，由Wasm字节码计量。
- Introduce syscall gas.
- 引入系统调用气体。
- Introduce extern gas.
- 引入外部气体。
- Redesign the IPLD state management fees.
- 重新设计IPLD状态管理费。

### Execution gas 执行的gas费

Execution gas is charged per [Wasm instruction][wasm-instructions]. Every Wasm
instruction expends a number of execution units, which can be static or dynamic
based on its parameters.

执行gas按[Wasm指令][Wasm-instructions]收费。每一条Wasm指令都会消耗大量的执行单元，
根据其参数，这些执行单元可以是静态的，也可以是动态的。

As part of this FIP, we assign a flat value of 1 execution unit for every Wasm
instruction, with the exception of structural instructions. These instructions
are: `nop`, `drop`, `block`, `loop`, `unreachable`, `return`, `else` and `end`.
The exemption is due to the fact that these Wasm instructions do not translate
into machine instructions by themselves, or they are control flow instructions
with no associated logic.

作为本FIP的一部分，我们为每个Wasm指令分配1个执行单元的固定值，结构指令除外。
这些指令是：`nop`、`drop`、`block`、`loop`、`unreachable'、`return`、`else`和`end'。
这种豁免是由于这些Wasm指令本身不转换为机器指令，或者它们是没有关联逻辑的控制流指令。

**Pricing formula**

**计费模型**

The conversion of Wasm execution units to Filecoin gas units is at a fixed ratio
of 4gas per execution unit.

Wasm执行单元到Filecoin气体单元的转换率为每个执行单元4gas的固定比率。

This ratio is consistent with the average instruction throughput observed in
benchmarks, and tracks the baseline target of 10 gas units/ns of wall clock
time.

该比率与基准测试中观察到的平均指令吞吐量一致，并跟踪10 gas units/ns挂钟时间的基线目标。

```
p_gas_per_exec_unit := 4
```

**Future considerations**

**未来考虑**

1. Our benchmarks revealed that memory instructions perform differently to pure
   CPU-level instructions. They also exhibit high variance depending on the
   memory access pattern of the actor, CPU caching dynamics, and other factors.
   At this stage, we have opted to simplify the model by setting a fixed average
   rate for all instructions. But in the future, we may tabulate execution gas
   per instruction, or per instruction family.
1. 我们的基准测试表明，内存指令的性能与纯CPU级指令不同。
   根据参与者的内存访问模式、CPU缓存动态和其他因素，它们也表现出很大的差异。
   在这个阶段，我们选择通过为所有指令设置固定的平均速率来简化模型。
   但在未来，我们可能会将每个指令或每个指令族的执行气体制成表格。
2. The pricing policy for Wasm instructions and the conversion rate need to be
   assessed and revised continuously as Wasm compilers and hardware evolve. The
   ongoing goal is to sustain high execution fidelity.
2. Wasm指令的定价政策和转换率需要随着Wasm编译器和硬件的发展不断评估和修订。
   持续的目标是保持高执行保真度。

### Syscall gas 系统调用的gas费

Syscalls are host-provided functions exposed to actor code. They allow the actor
to traverse the sandbox in a controlled manner to:

系统调用是主机提供的函数，公开给参与者代码。它们允许参与者以受控的方式遍历沙箱，以：

- access data (e.g. state data, chain data, randomness).
- 访问数据（例如，状态数据、链数据、随机性）。
- perform side effects (e.g. write to state).
- 执行副作用（如写入状态）。
- invoke operations that are more efficient to implement in native space than in
  Wasm (e.g. cryptograhic primitives).
- 调用在本机空间中比在Wasm中实现更高效的操作（例如，加密原文）。

Calling a syscall triggers a context switch out, and back in when execution
returns to Wasm. The syscall gas merely covers this overhead. It does not price
the actual work performed by the syscall; that work is charged for separately,
and is specific to every syscall.

调用系统调用会触发上下文切换，并在执行返回Wasm时返回。系统调用gas仅覆盖此开销。
它不为系统调用执行的实际工作定价；这项工作是单独收费的，并且特定于每个系统调用。

**Pricing formula**

**计价模型**

We have quantified the average context switch overhead to be equivalent to 3500
execution units. At the rate of 4 gas per execution unit, we set the syscall gas
price to be 14000 gas.

我们量化了平均上下文切换开销，相当于3500个执行单元。
按照每个执行单元4 gas的速率，我们将系统调用气体价格设置为14000 gas。

```
p_syscall_gas := 14000
```

**Future considerations**

**未来考虑**

The cost of syscall-driven context switches varies depending on the syscall, its
parameters, the amount of work it performs (thus invalidating CPU caches and
instruction pipeline), and other factors. In the future, we may tabulate syscall
gas depending on the complexity of the syscall, its arguments, its impact on CPU
optimizations, and more.

系统调用驱动的上下文切换的成本取决于系统调用、其参数、执行的工作量（从而使CPU缓存和指令管道无效）以及其他因素。
将来，我们可能会根据系统调用的复杂性、参数、对CPU优化的影响等将系统调用gas制成表格。

### Extern gas 外部的gas费

Some syscalls cannot be resolved entirely within FVM space. Such syscalls need
to traverse the _extern_ (FVM <> Node) boundary to access client functionality.

某些系统调用不能完全在FVM空间内解析。此类系统调用需要遍历_外部_（FVM<>节点）边界才能访问客户端功能。

Depending on the languages of the client and the FVM implementation, traversing
this boundary may involve foreign-function interface (FFI) mechanics. That is
indeed the case with the reference client (Lotus) and the reference FVM
(ref-fvm). This is the most common scenario today given the client distribution
of the network.

根据客户端和FVM实现的语言，穿越该边界可能涉及外部函数接口（FFI）机制。
引用客户端（Lotus）和引用FVM（ref-FVM）的情况确实如此。
考虑到网络的客户端分布，这是当今最常见的场景。

The associated overheads are:

相关间接费用包括：

- data structure translation between representations, including potential
  serialization overheads.
- 表示之间的数据结构转换，包括潜在的序列化开销。
- context switch.
- 上下文切换。
- the memory overhead on both sides of the interface.
- 接口两侧的内存开销。
- the amortized opportunity cost of dedicating OS threads to handle FFI calls.
- 专用操作系统线程处理FFI调用的摊余机会成本。
- the amortized garbage collection or deallocation costs once the extern
  concludes.
- 一次性外部得出的摊销垃圾收集或释放成本。
- function lookup and dispatch.
- 函数查找和调度。

**Pricing formulae**

**计价模型**

Extern gas is priced at 1.5x syscall gas. It is equivalent to a flat 5250
execution units, and therefore it costs 21000 gas. Extern gas is additive to
syscall gas.

外部gas的价格是系统调用gas的1.5倍。它相当于一个平面5250执行单元，因此需要21000 gas费。
外部gas是系统调用gas的添加剂。

```
p_extern_gas := 21000
```

The following syscalls are extern-traversing syscalls, and therefore are subject
to extern gas:

以下系统调用是外部遍历系统调用，因此受外部gas约束：

- `rand::get_chain_randomness`.
- `rand::get_beacon_randomness`.
- `crypto::verify_consensus_fault`.
- `ipld::block_open`.

**Future considerations**

**未来考虑**

It may be possible to drop the extern traversal on randomness fetches, by
passing in a reference to a ring-buffer-like data structure (maintained by the
Node) on Machine construction. This data structure would hold 2 * finality worth
of VRF proofs & beacons, for the FVM kernel to index into directly, thus
eliminating the extern access.

通过在机器构造上传递对环形缓冲区类数据结构（由节点维护）的引用，可以在随机性获取时丢弃外部遍历。该数据结构将保存2 * 最终价值的VRF证明和信标，供FVM内核直接索引，从而消除外部访问。

- The estimated size is 405Kb + data structure overhead (225 bytes of data per
  epoch x 1800 epochs).
- 估计大小为405Kb+数据结构开销（每个历元225字节数据 x 1800个历元）。
- This future improvement is tracked under [filecoin-project/ref-fvm#525](https://github.com/filecoin-project/ref-fvm/issues/525)
  for the reference FVM implementation.
- 这种未来的改进在[filecoin-project/ref-fvm#525](https://github.com/filecoin-project/ref-fvm/issues/525)下进行跟踪
  用于参考FVM实现。

### IPLD state management gas IPLD状态管理的gas费

Currently, IPLD state block operations incur these fees:

目前，IPLD状态区块运营产生以下费用：

- Reads incur a fixed fee.
- 读取操作产生一个固定费用。
- Writes incur a variable fee, based on the number of bytes written.
- 写入操作会根据写入的字节数产生可变费用。

With the introduction of the FVM, the state management operations have evolved
from a two-method design to five syscalls:

随着FVM的引入，状态管理操作已从两种方法设计演变为五种系统调用：

- `ipld::block_open`
- `ipld::block_read`
- `ipld::block_write`
- `ipld::block_link`
- `ipld::block_stat`

For more details, refer to `ipld` namespace syscall catalogue in
[FIP-0030](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md#namespace-ipld).

有关详细信息，请参阅中的`ipld`命名空间系统调用目录
[FIP-0030](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md#namespace-ipld)。

**memcpy per byte gas charge**

**每字节内存拷贝gas注入**

We introduce a gas charge for memcpy'ing bytes across system boundaries
(`p_memcpy_gas_per_byte`). The fee is 500 milligas per byte, resulting from
statistics collected on average memcpy latencies, adapted to the Filecoin gas
sizing baseline (10gas per ns).

我们引入了跨系统边界存储字节的gas注入（`p_memcpy_gas_per_byte'）。费用为每字节5亿gas，
根据平均memcpy延迟收集的统计数据得出，适用于Filecoin gas大小基线（每ns 10gas）。

```
p_memcpy_gas_per_byte := 0.5
```

**Memory retention per byte gas charge**

**每字节内存保持率gas注入**

Some operations retain buffers in FVM Kernel and Machine space. We introduce a
gas charge to cover memory retention, modelled after a 1GB limit of memory usage
per transaction (worst case scenario, contemporary block validator hardware).

有些操作在FVM内核和机器空间中保留缓冲区。
我们引入了一个gas费用来覆盖内存保留，模拟每个事务的1GB内存使用限制（最坏情况下，当代块验证器硬件）。

```
p_memret_gas_per_byte := 10
```

**Pricing for `ipld::block_open`**

**计价`ipld::block_open`**

_Overhead:_ This syscall traverses the FVM <> Node boundary via an extern to
access the state blockstore. The bytes returned through the extern are copied
into FVM kernel buffers; thus there is an memcpy and alloc cost (the latter is
absorbed into the former). The kernel buffers are retained until the actor call
finishes.

_开销:_ 此系统调用通过外部访问状态块存储区穿越FVM<>节点边界。
通过extern返回的字节被复制到FVM内核缓冲区；因此，存在memcpy和alloc成本（后者被吸收到前者中）。
内核缓冲区将保留，直到参与者调用完成。

The pricing formula is:

计价模型是：

```
p_blockopen_base_gas := 114617   # extant fee
op_ipld_blockopen := p_syscall_gas
                     + p_extern_gas
                     + p_blockopen_base_gas
                     + (p_memcpy_gas_per_byte * block_length)
                     + (p_memret_gas_per_byte * block_length)
```

**Pricing for `ipld::block_read`**

**计价`ipld::block_read`**

_Overhead:_ This syscall performs an memcpy from the FVM kernel's buffers to
Wasm memory.

_开销:_ 此系统调用执行从FVM内核缓冲区到Wasm内存的memcpy。

The pricing formula is:

计价模型是：

```
op_ipld_blockread := p_syscall_gas + (p_memcpy_gas_per_byte * byte_length)
```

**Pricing for `ipld::block_write`**

**计价`ipld::block_write`**

_Overhead:_ This syscall performs an memcpy from Wasm memory into the FVM
kernel's buffers, with a corresponding alloc (absorbed into the memcpy cost).
The kernel buffers are retained until the actor call finishes.

_开销:_ 此系统调用执行从Wasm内存到FVM内核缓冲区的memcpy，并使用相应的alloc（吸收到memcpy成本中）。
内核缓冲区将保留，直到参与者调用完成。

The pricing formula is:

定价公式为：

```
op_ipld_blockwrite := p_syscall_gas
                      + (p_memcpy_gas_per_byte * block_length)
                      + (p_memret_gas_per_byte * block_length)
```

Future considerations:

未来考虑

- This call will likely become more expensive when user programmability is
  introduced as `ipld::block_write` will need to reason about data
  availability/reachability.
- 当引入用户可编程性时，此调用可能会变得更昂贵，因为`ipld::block_write`需要考虑数据可用性/可达性。

**Pricing for `ipld::block_link`**

**计价`ipld::block_link`**

_Overhead:_ This syscall incurs a hashing operation to compute the CID of the
block. It then copies it over to the buffered blockstore, where it is retained
until the Machine is flushed and the blocks are written to the Node's blockstore
via an extern. The extern is amortized across all the blocks written throughout
the Machine's lifetime, and the cost is included in the storage multiplier.

_开销:_ 此系统调用导致哈希操作以计算块的CID。
然后，它将其复制到缓冲区块存储中，在那里它被保留，直到机器被刷新，并且块通过外部写入节点的区块存储中。
extern在整个机器生命周期内写入的所有块中进行摊销，成本包含在存储乘数中。

The pricing formula for compute gas is:

计算gas的定价公式为：

```
p_blocklink_base_gas := 353640    # extant fee
p_storage_gas_multiplier := 1300  # extant fee
op_ipld_blocklink_compute := p_syscall_gas
                             + p_blocklink_base_gas
                             + (p_memcpy_gas_per_byte * block_length * 2)
op_ipld_blocklink_storage := p_storage_gas_multiplier * block_length
op_ipld_blocklink := op_ipld_blocklink_compute + op_ipld_blocklink_storage
```

Note: we don't charge a memory retention cost here because it's already factored
into the base operation gas (this operation is equivalent in overhead to the
pre-FVM `StorePut`).

注意：我们这里不收取内存保留成本，因为它已经被计入基本操作gas（此操作的开销相当于FVM之前的`StorePut`）。

**Pricing for `ipld::block_stat`**

**计价`ipld::block_stat`**

No relevant overhead.

没有相关的开销。

The pricing formula is:

定价公式为：

```
op_ipld_blockstat := p_syscall_gas
```

### Milligas precision and under/overflow handling Milligas精度和欠/溢出处理

The unit of internal gas accounting is milligas (1e3 precision, increments of
1/1000). When message execution completes, or gas is queried by an actor, the
FVM rounds the _gas used_ up, and the _gas available_ down, to the nearest whole
gas.

内部gas计量单位为毫气体（1e3精度，增量为1/1000）。
当消息执行完成或参与者查询gas时，FVM将_gas used_向上和_gas available_向下四舍五入到最近的整个gas。

Operations that have the potential of either underflowing or overflowing should
be performed using [saturation arithmetic](https://en.wikipedia.org/wiki/Saturation_arithmetic).
The valid range for any execution units operation, including total execution units
consumed, is [0, 2<sup>64</sup>-1]. 

应使用[饱和算术](https://en.wikipedia.org/wiki/Saturation_arithmetic)执行可能下溢或溢出的操作.
任何执行单元操作的有效范围（包括消耗的总执行单元）为[0,2<sup>64</sup>-1]。

The valid range for any gas operation, including gas available and total gas
used, is [0, (2<sup>63</sup>-1)/1000]. Internally, during message execution, gas
used may ephimerally turn negative as of Filecoin nv15 (gas refunds). Upon
concluding message execution, gas used is clamped to 0 on the lower bound, thus
effectively setting a floor of 0.

任何gas操作的有效范围，包括可用gas和使用的总gas，为[0, (2<sup>63</sup>-1)/1000]。
在内部，在消息执行期间，所使用的gas可能会在Filecoin nv15（gas退款）后立即变为负值。
在结束消息执行时，所使用的gas在下界上被钳制为0，从而有效地将下限设置为0。

## Design Rationale 设计原理

### Gas sizing baseline Gas分级基线

We use a baseline of 10 gas/nanosecond of wall-clock time, or 1 milligas/10
picosecond. This baseline follows from the need to sustain a 30s epoch by
validating and applying unique messages in a tipset, with a block count
following a Poisson distribution (win rate), with a 10B block gas limit,
assuming 6s of block propagation time.

我们使用10gas/纳秒的挂钟时间基线，或1毫gas/10皮秒。
该基线源于需要通过验证和应用tipset中的唯一消息来维持30秒的历元，
其中块计数遵循Poisson分布（获胜率），假设块传播时间为6s，块气体限制为10B。

### Benchmark methodology 基准方法

The parameters for this FIP were calculated through extensive benchmarking using
three profiling techniques to obtain a performance data from various vantage
points:

本FIP的参数是通过广泛的基准测试计算的，使用三种分析技术从不同的有利位置获得性能数据：

1. Fine-grained tracing of 2.5MM mainnet message applications.
1. 2.5MM主网消息应用程序的细粒度跟踪。
2. Aggregated execution statistics of 2.5MM mainnet messages.
2. 2.5MM主网消息的聚合执行统计。
3. Raw Wasmtime micro-benchmarks.
3. 原始Wasmtime微观基准。

The machines had the following specs: 1 x AMD EPYC 7402P, 24 cores @ 2.8 GHz, 2
x 240GiB SSD (boot), 2 x 480GiB SSD (storage), 64GiB DDR4 RAM, running Debian
4.19.208-1.

这些机器的规格如下：1 x AMD EPYC 7402P, 24 cores @ 2.8 GHz, 2
x 240GiB SSD (boot), 2 x 480GiB SSD (storage), 64GiB DDR4 RAM, running Debian
4.19.208-1.

The code assets for (1) and (2) are available at
[filecoin-project/ref-fvm#465](https://github.com/filecoin-project/ref-fvm/pull/465).

(1)和(2)的代码资产可从以下网站获得：
[filecoin-project/ref-fvm#465](https://github.com/filecoin-project/ref-fvm/pull/465).

The raw Wasmtime micro-benchmarker is available at
[filecoin-project/ref-fvm#502](https://github.com/filecoin-project/ref-fvm/pull/502).

原始Wasmtime微基准可从以下网站获得：
[filecoin-project/ref-fvm#502](https://github.com/filecoin-project/ref-fvm/pull/502).

Benchmarking overheads were measured and corrected for in the data.

在数据中测量并校正了基准管理费用。

## Backwards Compatibility 向后兼容

External-facing gas representation (gas limit, gas used, etc.) remains
unchanged.

外部gas表示（gas极限、gas使用等）保持不变。

However, the gas usage of most messages will increase upwards, so systems using
static gas limits should be adapted accordingly.

然而，大多数消息的gas使用量将向上增加，因此应相应地调整使用静态gas限制的系统。

Clients sending messages near the activation epoch of this FIP should estimate
gas under the new pricing model to guarantee inclusion.

在本FIP激活期附近发送消息的客户应根据新的定价模型估算gas，以确保包含在内。

- If the message is included pre-activation, the message will result in a higher
  overestimation burn.
- 如果该消息包含在预激活中，则该消息将导致更高的高估燃烧。
- If the message is included post-activation, the message will be executed
  normally.
- 如果激活后包含消息，则消息将正常执行。

## Test Cases 测试用例

Refer to test plan laid out in [FIP-0031].

参考[FIP-0031]中的测试计划。

## Security Considerations 安全考虑

Metering and pricing execution enables more precise gas accounting, which
translates into higher network security.

计量和定价执行能够实现更精确的天然气会计，这转化为更高的网络安全性。

Note that this FIP alone is insufficient to secure the network for
user-programmed actors; it is merely a stepping stone. A future FIP will
introduce further the final gas accounting model changes before enabling user
programmability.

注意，仅此FIP不足以保护用户编程参与者的网络；它只是一块垫脚石。
未来的FIP将在启用用户可编程性之前进一步引入最终gas核算模型变更。

## Incentive Considerations 激励因素

The gas utilization of all messages will increase, except for value transfers
which are unaffected.

除不受影响的价值转移外，所有消息的气体利用率将增加。

In order to quantify the magnitude of change, we have backtested this model
against 2.5MM mainnet messages.

为了量化变化的幅度，我们针对2.5MM主网消息对该模型进行了回溯测试。

- Annex A includes a detailed breakdown of net utilization increase per message
  type (relative to original gas).
- 附件A包括每种消息类型的净利用率增加的详细分类（相对于原始gas）。
- Annex B includes the final total gas per message type.
- 附录B包括每种消息类型的最终gas总量。

On average, most message types increase their utilization between 1.5-3x. The
gas performance of specific messages types is subject to various factors. The
reader is advised to study Annex A to understand the variability involved.

平均而言，大多数消息类型的利用率提高了1.5-3倍。
特定消息类型的gas性能受各种因素的影响。
建议读者研究附录A，以了解所涉及的可变性。

In our backtest, the four most important messages for storage provider
operations were impacted in this manner:

在我们的回溯测试中，存储提供程序操作的四条最重要的消息以这种方式受到影响：

- SubmitWindowPoSt (miner actor): gas usage increases an average of 3x, with a
  median of 2.79x, and a 90% percentile score of 4.47x. Some executions spiked
  as much as 15.24x. These are likely window PoSt submissions that recovered
  power, as the resulting work has a significant computational cost that was not
  previously accounted for.
- SubmitWindowPoSt（矿工参与者）：gas使用量平均增加3倍，中位数为2.79x，90%百分位分数为4.47x。
  一些执行峰值高达15.24倍。这些可能是恢复存力后的PoSt窗口提交，因为由此产生的工作具有之前未考虑的巨大计算成本。
- PreCommitSector (miner actor): gas usage increases an average of 3.12x, with a
  median of 2.49x, and a 90% percentile score of 5.09x. Some executions spiked
  as much as 9.69x.
- 预提交扇区（矿工与者）：gas使用量平均增加3.12倍，中位数为2.49倍，90%百分位得分为5.09倍。
  一些执行峰值高达9.69倍。
- ProveCommitSector (miner actor): gas usage increases an average of 1.25x, with
  an identical median, and a 90% percentile score of 1.31x. Some executions
  spiked as much as 3.02x.
- 证明提交扇区（矿工参与者）：gas使用量平均增加1.25倍，中位数相同，90%百分位得分为1.31倍。
  一些执行峰值高达3.02倍。
- PublishStorageDeals (market actor): gas usage increases an average of 1.80x,
  with a median of 1.84x, and a 90% percentile score of 2.03x. Some executions
  spiked as much as 2.17x.
- 发布存储交易（市场参与者）：gas使用量平均增长1.80倍，中位数为1.84倍，90%的百分位数得分为2.03倍。
  一些执行峰值高达2.17倍。

These two message types deserve special attention.

这两种消息类型值得特别注意。

- DeclareFaultsRecovered (miner actor): highest variability between executions
  and highest multiplier; on average, gas cost increases 8.67x, with some
  instances spiking as much as 33.3x in the backtest. All executions were within
  the block gas limit.
- 定义错误恢复（矿工参与者）：执行之间的最大可变性和最高乘数；
  平均而言，天然气成本增加了8.67倍，在某些情况下，在回溯测试中达到33.3倍。
  所有执行都在区块气体限制范围内。
- ExtendSectorExpiration (miner actor): infrequent; was already an expensive
  message. Some executions lead to gas usage beyoned the block limit, rendering
  them inviable (up to 2.3e10 gas). However, this is a batched message, so
  clients should size batches within gas limits.
- 延长扇区到期（矿工角色）：不常见；已经是一个昂贵的消息。
  一些执行导致gas使用超过区块限制，使其不可高攀（高达2.3e10 gas)。
  但是，这是一个批处理消息，因此客户端应在gas限制内调整批处理的大小。

On average, we observe a global increase of 1.95x in gas utilization, if the
current chain activity were to remain unchanged. However, the HyperDrive upgrade
introduced power onboarding batching mechanisms in [FIP-0008] and [FIP-0013]
that have stayed underutilized. We expect these to kick in an adaptive response
to lower usage and increase chain bandwidth again.

平均而言，如果当前链活动保持不变，我们观察到全局gas利用率增加了1.95倍。
然而，HyperDrive升级在[FIP-0008]和[FIP-0013]中引入了存力加载批处理机制，这些机制一直未得到充分利用。
我们预计，这些将启动对低使用率的自适应响应，并再次增加链带宽。

At last, we expect upwards pressure on the base fee, due to two factors:

最后，由于两个因素，我们预计基本费用将面临上涨压力：

- higher chain bandwidth utilization.
- 链带宽利用率更高。
- message executions more frequently surpassing the base fee target (50% of
  block gas limit, i.e. 5B).
- 消息执行频率更高，超过基本费用目标（区块gas限额的50%，即5B）。

## Product Considerations 产品考虑

1. Messages get more expensive. See Incentive Considerations section for a
   synthesis, and Annex A and Annex B for raw data.
1. 消息变得越来越昂贵。综合信息见激励因素一节，原始数据见附件A和附件B。
2. Potential for higher community motivation to optimize built-in actors logic,
   as doing so directly translates into cost reductions.
2. 更高社区动机优化内置参与者逻辑的潜力，因为这样做直接转化为成本降低。
3. Messages estimated pre-FVM will likely not be executed post-FVM. See
   Backwards Compatibility section for ways to mitigate.
3. 在FVM之前估计的消息可能不会在FVM之后执行。请参阅向后兼容性一节，了解缓解方法。
4. Expect higher usage of aggregated forms of PreCommitSector and
   ProveCommitSector, as they become more cost-effective under the new gas
   scenario.
4. 由于在新的gas情景下，聚合形式的预提交扇区和证明提交扇区的成本效益更高，因此预计会有更高的使用率。
5. Clients may need to re-evaluate thresholds for batched operations, so as to
   avoid generating inviable messages due to exceeding the block gas limit.
5. 客户端可能需要重新评估批处理操作的阈值，以避免由于超出块gas限制而生成不可更改的消息。

## Implementation 实现

For clients relying on the reference FVM implementation
([filecoin-project/ref-fvm](https://github.com/filecoin-project/ref-fvm)), the
implementation of this FIP will be transparent, as it is self-contained inside
the FVM.

对于依赖参考FVM实施的客户
（[filecoin项目/ref FVM](https://github.com/filecoin-project/ref-fvm))，
本FIP的实施将是透明的，因为它在FVM中是独立的。

## Assets 资产

Raw data assets, as well as synthesised backtest results, can be found at
[raulk/fil-fip-0032-data/](https://github.com/raulk/fil-fip-0032-data/).

原始数据资产以及综合回溯测试结果可在[raulk/fil-fip-0032-data/](https://github.com/raulk/fil-fip-0032-data/)中找到.

## Annex A: Table of gas utilization increase per message type 附录A：每种消息类型的gas利用率增加表

|                        |        | gas\_utilization\_increase |             |                |             |             |             |             |             |             |             |             |             |
| ---------------------- | ------ | -------------------------- | ----------- | -------------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
|                        |        | count                      | mean        | std            | min         | 50%         | 60%         | 70%         | 80%         | 90%         | 95%         | 99%         | max         |
| code                   | method |                            |             |                |             |             |             |             |             |             |             |             |             |
| fil/7/account          | 0      | 45871                      | 1.00597202  | 0.02439399865  | 1           | 1           | 1           | 1           | 1           | 1           | 1.105295017 | 1.105620324 | 1.116646265 |
| fil/7/init             | 2      | 6                          | 1.492804871 | 0.004038268814 | 1.48830639  | 1.491762603 | 1.493232453 | 1.494468613 | 1.495704774 | 1.497400912 | 1.498248981 | 1.498927436 | 1.49909705  |
| fil/7/multisig         | 0      | 111                        | 1           | 0              | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           |
| fil/7/multisig         | 2      | 1837                       | 1.525822767 | 0.1538518354   | 1.475017101 | 1.49094471  | 1.490971227 | 1.491292227 | 1.492171101 | 1.503233732 | 1.628553559 | 2.395083768 | 2.559610339 |
| fil/7/multisig         | 3      | 467                        | 2.092939348 | 0.5178361861   | 1.419330832 | 2.004487358 | 2.04664811  | 2.402342227 | 2.681836119 | 2.943685447 | 3.024047067 | 3.13563372  | 3.282831103 |
| fil/7/multisig         | 4      | 1                          | 1.568009198 |                | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 | 1.568009198 |
| fil/7/paymentchannel   | 0      | 1                          | 1           |                | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           |
| fil/7/storagemarket    | 2      | 2586                       | 1.767349037 | 0.0179153187   | 1.705184746 | 1.764432273 | 1.771243992 | 1.777229832 | 1.781349799 | 1.795999314 | 1.79848267  | 1.799319127 | 1.80223568  |
| fil/7/storagemarket    | 3      | 1                          | 1.947764405 |                | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 | 1.947764405 |
| fil/7/storagemarket    | 4      | 5265                       | 1.800550738 | 0.1892337028   | 1.421170992 | 1.836807517 | 1.846956112 | 1.8859387   | 1.965049598 | 2.032417936 | 2.058474275 | 2.098231267 | 2.166245941 |
| fil/7/storageminer     | 0      | 22                         | 1           | 0              | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           | 1           |
| fil/7/storageminer     | 3      | 33                         | 1.581185587 | 0.01756393579  | 1.55043385  | 1.576662766 | 1.587495788 | 1.588772509 | 1.594206699 | 1.609600058 | 1.60997889  | 1.615531646 | 1.618144708 |
| fil/7/storageminer     | 4      | 56                         | 1.437003934 | 0.01053285152  | 1.412468752 | 1.440333482 | 1.440578616 | 1.443058913 | 1.445157793 | 1.448293134 | 1.448357005 | 1.457685961 | 1.458410689 |
| fil/7/storageminer     | 5      | 217040                     | 3.062597507 | 1.284676077    | 1.084260869 | 2.794653576 | 3.170948509 | 3.551019009 | 3.954638061 | 4.47275546  | 5.219714821 | 7.854538141 | 15.24397403 |
| fil/7/storageminer     | 6      | 1185718                    | 3.122825812 | 1.479978666    | 1           | 2.489184964 | 2.66072048  | 3.119869193 | 4.095753137 | 5.087751869 | 6.484390167 | 8.728160893 | 9.688604136 |
| fil/7/storageminer     | 7      | 1053752                    | 1.254059457 | 0.04338576394  | 1           | 1.251206444 | 1.262072669 | 1.274505314 | 1.289751641 | 1.310614092 | 1.326975995 | 1.363865829 | 3.020495782 |
| fil/7/storageminer     | 8      | 30                         | 2.561482948 | 0.8795361903   | 1.659726975 | 2.193520195 | 2.353174497 | 2.878519824 | 3.252169093 | 4.1322215   | 4.158195316 | 4.196685589 | 4.208294346 |
| fil/7/storageminer     | 9      | 104                        | 2.773999785 | 0.7078355684   | 1.924300767 | 2.470266911 | 2.830244664 | 3.240411111 | 3.318940856 | 3.485240501 | 3.720053128 | 5.585537848 | 5.639734092 |
| fil/7/storageminer     | 11     | 1615                       | 8.670368309 | 8.066618708    | 1.711216051 | 3.91247307  | 4.886204415 | 10.96614305 | 17.87576882 | 23.55604103 | 24.2148853  | 27.84382546 | 33.30957868 |
| fil/7/storageminer     | 16     | 1310                       | 2.162632701 | 0.05734323166  | 1.582717534 | 2.161534147 | 2.165197244 | 2.169072728 | 2.170384968 | 2.171488192 | 2.173423405 | 2.355466711 | 2.394978809 |
| fil/7/storageminer     | 18     | 18                         | 1.445781244 | 0.01207120396  | 1.419464401 | 1.447819167 | 1.454712965 | 1.454744278 | 1.454813087 | 1.454813087 | 1.456660407 | 1.465034926 | 1.467128555 |
| fil/7/storageminer     | 22     | 2                          | 1.578751057 | 0.000634365616 | 1.578302493 | 1.578751057 | 1.57884077  | 1.578930483 | 1.579020196 | 1.579109909 | 1.579154765 | 1.57919065  | 1.579199622 |
| fil/7/storageminer     | 23     | 37                         | 1.44732633  | 0.006346257808 | 1.429727021 | 1.44896509  | 1.449177713 | 1.449769555 | 1.451859339 | 1.455148039 | 1.455649445 | 1.456810884 | 1.457432851 |
| fil/7/storageminer     | 25     | 7482                       | 2.846175784 | 0.8397517462   | 1           | 2.509378787 | 2.641833976 | 2.899623107 | 3.117726086 | 3.476581572 | 4.980988959 | 6.008160965 | 8.725638329 |
| fil/7/storageminer     | 26     | 5810                       | 1.914048048 | 0.428505585    | 1           | 1.909877692 | 2.135478681 | 2.278302186 | 2.332170825 | 2.468622524 | 2.556291786 | 2.719278562 | 3.132584833 |
| fil/7/storageminer     | 27     | 81                         | 1.736822205 | 0.1415775839   | 1.65961248  | 1.698638407 | 1.702767421 | 1.706075911 | 1.719693881 | 1.825980876 | 1.926526938 | 2.186172291 | 2.823086016 |
| fil/7/storagepower     | 2      | 34                         | 1.479426147 | 0.007010768301 | 1.466525427 | 1.479850457 | 1.481154027 | 1.482644207 | 1.485159657 | 1.487302302 | 1.490627682 | 1.495758666 | 1.496217996 |
| fil/7/verifiedregistry | 4      | 11                         | 1.804133732 | 0.03922232077  | 1.692439068 | 1.813042981 | 1.825646137 | 1.825646137 | 1.82587245  | 1.82587245  | 1.828249976 | 1.830151998 | 1.830627503 |

## Annex B: Table of total final gas used per message type 附录B：每种消息类型使用的最终gas总量表

|                        |        | final\_total\_gas |             |             |            |             |             |             |             |             |             |             |             |
| ---------------------- | ------ | ----------------- | ----------- | ----------- | ---------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
|                        |        | count             | mean        | std         | min        | 50%         | 60%         | 70%         | 80%         | 90%         | 95%         | 99%         | max         |
| code                   | method |                   |             |             |            |             |             |             |             |             |             |             |             |
| fil/7/account          | 0      | 45871             | 579530.8664 | 429997.1536 | 278696     | 491868      | 491868      | 491868      | 493168      | 495768      | 2326134.5   | 2332634.5   | 2415834.5   |
| fil/7/init             | 2      | 6                 | 18638578.58 | 729881.099  | 17420380   | 18954972.25 | 19060170.5  | 19129952.75 | 19199735    | 19208524.75 | 19212919.63 | 19216435.53 | 19217314.5  |
| fil/7/multisig         | 0      | 111               | 443475.2072 | 23048.51686 | 415168     | 441168      | 441168      | 442468      | 443768      | 493168      | 493168      | 493168      | 494468      |
| fil/7/multisig         | 2      | 1837              | 5837582.178 | 2411162.29  | 3558017.5  | 5629077     | 5629866     | 5634561     | 5638611.5   | 5643135.5   | 5682706.9   | 18325446.76 | 30094244    |
| fil/7/multisig         | 3      | 467               | 13862511.4  | 12066758.5  | 991698.5   | 8122883     | 12734041    | 14655463.8  | 25220149    | 35496028    | 41887381.8  | 46613173.3  | 50468854.5  |
| fil/7/multisig         | 4      | 1                 | 1997154.5   |             | 1997154.5  | 1997154.5   | 1997154.5   | 1997154.5   | 1997154.5   | 1997154.5   | 1997154.5   | 1997154.5   | 1997154.5   |
| fil/7/paymentchannel   | 0      | 1                 | 486668      |             | 486668     | 486668      | 486668      | 486668      | 486668      | 486668      | 486668      | 486668      | 486668      |
| fil/7/storagemarket    | 2      | 2586              | 19653863.1  | 479936.3183 | 18433677   | 19536613.5  | 19608916    | 19958224    | 20155859.5  | 20396658.5  | 20397347    | 20653887    | 21137437.5  |
| fil/7/storagemarket    | 3      | 1                 | 22612491    |             | 22612491   | 22612491    | 22612491    | 22612491    | 22612491    | 22612491    | 22612491    | 22612491    | 22612491    |
| fil/7/storagemarket    | 4      | 5265              | 431978034.6 | 476844103.7 | 65922953   | 288570649   | 339742341.4 | 404653981.3 | 843094881.7 | 888783311.4 | 929103138.6 | 2511917888  | 2630317760  |
| fil/7/storageminer     | 0      | 22                | 440636.1818 | 33021.16762 | 417768     | 419718      | 420368      | 454038      | 488228      | 493168      | 495638      | 495768      | 495768      |
| fil/7/storageminer     | 3      | 33                | 3041236.333 | 156295.4373 | 2911417.5  | 2984946     | 3001525.2   | 3021368.7   | 3051762.3   | 3367924.5   | 3369224.5   | 3442895.7   | 3477564.5   |
| fil/7/storageminer     | 4      | 56                | 2523479.821 | 48041.34968 | 2410819.5  | 2528831.5   | 2534763.5   | 2556110.5   | 2559944.5   | 2580522.5   | 2580616.5   | 2612800.1   | 2621395.5   |
| fil/7/storageminer     | 5      | 217040            | 50567157.53 | 129615886.8 | 1277248    | 29928389.5  | 38593441    | 49359013.65 | 64280830.6  | 87800703.9  | 112949545.8 | 278422165.1 | 7276968168  |
| fil/7/storageminer     | 6      | 1185718           | 57884701.53 | 64060736.44 | 335919     | 33593786.5  | 37303189    | 46857982.6  | 72455034.6  | 111511861.5 | 179449550.5 | 364239625   | 476861537.5 |
| fil/7/storageminer     | 7      | 1053752           | 65322144.86 | 10709175.02 | 2759119    | 62072363    | 64138457.6  | 66994027.95 | 71603156.3  | 80459562.8  | 89151232.75 | 102002577.8 | 124322877.5 |
| fil/7/storageminer     | 8      | 30                | 6142153792  | 8813636083  | 31233192   | 57607162.25 | 117330392.8 | 15159834743 | 16307686048 | 19076238418 | 21880538294 | 22785383969 | 23059819584 |
| fil/7/storageminer     | 9      | 104               | 146564002.1 | 186092472.6 | 27323860.5 | 82113225.25 | 115345332.9 | 146638929.4 | 186883609.9 | 294737331.4 | 336147597.8 | 1108397934  | 1497431427  |
| fil/7/storageminer     | 11     | 1615              | 223766675.3 | 417849908   | 10456724.5 | 49380451.5  | 75358129.9  | 157136408.2 | 407604727.1 | 536195865.4 | 909680332.4 | 2094319347  | 4050112282  |
| fil/7/storageminer     | 16     | 1310              | 20528854.52 | 2019509.742 | 2314035    | 20569943.25 | 20687486.7  | 20908518.25 | 21065727.2  | 21164879.5  | 21244818.55 | 25779334.27 | 26217311    |
| fil/7/storageminer     | 18     | 18                | 2488603.722 | 57093.71661 | 2374243    | 2502224.5   | 2531594     | 2533234.7   | 2533605     | 2533605     | 2540255.25  | 2570403.05  | 2577940     |
| fil/7/storageminer     | 22     | 2                 | 1942800     | 780.6458864 | 1942248    | 1942800     | 1942910.4   | 1943020.8   | 1943131.2   | 1943241.6   | 1943296.8   | 1943340.96  | 1943352     |
| fil/7/storageminer     | 23     | 37                | 2466446.095 | 39663.66082 | 2364863.5  | 2457840.5   | 2471928.1   | 2478669.7   | 2495597.9   | 2528554.9   | 2534383.5   | 2535670.54  | 2536394.5   |
| fil/7/storageminer     | 25     | 7482              | 217542474.4 | 259599842.7 | 341119     | 137725560.5 | 183224106.5 | 230833197.3 | 282139260.3 | 566387010.5 | 868583081.1 | 1192473146  | 1276876882  |
| fil/7/storageminer     | 26     | 5810              | 842389295.1 | 707401414.1 | 30303519   | 525095349.8 | 790506876.3 | 1021712113  | 1521512231  | 1685691298  | 2230385687  | 3368285319  | 4734149631  |
| fil/7/storageminer     | 27     | 81                | 137359349.3 | 18291408.57 | 124208174  | 133304884.5 | 134924673   | 136404981.5 | 139511612   | 150077218.5 | 158469749.5 | 208944161.6 | 263801274   |
| fil/7/storagepower     | 2      | 34                | 41528419.24 | 605565.076  | 40013457   | 41455588.5  | 41645275.8  | 41789855.7  | 42007558.7  | 42477707.6  | 42543546    | 42586868.5  | 42586868.5  |
| fil/7/verifiedregistry | 4      | 11                | 13307839.5  | 787667.1698 | 12461938   | 12928372.5  | 12929241    | 14163140    | 14163140    | 14190283    | 14383126    | 14537400.4  | 14575969    |

## Acknowledgements 致谢

Credits to:

- [@magik6k](https://github.com/magik6k) for inputs throughout the design process.
- [@arajasek](https://github.com/arajasek) for the initial implementation of FIP-0032 in ref-fvm.

## Copyright 版权

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[wasm-instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html
[FIP-0008]: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0008.md
[FIP-0013]: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md
[FIP-0030]: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md
[FIP-0031]: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0031.md
