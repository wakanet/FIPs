---
fip: "0010"
title: Off-Chain Window PoSt Verification
author: Steven (@stebalien), Alex (@anorth), et al.
discussions-to: https://github.com/filecoin-project/FIPs/issues/42
status: Final
review-period-end: 2021-01-21
type: Technical
category: Core
created: 2021-01-07
spec-sections: 
   - TBD
---

## Simple Summary 简述

Optimistically accept Window PoSt proofs on-chain without verification, allowing them to be disputed
later by off-chain verifiers.

乐观地接受链上的窗口后证明，而无需验证，允许链外验证人员稍后对其进行质疑。

## Abstract

When a miner proves continued storage of data to the chain (`SubmitWindowedPoSt`), optimistically
accept and record the proof on-chain instead of verifying it. After the chain has accepted the
proof, a third party may dispute it by invoking `DisputeWindowedPoSt`. A successful dispute marks
the incorrectly proven sectors as faulty, removes the associated power (until proven again in a
subsequent Window PoSt), and fines the miner proportional to the expected block reward received from
the incorrectly proved sectors.

## Change Motivation

Window PoSt messages are necessary for ongoing maintenance of storage power. Verifying the submitted
proofs is expensive and when the gas base fee rises (due to congestion) these messages become
expensive. In bad cases, for small miners with mostly-empty partitions, this cost can exceed their
expected reward from maintaining power. We need to ensure that these messages are cheap for miners,
even when specifying a very high gas fee cap.

While, as-of [FIP 0009](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0009.md),
`SubmitWindowedPoSt` messages are free, this was an imperfect stop-gap measure and it does not
reduce the load on the network itself.  The proposed change will, remove nearly all of the burden of
checking `SubmitWindowedPoSt` proofs from the chain (~13% of network bandwidth).

## Specification

We propose two key changes:

1. New Window PoSts, except those that restore faulty sectors, are optimistically accepted, assumed
   correct, and recorded in the state-tree for one proving period.
2. Optimistically accepted window posts can be disputed until `WPoStProofDisputeWindow` epochs after
   the challenge window in which they were submitted closes. When a dispute successfully refutes an
   optimistically accepted Window PoSt, the miner is fined one IPF per active sector in the
   partition (at the moment when the proof was submitted) plus a flat fee of 20FIL, all incorrectly proved sectors are marked
   faulty, and the disputer (address that submitted the dispute) is rewarded a fixed
   `DipsuteReward`.

Parameters:

* `WPoStProofDisputeWindow`: 1800 (2x finality).
* `IPF`: 5.51BR (IPF = Invalid Proof Fee, BR = Expected Block Reward per sector in 24h).
* `DisputeReward`: 4FIL. The reward should cover the expected gas cost for sending a dispute message.
* `FlatFee`: 20FIL. The flat feet sets a high minimum penalty for sending an invalid proof independently of the partition size and status (e.g., number of already faulted sectors).

In the following sections, we describe the changes to the Miner actor necessary to implement this FIP.

### Change: Deadline State

To accommodate this change, the deadline state is extended as follows:

```go
type Deadline struct {
    ...

	// AMT of optimistically accepted WindowPoSt proofs, submitted during
	// the current challenge window. At the end of the challenge window,
	// this AMT will be moved to PoStSubmissionsSnapshot. WindowPoSt proofs
	// verified on-chain do not appear in this AMT.
	OptimisticPoStSubmissions cid.Cid // AMT[]WindowedPoSt

	// Snapshot of partition state at the end of the previous challenge
	// window for this deadline.
	PartitionsSnapshot cid.Cid // AMT[]Partition

	// Snapshot of the proofs submitted by the end of the previous challenge
	// window for this deadline.
	//
	// These proofs may be disputed via DisputeWindowedPoSt. Successfully
	// disputed window PoSts are removed from the snapshot.
	OptimisticPoStSubmissionsSnapshot cid.Cid  // AMT[]WindowedPoSt
}

type WindowedPoSt struct {
	// Partitions proved by this WindowedPoSt.
	Partitions bitfield.BitField
	// Array of proofs, one per distinct registered proof type present in
	// the sectors being proven. In the usual case of a single proof type,
	// this array will always have a single element (independent of number
	// of partitions).
	Proofs []proof.PoStProof
}
```

### Change: SubmitWindowedPoSt

Proofs that do not recover faulty power are not checked on-chain. Instead,

* The proof and a bitfield of the affected partitions are recorded in an AMT of "optimistic" proofs
  (`Deadline.OptimisticPoStSubmissions`).
* The proof is not verified.
* The proof may be "disputed" by anyone for 1800 epochs (2x finality) after the challenge window
  ends.

As usual, skipped sectors are marked as faulty and "unproven" sectors are marked as "proven".

Notes:

* If a proof _recovers_ faulty power, it is checked on-chain immediately to prevent "recovering"
  faulty sectors with invalid proofs.
* "Unproven" sectors (sectors newly added to a partition but never proven with a window PoSt) do not
  force an on-chain proof verification. This is intentional.

### Change: Deadline End

As usual, partitions without proofs are marked as faulty, "new" (unproven) sectors are activated if
proofs for them are present, etc.

Immediately after processing all proof related logic, and before terminating any sectors, the
deadline's partitions array and optimistically accepted posts are "snapshotted" and stored in the
deadline. Specifically:

1. `Deadline.Partitions` is saved to `Deadline.PartitionsSnapshot`.
2. `Deadline.OptimisticPoStSubmissions` is saved to `Deadline.OptimisticPoStSubmissionsSnapshot`.

Future disputes will use these snapshots.

We then proceed to terminate sectors as usual.

### Change: TerminateSectors

Sector termination is no longer be allowed in currently-being-proved deadline, or the following
deadline. This brings sector termination in-line with partition compaction and sector assignment and
ensures that the partition snapshot taken at the end of the challenge window accurately reflects the
state that was proven.

### Change: CompactPartitions

As `CompactPartitions` can move sectors between partitions, it makes it difficult mark these sectors
as faulty when disputed. While the disputer could submit bitfields describing the new locations of
the sectors in question, this dispute could be defeated by repeatedly re-compacting.

To simplify the prevent this attack and simplify the disputer's job, partition compaction is
prohibited for `WPoStProofDisputeWindow` epochs after the deadline's last challenge window. This
leaves a window of 960 epochs (8 hours) for partition compaction.

Proofs may not be disputed after this time has passed.

### New: DisputeWindowedPoSt

`DisputeWindowedPoSt` selects a proof from a specific deadline's previous challenge window (`Deadline.OptimisticPoStSubmissionsSnapshot`) and attempts to disprove it.

0. The epoch and deadline are checked to make sure that the window for disputing proofs is still
   open (`WPoStProofDisputeWindow`).
1. The proof is verified against the partition state snapshot (`PartitionsSnapshot`) from the end of
   the last challenge window. This represents the state that was (supposedly) proven.
2. If the proof is valid, this method aborts with no change (the proof was _not_
   disproven). Otherwise, we continue.
3. All sectors that were active in the partition snapshot are declared as faulty (sectors terminated
   since the last challenge window are skipped).
4. The target miner loses power based on the sectors marked faulty in step 3.
5. The disputed proof is removed from `Deadline.OptimisticPoStSubmissionsSnapshot` so it can't be
   disputed twice.
6. The target miner is fined IPF based on the power was incorrectly proven (taken from
   the partitions snapshot) plus the flat fee and the dispute reward. This fine will also cover sectors that have since been terminated. This
   fine is taken from the miner's "vesting rewards" in the following order.
    1. The IPF portion of the fine (variable depending on the amount of incorrectly proven power) is
       burnt. If insufficient funds are available in the vesting rewards, the remainder is added to
       the miner's fee debt to be burnt if and when the fee debt is repaid.
	2. The `DisputeReward` portion of the fine (fixed) is sent to the address that submitted the
       dispute. If insufficient funds are available in the vesting rewards, the remainder of the
       reward is added to the miner's fee debt _and not_ sent to the address that submitted the
       dispute.

## Design Rationale

This section discusses the design and its trade-offs, alternative designs, and future work.

### Design Decisions

While drafting and subsequently implementing this proposal, we made several non-obvious design
decisions, detailed in this section.

#### On-Chain Proof Storage

Instead of storing proofs on-chain, along with all the relevant information needed to verify these
proofs, this data could have been hashed and the hash could have been stored.

This would have reduced on-chain storage used by submitted Window PoSt messages. However, it would
have complicated the process of disputing window posts as the disputing party would need to:

1. Find the message that submitted the proof.
2. Include the faulty proof, and all other necessary information, in the dispute message.

#### State Snapshots

This FIP proposes taking a snapshot of all partitions at the time a proof was submitted. An
alternative would have been to record a bitfield of the exact sectors that were proven.

This alternative has the benefit of being easier to reason about and prove correct.

However, taking a snapshot of partitions is effectively a free operation (the partitions already
exist in state so no additional writes are required) while saving bitfields of proven sectors would
require storing additional state.

#### On-chain Verification of Window PoSts Restoring Power

When a sector is marked "faulty", it is set to terminate within 14 days unless it is subsequently
recovered. If Window PoSts recovering power were not verified on-chain, a miner would be able to
indefinitely extend a faulty sector by repeatedly restoring it.

Additionally, the on-chain verification requirement ensures that power cannot be recovered faster
than it can be disputed. That is, it takes the same amount of chain bandwidth to recover power as it
does to dispute said power.

### Alternatives

Two alternative designs were considered:

* [Fast track for Window PoSt](https://github.com/filecoin-project/FIPs/issues/24)
* Batch verification of Window PoSt proofs.

#### Fast Track

The general idea here is to create a reserved, "fast-lane" for Window PoSt messages. This would
ensure that Window PoSts can be included by the chain, even when chain utilization is high.

Unfortunately, this would not actually free up any chain bandwidth as Window PoSt messages would
still need to be verified on-chain.

Either way, these two proposals are not mutually exclusive. A fast lane for necessary network
operations could still be beneficial to the network, even if this proposal is implemented.

#### Batch Verification

Window PoSt proofs could be "batch verified" the same way ProveCommit proofs are. That is,
SubmitWindowedPoSt could be broken up into the following steps:

1. The proof is processed and all the information needed to verify the proof is loaded into memory.
2. This state is submitted to the runtime for asynchronous and parallel processing.
3. After processing all messages in the block, the results of all successfully asynchronously
   verified proofs are sent back to the miner actors for further processing.

Unfortunately:

1. For moderately sized miners, about half of the time on-chain is spent loading information for
   proof verification (step 1). This step would still need to be done sequentially.
2. The speedup would be a constant factor (based on the expected parallelism available on all
   miners) and we'd eventually outgrow this constant speedup as storage is added to the network.

### Future Work

This proposal aims to be a step in the right direction, not a complete solution.

#### Incentive Improvements

One issue with the current proposal is the relative lack of incentives to check for Window PoSts,
given their relative rarity (see the Incentives section).

At the moment, the cost of simply verifying all Window PoSts is so low that many parties will do
this altruistically for the good of the network. Or, more pessimistically, they'll do it to ensure
their vesting Filecoin rewards aren't devalued (again, see the Incentives section). However, as
storage is added to the network, this cost may increase over time and additional incentives may be
required.

One potential solution is to (implicitly) elect verifiers (proportionally to the miners power) some
number of epochs after a proof has been accepted on-chain. If an invalid proof is not disputed
within some period, these verifiers would be fined if the proof is disputed after this period.

#### Optimizations

The current proposed implementation of off-chain Window PoSt verification has room for
improvement. Currently, to simplify the change, it loads every partition being proven in
`SubmitWindowedPoSt`, even if the proof is not being verified on-chain. It does this to determine if
the proof is restoring power (which would require on-chain verification).

In the future, additional state could be tracked in the `Deadline` object to avoid loading
partitions when "optimistically" accepting a proof.

See:
[@filecoin-project/specs-actors#1352](https://github.com/filecoin-project/specs-actors/issues/1352)

## Backwards Compatibility

As this is a consensus breaking change, all Filecoin implementers must implement the changes to the
Filecoin actors. Specifically, the Miner actor has changed as described in the "specification"
section.

In terms of user impact:

* `CompactPartitions` is now forbidden during the `WPoStDisputeWindow` (1800 epochs following a
  challenge window). Given that this is a very rarely invoked maintenance method, this should not
  impact any users.
* `TerminateSectors` is now forbidden for the deadline currently being challenged, and the deadline
  to be challenged next. As this is a rare, manually invoked method, this shouldn't significantly
  impact miners.
* Three fields are being added to the end of the `Deadline` object (described in the specification
  section). State inspection tools will need to accommodate this.
* The parameters for `SubmitWindowedPoSt` have not changed and the logic for generating and
  submitting proofs remains the same.

## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus
changes. Other FIPs can choose to include links to test cases if applicable.--> Test cases for an
implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to
include links to test cases if applicable.

TODO

## Security Considerations

The following security considerations are addressed as incentives considerations:
* If nobody disputes incorrect proofs, there is no incentive to reliably store data.
* If dispute proofs cannot make it on-chain, there is no incentive to reliably store data.

This section addresses:

* Chain state size attacks.
* Proof disputability attacks.
* Power & sector extension attacks.
* Vesting funds extraction.
* Proof spam attacks.

### Chain State Size

This change should not appreciably increase the size of chain state. The stored proofs are small
(190 bytes) only one proof may be submitted per partition with live sectors.

This will lead to approximately 2.5MiB (plus overhead) per EiB of storage on chain (13164
partitions), assuming densely packed partitions.

Additionally, the cost of storing the partition snapshots should be free (or close enough) assuming
nodes store the full chain history going at least one proving period (24 hours) into the past.

### All Invalid Proofs Are Disputable

The chain will not accept proofs that cannot be disputed.

1. Dispute messages are always a constant size, regardless of the size of the proof being disputed.
2. Proofs are still limited to proving 4 partitions at a time. The amount of work required by any
   given dispute is bounded by the maximum number of sectors/partitions that could have been proven.

However, we can not guarantee that all invalid proofs are disputed. It can happen that a dispute
message arrives too late on chain: for example, this happens when the messege is included after
`WPoStProofDisputeWindow` epochs after the challenge window closes, or just before this deadline and
a short fork rewrites few epochs and deletes the message from the chain.

To avoid this, disputes should be submitted promptly, ideally a full finality (900 epochs) before
the end of the dispute window.

### Power & Sector Extension

Given that proofs are accepted optimistically, an attacker can keep a faulty sector "active" by
submitting invalid window PoSts.

Currently, a fully corrupted or missing sector will be discovered after at most 24 hours (one
proving period) and the power will be remove from chain 900 (finality) epochs after that.

After this change, an invalid proof can extend this period until 900 epochs after a dispute for the
invalid proof is included on-chain.

However:

1. If these posts are successfully disputed, the attacker will lose more than 5.51 times the expected block
   reward per-day for the incorrectly proven sectors.
2. Restoring faulty sectors requires an on-chain proof. This prevents attackers from extending
   faulty sectors indefinitely by repeatedly submitting faulty proofs.

5.51 was chosen as it gives negative profit for miners as long as invalid proofs are caught within
12 hours with probability ≥ 1/3.

### Vesting Fund Extraction

`DisputeWindowedPost` takes fees from the "vesting funds". While it burns the fine (which should
generally be the majority of the fees), it distributes a small reward to the address that submitted
the dispute message.

This could allow a miner to take an advance on "vesting" funds by intentionally submitting invalid
proofs, and disputing their own proofs. However:

1. This attack is rate-limited. It can only be used to extract one reward per active partition every
   two days.
2. Another miner may dispute the proof first and steal the reward. This is especially true because
   the attack is slow so other miners will have a chance to notice it happening.
3. The attack is expensive. The minimum fine is 20FIL, and the reward is 4FIL. This means an
   attacker would lose at least 5/6th of their vesting FIL.

### Proof Spam Attack

An attacker may attempt to spam the network with invalid proofs. The most likely avenue for this
attack is to:

1. Register a new miner.
2. Seal one sector.
3. Start proving the sector with bad proofs.

Given that this miner will have less power than necessary to mine blocks, the "vesting rewards" will
be empty. Therefore, disputes will pay out no rewards.

Most rational miners will simply ignore these proofs as the target miner cannot pay out a
reward. However, for the good of the network, some parties may wish to dispute any/all invalid
proofs.

Luckily, even though the miner in question will offer no _pay out_ rewards to the party that submitted the
dispute, they still need to pay the penalty and rewards eventually or abandon the miner. That is, unpaid penalties and rewards are
added to the miner's debt and the miner will be unable to declare their sectors "recovered" until
they pay their debts.

Given that the minimum fine, even for one sector, is 20FIL, and the reward is 4FIL, the attacker
will need to pay over 24FIL to "revive" their sector. So the cost of this attack is either ~24FIL or
the cost of creating a new miner and sealing a new sector.

## Incentive Considerations

In this section, we discuss the three actions being incentivized:

1. Submitting invalid proofs must not be rational.
2. Disputing invalid proofs must be rational.
3. Including proofs dispute messages must be rational.

### Submitting Invalid Proofs Is Irrational

As long as one participant in the network is reliably checking and disputing invalid proofs,
submitting an invalid proof will incur a higher penalty than simply not submitting a proof, or
declaring the sector faulty.

Of course, there needs to be at least one party disputing proofs for this to work.

### Incentives For Checking Proofs 

Currently, there is one primary reason to check all the proofs in order to dispute the invalid ones: to ensure the healthy operation
of the network.  Organizations invested in the continued operation of the network and the
reliability of its storage have a vested interest in ensuring that all data purported to be stored
on the network is being correctly proven.

Specifically:

1. Checking proofs is relatively cheap compared with running a Filecoin miner. At the moment, _all_
   Filecoin nodes are verifying _all_ proofs.
2. If a significant number of invalid proofs go undisputed, the confidence in the Filecoin network
   would be affected, impacting the value of the token. Given that all large miners have large
   amounts of vesting Filecoin rewards, they have a strong incentive to ensure this doesn't happen.

Disputing a proof also yields a minimal reward. Unfortunately, given the expected rarity of invalid
proofs, no reward will fully cover the cost of actually checking all these proofs. However, the
reward will at least cover the cost of submitting the dispute, and provide a little extra as an
incentive.

### Including Proof Dispute Messages Is Rational

Proof dispute messages directly reduce the power of other miners on the network, proportionally
increasing the relative power of all other miners in the network. Therefore, barring collusion, it is
always rational to include dispute messages.

Furthermore, proof dispute messages give a reward. Assuming the base fee isn't so high that the
reward doesn't cover the gas fees, a rational miner will front-run proof dispute messages to collect
the reward.

## Product Considerations

This FIP will increase chain bandwidth by moving Window PoSt processing off-chain. This will reduce
the per-gigabyte network-wide maintenance cost, removing a scalability constraint on the total
amount of storage that can be supported by the network.

## Implementation

The changes to the miner actor are in: https://github.com/filecoin-project/specs-actors/pull/1327

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
