---
fip: "0013"
title: Add ProveCommitSectorAggregated method to reduce on-chain congestion
author: Anca (@ninitrava), Nicola (@nicola), Zenground0, nemo (@cryptonemo), nikkolasg, jbenet, zixuanzh
discussions-to: https://github.com/filecoin-project/FIPs/issues/50
status: Final
type: Technical
category: Core
created: 2021-02-17
spec-sections: 
  - section-systems.filecoin_mining.sector.lifecycle

---

## Simple Summary 简述

<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
Add a method for a miner to submit several sector prove commit messages in a single one.
添加一种方法，使矿工可以在单个扇区中提交多个扇区证明提交消息。

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->
On-chain proofs scale linearly with network growth. This leads to (1) blockchain
being at capacity most of the time leading to high base fee, (2) chain capacity
is currently limiting network growth.

The miner `ProveCommitSector` method only supports committing a single sector at
a time.  It is both frequently executed and expensive.  This proposal adds a
way for miners to post multiple `ProveCommit`s at once in an aggregated fashion
using a `ProveCommitSectorAggregated`. This method amortizes some of the costs
across multiple sectors, removes some redundant but costly checks and
drastically reduces per-sector proof size and verification times taking
advantage of a novel cryptography result. This proposal also increases the delay
between pre-commit and prove-commit to allow miners of all sizes to enjoy the
highest gas saving factor.


## Change Motivation

<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

The miner `ProveCommitSector` method only supports committing a single sector at
a time.  It's one of the two highest frequency methods observed on the chain at
present (the other being `PreCommitSector`).  High-growth miners commit sectors
at rates exceeding 1 per epoch.  It's also a relatively expensive method, with
multiple internal sends and state loads and stores.

Aggregated proof verification allows for more sector commitments to be proven in
less time which will reduce processing time and therefore gas cost per prove
commit. Verification time and size of an aggregated proof scales logarithmically
with the number of proofs being included.

In addition to this primary optimization, there are several secondary
opportunities for improved processing time and gas cost related to amortizing state acceses costs through batch processing many prove commits at once: 

- Using a bitfield to specify sector numbers in the `ProveCommitSector` parameters could reduce message size
- PreCommits loading overhead can be done once per batch
- Power actor claim validation can be done once per batch
- Market actor `ComputeDataCommitment` calls can be batched

Additionally the `ProveCommitSectorAggregated` method can do away with the
temporary storage and cron-batching currently used to verify individual prove
commits. This opens further cost reduction opportunities:

- PreCommit info can be loaded once per prove commit rather than once for recording, and again for batch verifying in the cron callback
- With no processing needed in the cron callback, sectors proven through `ProveCommitSectorAggregated` will not need to be stored and read from the power actor's `ProofValidationBatch` map

In summary if miner operators implement a relatively short aggregation period,
the `ProveCommitAggregated` method has the potential to reduce gas costs for:

- State operations: some of the costs listed above can be amortized across
  multiple sectors
- Proof verification: the gas used for proofs can scale sub-linearly with the
  growth of the network using a novel proof aggregation scheme.

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

### Actor changes

Add a new method `ProveCommitSectorAggregated` which supports a miner
prove-committing a number of sectors all at once.  The parameters for this
method are a list of prove-commit infos:

```
type ProveCommitSectorAggregatedParams {
    SectorsNumbers bitfield.BitField
}
```

Semantics will be similar to those of `ProveCommitSector` with the following proposed changes:

- Use a SectorNumber bitfield in place of a single abi.SectorNumber in parameters
- Include an Aggregate proof in place of single porep proof in parameters
- MaxProveCommitSize parameter becomes MaxAggregateProofSize = 81,960
- Minimum and maximum number of sectors proven will be enforced
- PreCommitInfos read in batch
- SealVerifyInfos constructed in batch
- Market actor ComputeDataCommittment method changed to compute batches of commDs
- Gas cost for verification will be updated and now computed as a function of the number of sectors aggregated
- No storing proof info in power actor for batched verification at the end of the epoch.
- `ProveCommitSectorAggregated` will call into a new runtime syscall `AggregateVerifySeals` in place of power actor `BatchVerifySeals` call.
- ConfirmSectorProofsValid logic will be copied over to the second half of `ProveCommitSectorAggregated`.

#### Failure handling

- If any predicate on the parameters fails, the call aborts (no change is persisted).
- If the miner has insufficient balance for all prove-commit pledge, the call aborts.

#### Scale and limits

The number of sectors that may be proven in a single aggregation
is a minimum of 4 and a maximum of 819

MaxProveCommitDuration, the enforced delay between pre-commit and prove-commit is also increased from 1 day + PreCommitChallengeDelay to 30 days + PreCommitChallengeDelay. This does not impact security of the system it only increases the possible time pre-commits remain on chain before expiry.  See the section about incentives for motivation.

#### Gas calculations

Similar to existing PoRep gas charges, gas values are determined from empirical measurements of aggregate proof validation on representative hardware. Each PoRep count corresponding to a power of two snark count will be assigned to a gas table. See the "Proof scheme changes" section for discussion on padding to the next power of two for motivation. The gas charged to verify an aggregate proof in `ProveCommitSectorAggregated` includes a minor linear component and a major logarithmic component. For the linear component a fixed gas cost is charged per sector being proven. For the logarithmic component the aggregate size is rounded up to the nearest number in the relevant gas table and added to the total. Because verification costs are slightly different there is one linear constant and gas table for each of 32 GiB and 64 GiB sectors.

##### 32 GiB Gas Cost

Minor cost per sector: 449,900 gas

Major cost by rounded aggregate size:
- 4-6 PoReps (64 snarks): 103,994,170
- 7-12 PoReps (128 snarks): 112,356,810
- 13-25 PoReps (256 snarks): 122,912,610
- 26-51 PoReps (512 snarks): 137,559,930
- 52-102 PoReps (1024 snarks): 162,039,100
- 103-204 PoReps (2048 snarks): 210,960,780
- 205-409 PoReps (4096 snarks): 318,351,180
- 410-819 PoReps (8192 snarks): 528,274,980

##### 64 GiB Gas Cost

Minor cost per sector: 359,272 gas

Major cost by rounded aggregate size:
- 4-6 PoReps (64 snarks): 102,581,240
- 7-12 PoReps (128 snarks): 110,803,030
- 13-25 PoReps (256 snarks): 120,803,700
- 26-51 PoReps (512 snarks): 134,642,130
- 52-102 PoReps (1024 snarks): 157,357,890
- 103-204 PoReps (2048 snarks): 203,017,690
- 205-409 PoReps (4096 snarks): 304,253,590
- 410-819 PoReps (8192 snarks): 509,880,640

#### Batch Gas Charge

Currently, **GasUsed * BaseFee** is burned for every message. We can achieve the requirements described in [Incentive Considerations](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md#incentive-considerations) by charging an additional proportional gas cost to an aggregated batch of proofs, and by balancing their gas costs with a minimum gas fee. Three concepts need to be introduced:
- **BatchGasCharge** - the network fee for adding batched proofs
- **BatchBalancer** - a minimum gas fee for BatchGasCharge
- **BatchDiscount** - a heavy discount for aggregated proofs, which also benefits other messages

The following charge is calculated for each **BatchProveCommit** message.

```
func PayBatchGasCharge(numProofsBatched, BaseFee) {
// Cryptoecon Params (need to be updated if verification benchmarks change)
BatchDiscount = 1/20 unitless
BatchBalancer = 2 nanoFIL
SingleProofGasUsage = 65733296.73

// Calculating BatchGasCharge
numProofsBatched = <# of proofs in this batched operation>
BatchGasFee = Max(BatchBalancer, BaseFee)
BatchGasCharge = BatchGasFee * SingleProofGasUsage *  numProofsBatched * BatchDiscount

// Pay for the batch
PayNetFee(BatchGasCharge) // this can be a msg.Send to f99. Does not affect BaseFee
// normal gas for the verification computation is paid as usual (using & affecting BaseFee)
}
```

Implications and rough estimates for this function are described in [Batch Incentive Alignment](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md#batch-incentive-alignment).

#### State Migrations

Neither changes to the state schema of any actors nor changes to the fields of existing actors are required to make this change. Therefore a state migration is not needed.


### Proof scheme changes

Protocol Labs research teams in collaboration with external researchers have
worked on an improvement of the Inner Product Pairing result from [Bunz et
al.](https://eprint.iacr.org/2019/1177.pdf).

In high level, the idea is the following: given some Groth16 proofs, one can
generate a single proof of logarithmic size that these were correctly aggregated.

A major transformation from the paper is that this scheme works in the settings
of Filecoin as-is; there is no need for another curve or trusted setup.
More specifically, it works by re-using Filecoin trusted setup and taking Zcash
trusted setup together to provide the aggregating proving and verifiying key. 

A more detailed technical report on the new constructions can be found here (TODO).

#### Proofs API

##### RegisteredAggregationProof

The API changes introduce a new type RegisteredAggregationProof for future proofing the aggregation scheme. As of this change there is only one valid member of the RegisteredAggregationProof type corresponding to the new aggregation scheme.

##### Aggregation

The proofs aggregation procedure expects the following inputs:

```rust
pub fn aggregate_seal_commit_proofs(
    registered_proof: RegisteredSealProof,
    registered_aggregation: RegisteredAggregationProof,
    comm_rs: &[Commitment],
    seeds: &[Ticket],
    commit_outputs: &[SealCommitPhase2Output],
) -> Result<AggregateSnarkProof>;
```

The `comm_rs` are an ordered list of public replica commitments and `seeds` are an ordered list of randomness used to generate seal proof challenges.  The `commit_outputs` are the objects returned from the seal commit phase2 API.  The idea is that multiple sectors have been properly committed, and those outputs are compiled into a list for aggregation at some point later in time.

**Requirements**: The scheme can only aggregate a power of two number of proofs
currently. Although there might be some ways to alleviate that requirement, we
currently pad the number of input proofs to match a power of two. Thanks to the
logarithmic nature of the scheme, performance is still very much acceptable.

Padding is currently _naive_ in the sense that if the passed in count of seal proofs is not a power of 2, we arbitrarily take the last proof and duplicate it until the count is the next power of 2.  The single exception is when the proof count is 1.  In this case, we duplicate it since the aggregation algorithm cannot work with a single proof.

##### Verification

The proofs verification procedure expects the following inputs:

```rust
pub fn verify_aggregate_seal_commit_proofs(
    registered_proof: RegisteredSealProof,
    registered_aggregation: RegisteredAggregationProof,
    aggregate_proof_bytes: AggregateSnarkProof,
    comm_rs: &[Commitment],
    seeds: &[Ticket],
    commit_inputs: Vec<Vec<Fr>>,
) -> Result<bool>;
```

The `comm_rs` are an ordered list of public replica commitments and the `seeds` are an ordered list of randomness used to generate seal proof challenges.

The `commit_inputs` above also have a specific order to them, which *must* match the order of the `commit_outputs` passed into `aggregate_seal_commit_proofs`, but in a flattened manner.  First, to retrieve the `commit_inputs` for a single sector, you can call this:

```rust
pub fn get_seal_inputs(
    registered_proof: RegisteredSealProof,
    comm_r: Commitment,
    comm_d: Commitment,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    seed: Ticket,
) -> Result<Vec<Vec<Fr>>>;
```

As an example, if `aggregate_seal_commit_proofs` is called with the `commit_outputs` of Sector 1 and Sector 2 (where that order is important), we would want to compile the `commit_inputs` for verification as follows (pseudo-code for readability):

```rust
let commit_inputs: Vec<Vec<Fr>> = Vec::new();
commit_inputs.extend(get_seal_inputs(..., sector_one_id, ...));
commit_inputs.extend(get_seal_inputs(..., sector_two_id, ...));
```

What this example code does is flattens all of the individual proof commit inputs into a single list, while properly maintaining the exact ordering matching the `commit_outputs` order going into `aggregate_seal_commit_proofs`.  When compiled like this, the `commit_inputs` will be in the exact format required for the `verify_aggregate_seal_commit_proofs` API call.

Similar to aggregation, padding for verification is currently also _naive_. If the passed in count of proof input sets (while noting that the inputs are a linear list of equally sized input sets) is not a power of 2, we arbitrarily take the last set of inputs and duplicate it until the count is the next power of 2.  Again, the single exception is when the input count is 1.  In this case, we duplicate it since the verification algorithm cannot work with a single proof or input.


#### Proofs format 

**Notation**: G_1, G_2, and G_t represents the first, second and target group of
the pairing curve, in this case BLS12-381. Fr represents the scalar field.

**Structure**:A proof can be represented as in the following structure: 
```
struct AggregatedProof {
    // Groth16 part
    com_ab0 (G_t, G_t)
    com_c (G_t, G_t)
    ip_ab G_t
    agg_c G_1
    // TIPP/MIPP part
    nproofs u32
    comms_ab [(G_t,G_t),(G_t,G_t)] // a vector of size ceil(log(nproofs))
    comms_c [(G_t,G_t),(G_t,G_t)] // a vector of size ceil(log(nproofs))
    z_ab [(G_t, G_t)] // a vector of size ceil(log(nproofs))
    z_c [(G_1, G_1)] // a vector of size ceil(log(nproofs))
    final_a G_1
    final_b G_2
    final_c G_1
    final_r Fr,
    final_vkey (G_2, G_2)
    final_wkey (G_1, G_1)
}
```

**Serialization**:
* Fields of the struct are serialized in order using little endian mode.
* The field `nproof` is used to determine the size of the vectors that must be
  read afterwise.
* Fr, G_1 and G_2 are serialized according to the Appendix A in the
  [RFC](https://tools.ietf.org/html/draft-irtf-cfrg-bls-signature-04#appendix-A)
  spec that follows out ZCash definition
* G_t serialized (respectively deserialized) using a compression technique from
  "On Compressible Pairings and Their Computation" by Naehrig et al. You can
  find the reference code on the [RELIC
  library](https://github.com/relic-toolkit/relic/commit/b3d831734d8bf3c531a738b9201603fb51108034#diff-a678346b3ab6e13b60395e116deeae7931b3b738c649e89cc146cbec8d00afbcR109).


## Design Rationale

The existing `ProveCommitSector` method will not become redundant, since
aggregation of smaller batches may not be efficient in terms of gas cost (proofs
too big or too expensive to verify).  The method is left intact to support
smooth operation through the upgrade period.

### Failure handling

Aborting on any precondition failure is chosen for simplicity. 
Submitting an invalid prove commitment should never happen for correctly-functioning miners.  Aborting on failure will provide
a clear indication that something is wrong, which might be overlooked by an operator otherwise.

Submitting an aggregate including an already proven sector is a failure.

### Scale and limits

Each aggregated proof is bounded at 819 sectors. The motivation for the bound on aggregation size is as follows:

- to limit the impact of potentially mistaken or malicious behaviour.  
- to gradually increase the scalability, to have time to observe how the network
  is growing with this new method.
- to ensure the gas savings are equally accessible to small miners with a sealing rate as low as 1TB/day

A miner may submit multiple batches in a single epoch to grow faster.

## Backwards Compatibility

This proposal introduces a new exported miner actor method, and thus changes the
exported method API.  While addition of a method may seem logically backwards
compatible, it is difficult to retain the precise behaviour of an invocation to
the (unallocated) method number before the method existed.  Thus, such changes
must be delivered through a major version upgrade to the actors.

This proposal retains the existing non-batch `ProveCommitSector` method, so
mining operations need not change workflows due to this proposal (but _should_
in order to enjoy the reduced gas costs).

## Test Cases

Test cases will accompany implementation. Suggested scenarios include:

1. ProveCommitAggregate with number of sectors below `MinAggregatedSectors` (still tbd depending on final verification gas values) with all preconditions succeeding => failures due to violation of minimum aggregated sectors
2. ProveCommitAggregate with number of sectors above `MaxAggregatedSectors` (still tbd) with all preconditions succeeding => failure due to violation of maximum aggregated sectors
3. ProveCommitAggregate with with one sector already been proven => failure
4. ProveCommitAggregate with # of sectors 20, all sectors with deals => OK and deals activated
5. ProveCommitAggregate with # of sectors 20, one sector with one deal => OK an deal activated
6  ProveCommitAggregate with failing aggregate verification => failure
7. ProveCommitAggregate with one precommit out of date => failure
8. ProveCommitAggregate with 20 sectors, 19 have same PreCommitEpoch, 1 has PreCommitEpoch+1 => OK
9. ProveCommitAggregate with 20 sectors, 19 have same SealRandEpoch, 1 has SealRandEpoch+1 => OK
10. ProveCommitAggregate with 20 sectors, not enough funds to cover pledge => failure

## Security Considerations

All significant implementation changes carry risk.

The core cryptographic techniques come from the [Bunz et
al.](https://eprint.iacr.org/2019/1177.pdf) paper from 2019 with strong security
proofs.  Our protocol is a derivation of this paper that is able to use the
already existing powers of tau and with increased performance. The paper is also
accompanied by formal security proofs in the same framework as the original
paper. The trusted setup used is the Filecoin Powers of Tau and the ZCash Powers
of Tau: the full key is generated thanks to this
[tool](https://github.com/nikkolasg/taupipp). The cryptographic implementation
is being audited and a report will be made available soon.

## Incentive Considerations

Requirements. The cryptoeconomics of FIP13 require balancing a number of different goals and constraints:

- **Miner Fairness.** Small, Medium, and Large miners must have proportional gas economics -- there should be no exploitative economy of scale that benefits larger miners disproportionately. Costs should remain balanced across strategies. The cost reduction should be a well-defined discount, and should be available to small miners.

- **Share the gas cost reduction w/ other messages (especially Storage Deals).** Storage onboarding represents a huge fraction of the transactions in the network -- these increase the BaseFee and make other transactions (eg Storage Deals) very expensive. Aggregations enable a separate “gas lane” for onboarding that can keep gas costs low for other messages (eg Deals, Sends, and future contracts). This is key for deal-making, useful storage, and FIL DeFi.

- **Aligning with the Network & Paying the Network.** Storage onboarding is very profitable for miners who engage in it. Allowing orders of magnitude faster onboarding will benefit many miners who grow their operations significantly. This great capacity increase and great cost reduction in storage onboarding must be incentive-aligned with the broader network ecosystem and economy. All actors around the network should benefit from this onboarding, and the best way to do so is to pay the network an appropriate network gas fee.

- **Make Base Fee Spiking Attacks Expensive.** Some attackers may attempt to spike the base fee to make it expensive for small or new miners to onboard their storage. Cheap aggregation reduces the attack surface for miners who aggregate, but would shift the target to miners who do not aggregate, or miners who use deals and filled sectors. It is key that BaseFee attacks become even more expensive to mount.

### Batch Incentive Alignment
Given the implementation described in [Batch Gas Charge](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md#batch-gas-charge), we achieve the following benefits:

- **BatchDiscount** and **BatchBalancer** are set to balance the power between small and large miners, and align participants' incentives with the long term health and success of the network. 
- By adding a **BatchGasCharge**, large players are paying proportional network transaction fees to the network based on the amount of storage that they are adding without affecting the underlying **GasUsage** or **BaseFee**. 
- By using a separate gas lane, the gas savings are passed as a big cost reduction to other messages, likely reducing the **BaseFee**.
- **BatchBalancer** establishes a regulating feedback process in the gas market to keep the **BaseFee** low for other operations but still meaningful in network transaction fees for batch commits. When the **BaseFee** is lower than **BatchBalancer * BatchDiscount**, some miners may find it more attractive to submit commit messages for individual proofs. When the **BaseFee** approaches **BatchBalancer * BatchDiscount**, miners may switch to batch commit messages to take advantage of the cost savings. In turn, this reduces load on the **BaseFee**.
- **BaseFee** spiking attacks are not neutralized, but are more expensive to mount, as (a) most of the chain throughput will move into aggregation, making it much more expensive for an attacker to increase and sustain the base fee, and (b) it is even more expensive for large miners to mount such an attack while growing their own storage.

**Rough Estimates.** (these are ballpark estimates and are likely wrong -- make your own models and measurements)

- With **BatchDiscount = 1/20** and **BatchBalancer = 2 nFIL**, we hope **BaseFee** will reduce from current avg **(1-2 nFIL)** to ~ **0.15 nFIL** with present message distributions plus the increases in storage onboarding throughput.
- With **BaseFee ~ 0.15 nFIL**, **PublishStorageDeals** could cost about **7 mFIL**. (down from 182 mFIL)
- Amortized unit **ProveCommit** gas costs may drop below **5 - 10 mFIL** (down from **50 - 100 mFIL**).

To illustrate the balancing dynamic of **BatchBalancer** and **BatchDiscount** at work, here are the unit network fees for a 32GiB sector at different BaseFee levels. Note that unit storage network fees may be halved when 64GiB sectors are used.

At a network BaseFee of 0.01 nanoFIL, unit economics is in favor of adding single proofs to the network.

<img width="681" alt="0.01nFIL" src="https://user-images.githubusercontent.com/16908497/119754607-5944f380-bed3-11eb-80c3-2082197efb3b.png">

As BaseFee increases to 0.1 nFIL, unit sector network fee for a single proof catches up to that of aggregated proofs.

<img width="680" alt="0.1nFIL" src="https://user-images.githubusercontent.com/16908497/119754633-6366f200-bed3-11eb-9a1c-ef86bfa57d92.png">

A crossover in the unit economics happens at around 0.15nFIL where miners are incentivized to take advantage of proof aggregation to free up more chain capacity. This will thus create a damping force on the BaseFee. 

<img width="685" alt="0.15nFIL" src="https://user-images.githubusercontent.com/16908497/119754760-7c6fa300-bed3-11eb-806a-c59f70941298.png">

In the hypothetical event when the BaseFee continues to rise to 1 nFIL or 2 nFIL (which is considered low today), miners are strongly incentivized to aggregate and take advantage of the savings that proof aggregation brings.

<img width="675" alt="1nFIL" src="https://user-images.githubusercontent.com/16908497/119754793-872a3800-bed3-11eb-9b0b-7e0fafe54d13.png">
<img width="692" alt="2nFIL" src="https://user-images.githubusercontent.com/16908497/119754857-a1641600-bed3-11eb-910b-37dcfa1e4787.png">

In aggregate, daily network fee spend grows as the network grows in size. With Single Proofs, it is impossible for the network to grow at >40PiB/day and maintain a 0.2nFIL BaseFee due to the constraint on chain capacity. Batch Proofs, however, enable the network to grow at > 800PiB/day.

### MaxProveCommitDuration incentives

Given the logarithmic nature of the verification algorithm, the _reduction in
gas spent_ in verifying an aggregated proof is _higher_ when the number of
proofs aggregated is _higher_. In short, the more proofs one aggregates, the less
gas one spends _per proof_. 

While this feature is desirable to reduce fees and scale up the emboarding rate,
miners with a low emboarding rate may not be able to have the time to pack
enough prove commits to enjoy the largest gas reduction benefits. For example, a
miner that can aggregate 800 proofs in a day will be able to enjoy a 20x gas
reduction (theoretical numbers) while a miner that can only aggregate 200 proofs
a day will enjoy "only" a 10x gas reduction. Note that the gas reduction here is
_not_ logarithmic because all the states operations performed in the
`ProveCommitSectorAggregated` do not scale logarithmically with batch size.

In order to support similar gas improvements for all sizes of miners on the Filecoin network, this FIP
proposes to:

1. Increase the maximum delay between pre-commit and prove-commit to 30 days 
2. Limit the number of aggregated proofs to 819 (`=floor(8192/10)`)

This FIP consider the baseline of 1TB/day of sealing capacity, or 32 sectors,
for a "small" miner. By allowing a larger delay up to 30 days, such a miner can
aggregate up to 960 proofs in 30 days. Using this strategy, small miners can
benefit from an equal per sector gas reduction.

 
## Product Considerations

This proposal reduces the aggregate cost of committing new sectors to the
Filecoin network. 

This will reduce miner costs overall, as well as reduce contention for chain
transaction bandwidth that can crowd out other messages. It unlocks a
larger storage onboarding rate for the Filecoin network.

## Implementation

* Cryptographic implementation is currently located on the `feat-ipp2` of the bellperson [repo](https://github.com/filecoin-project/bellperson/tree/feat-ipp2/src/groth16/aggregate)
* Integration between Lotus and crypto-land can be found in [rust-fil-proofs](https://github.com/filecoin-project/rust-fil-proofs/tree/feat-aggregation) and the FFI [here](https://github.com/filecoin-project/filecoin-ffi/blob/feat-aggregation).
* Actors changes are in progress here: https://github.com/filecoin-project/specs-actors/pull/1381
* Lotus integration putting everything together is in progress here: https://github.com/filecoin-project/lotus/pull/5769

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
