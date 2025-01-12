---
fip: "0008"
title: Add miner batched sector pre-commit method
author: Alex North (@anorth), @ZenGround0, @nicola
discussions-to: https://github.com/filecoin-project/FIPs/issues/25
status: Final
type: Technical
category: Core
created: 2020-11-04
spec-sections: 
  - section-systems.filecoin_mining.sector.lifecycle
---

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
Add a method for a miner to submit sector pre-commitments in batches.
添加矿工批量提交扇区预承诺的方法。

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
The miner `PreCommitSector` method only supports committing a single sector at a time.
It is both frequently executed and relatively expensive.
This proposal adds a `PreCommitSectorBatch` method to amortize some of the costs across multiple sectors,
and removes some redundant but costly checks.


## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->
The miner `PreCommitSector` method only supports committing a single sector at a time. 
It's one of the two highest frequency methods observed on the chain at present (the other being `ProveCommitSector`). 
High-growth miners commit sectors at rates exceeding 1 per epoch. 
It's also a relatively expensive method, with multiple internal sends and loading and storing state including:
- the Storage Power actor's power total (read)
- the Storage Market actor's deal proposal AMT (read)
- the Reward actor's totals (read)
- the `AllocatedSectors` bitfield (read and modify)
- the `PrecommittedSectors` HAMT (read and modify)
- the `Sectors` AMT (read)
- the `PreCommittedSectorsExpiry` AMT (read and modify)
- the Storage Power actor's pledge total (read and modify)

A `PreCommitSectorBatch` method has potential to amortize some of these costs across multiple sectors. 
If miner operators implemented a relatively short batch aggregation period (a few epochs), the number of invocations could be reduced significantly, 
and some of the state manipulations above reduced in proportion.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->
Add a new method `PreCommitSectorBatch` which supports a miner pre-committing a number of sectors all at once.
The parameters for this method are a list of pre-commit infos:

```
type PreCommitSectorBatchParams {
    Sectors []*SectorPreCommitInfo
}
```

For the most part, semantics should be equivalent to invoking `PreCommitSector` with each sector info in turn. Some notable changes include:
- The reward and power stats use for calculating deposits need be fetched only once.
- The market actor `VerifyDealsForActivation` should be batched similarly, so it can be invoked just once.
- Sector numbers are allocated in batch, rather than mutating the bitfield for each sector (this is a big cost saving).
- The `PreCommittedSectors` HAMT check is removed, since it is redundant after checking the sector number allocation.
- The `Sectors` AMT check is removed, since it is redundant after checking the sector number allocation.
- The `PreCommittedSectors` HAMT is loaded and stored just once, to store all new sector information in batch.
- The `PreCommittedSectorsExpiry` AMT queue is loaded and stored just once for the batch.

The redundant state checks are removed from the non-batch `PreCommitSector` method too.

### Failure handling
- If any predicate on the parameters fails, the call aborts (no change is persisted).
- If the miner has insufficient balance for all pre-commit deposits, the call aborts.

### Scale and limits
The number of sectors that may be pre-committed in a single batch is limited to 256. 


## Design Rationale
The existing `PreCommitSector` method will become redundant, since a batch of one will not be significantly less efficient.
The method is left intact to support smooth operation through the upgrade period.
Support for the `PreCommitSector` method may be dropped in a future FIP.

### Failure handling
Aborting on any precondition failure is chosen for simplicity. 
There is no good reason for submitting an invalid pre-commitment, so this should never happen for correctly-functioning miners. 
Aborting on failure will provide a clear indication that something is wrong, which might be overlooked by an operator otherwise.

An alternative could be to allow sectors in the batch to succeed or fail independently. 
In this case, the method should return a value indicating which sectors succeeded and which failed.
This would complicate both the actor and node implementations somewhat, though not unduly.

### Scale and limits
The bound on batch size is not intended to actually constrain a miner's behaviour, but limit the impact of potentially mistaken or malicious behaviour.
256 sectors per epoch would support a single miner onboarding 8EiB of 32GiB sectors in 1 year.

A miner may submit multiple batches in a single epoch to grow faster.


## Backwards Compatibility
This proposal introduces a new exported miner actor method, and thus changes the exported method API. 
While addition of a method may seem logically backwards compatible, it is difficult to retain the precise behaviour of an invocation to the (unallocated) method number before the method existed.
Thus, such changes must be delivered through a major version upgrade to the actors.

This proposal retains the existing non-batch `PreCommitSector` method, so mining operations need not change workflows due to this proposal (but _should_ in order to enjoy the reduced gas costs).

## Test Cases

Test cases will accompany implementation. Suggested scenarios include:

1. PreCommitSectorBatch with # of sectors 10, with all preconditions succeeding => OK.
2. PreCommitSectorBatch with # of sectors 1, with all preconditions succeeding => OK and gas cost similar to PreCommitSector.
3. PreCommitSectorBatch with # of sectors 33, with all preconditions succeeding => failure due to violation of maximum sector count (256).
4. PreCommitSectorBatch with empty sector array, with all preconditions succeeding => failure?
5. PreCommitSectorBatch with # of sectors 10, with insufficient (starting) balance to carry out pledge 1 => failure and state untouched.
6. PreCommitSectorBatch with # of sectors 10, with insufficient balance to carry out pledge 10 => failure and state untouched.
7. PreCommitSectorBatch with # of sectors 256, all sectors with deals => OK and deals activated. (Note from @ZenGround0 Deals aren't activated until ProveCommit so for this to make sense this test should include subsequent prove commit.)
8. PreCommitSectorBatch with # of sectors 256, one deal in sector 20 => OK and deal activated.

## Security Considerations
All significant implementation changes carry risk. This change is not particularly complex, and existing validation practises and mechanisms are expected to suffice.

## Incentive Considerations
This proposal amortizes per-sector costs for high-growth miners, providing an economy of scale. This same economy cannot be enjoyed by miners growing more slowly.

This may present a minor incentive against splitting a single physical operation into many miner actors (aka Sybils).

## Product Considerations
This proposal reduces the aggregate cost of committing new sectors to the Filecoin network. 

This will reduce miner costs overall, as well as reduce contention for chain transaction bandwidth that can crowd out other messages.

## Implementation
Implementation to follow discussion and acceptance of this proposal.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
