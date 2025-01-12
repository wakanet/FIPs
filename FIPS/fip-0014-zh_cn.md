---
fip: "0014"
title: Allow V1 proof sectors to be extended up to a maximum of 540 days
author: deltazxm (@deltazxm), Neo Ge(@neogeweb3), Fatman13(@Fatman13 )
discussions-to: https://github.com/filecoin-project/FIPs/issues/56
status: Final
type: Technical
category: Core
created: 2021-26-02
spec-sections: 
  - section-systems.filecoin_mining.sector.lifecycle

---

## Simple Summary 简述
Allow V1 proof sectors to be extended up to a maximum lifetime of 540 days.

允许V1验证扇区延长到540天的最大使用寿命。

## Abstract
Allow sectors sealed using V1 proof before network version 7 on Nov 27th, 2020 to be extended up to a maximum lifetime of 540 days including the days that they have already been active. This means the latest a v1 sector will be live on the network is May 21, 2022.

## Change Motivation
While every encryption/proof created ever by human beings has security issues, V1 proof is now being challenged with unrealistically high standard of security concerns. This fip advocates that V1 proof sectors should be treated equally just like any other sectors.

## Specification
Key changes proposed by @steven004 could be found [here](https://github.com/filecoin-project/FIPs/pull/75#issuecomment-789405523).

## Design Rationale
The design is quite straight forward. Suggestions are welcome.

## Backwards Compatibility
TODO: Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.

## Security Considerations
Security concerns from the community boils down to @nicola's following comment.
> There is a risk that a year+ from now, with highly pipelined custom hardware (less likely to be software - but not impossible), one may able to avoid storing 100% of the sector and still succeed at WindowPoSt.**
> As we come closer to these sectors' expiration (most are a ~year from now), depending on (1) what percentage of power these sectors sum up to, (2) what is the state of the art of ASICs and Proof of Space software, the community should decide what to do with those sectors.
> I would not recommend to allow for sector extension right now.

### Cost of designing custom hardware (ASICs)
From [data](https://semiengineering.com/big-trouble-at-3nm/) aggregated by semiengineering, the design cost alone of developing a cutting edge chip could amount to ~$540M which is equivalent to the revenue of running a 120Pib node for 3 years given the current network parameters. If attackers were in this for profits, it would be extremely costly for them to recoup their initial investment.

### Time for development of custom hardware
It is estimated that manufacturing a chip from scratch could take [1 to 2 years](https://www.quora.com/How-long-does-it-take-to-build-a-computer-chip-from-scratch). By this point, most, if not all, v1 sectors will have expired.

### Securing production of custom hardware
There are mainly two foundries someone could get their chip design produced, namely from TSMC or Samsung. Attackers will be competing with all smart devices manufacturers for "production cap". [Months of waiting](https://www.dvhardware.net/article72264.html) time is the norm given how the market is craving smart devices nowadays.

### Incentives of attackers
Under the circumstances mentioned above, there wouldn't be any incentives for attackers to invest huge amount of time and money just to fake a discontinued proof.

### Core dev meeting
Many of the security considerations were also extensively discussed in [Filecoin Core Dev Meeting #13](https://github.com/filecoin-project/tpm/blob/master/Core%20Dev%20Meetings/Meeting%200013.md). Some of the key takeaways are...
> Core devs need to agree on final answers to:
> - whether adopting this FIP is acceptable from a security perspective
> - whether the community's overall preference is in fact for this FIP
> 
> The answer for the first question is tending towards "yes", and the answer to the second question should be found out via a community poll.

### Summary of security considerations
The general consensus is that it is not economically viable that an attacker could develop custom hardware using tech that is a year+ from now. Some highlights of community comments are by @neogeweb3 at [here](https://github.com/filecoin-project/FIPs/pull/75#discussion_r586057806) and by @steven004 at [here](https://github.com/filecoin-project/FIPs/issues/56#issuecomment-785715308).

## Incentive Considerations
The majority of the people who sealed sectors using V1 proof for 180 days will now most likely to extend them up to 540 days. 

We expect such behaviour thanks to one of crucial points that @steven004 brought up, which is [fairness](https://github.com/filecoin-project/FIPs/issues/56#issuecomment-794778752). As summed up by Aayush in this [thread](https://filecoinproject.slack.com/archives/C01EU76LPCJ/p1615524308014600?thread_ts=1615283805.008800&cid=C01EU76LPCJ)...

> My personal opinion is that it’s a fairness question — folks who sealed 6 month v1 sectors thinking they could be easily extended later are now in the unfortunate position of not being able to do so (while those who sealed them for 18 months will continue to have power for them).

## Product Considerations
This FIP protects interests of miners who have contributed to Filecoin's growth since mainnet launch. 

This FIP is both a both security/implementation decision and governance decision. The decision made in [v1.2.0](https://github.com/filecoin-project/lotus/releases/tag/v1.2.0) of not allowing extending V1 proof sectors was not an obvious catch. It wasn't until @deltazxm tried to extend a V1 proof sector and got an [error message](https://github.com/filecoin-project/specs-actors/issues/1309) that people realized the decision had been made. What if we find a bug in V1.1 tomorrow? Do we silently let V1.1 proof expire too? We need to establish some kind of process to reach consensus on making decisions like this together with different kinds of stakeholders in the network especially when changing policy, network parameters or other equivalents.

## Implementation

Set `SectorMaxLifetime` for v1 sectors to be 540 days

```go
const EpochsInFiveYears = stabi.ChainEpoch(5 * EpochsInYear)
const EpochIn540Days = stabi.ChainEpoch(EpochsInYear + EpochsInYear/2)

var SealProofPolicies = map[stabi.RegisteredSealProof]*SealProofPolicy{
	stabi.RegisteredSealProof_StackedDrg2KiBV1: {
		SectorMaxLifetime: EpochIn540Days,
	},
	stabi.RegisteredSealProof_StackedDrg8MiBV1: {
		SectorMaxLifetime: EpochIn540Days,
	},
	stabi.RegisteredSealProof_StackedDrg512MiBV1: {
		SectorMaxLifetime: EpochIn540Days,
	},
	stabi.RegisteredSealProof_StackedDrg32GiBV1: {
		SectorMaxLifetime: EpochIn540Days,
	},
	stabi.RegisteredSealProof_StackedDrg64GiBV1: {
		SectorMaxLifetime: EpochIn540Days,
	},

	stabi.RegisteredSealProof_StackedDrg2KiBV1_1: {
		SectorMaxLifetime: EpochsInFiveYears,
	},
	stabi.RegisteredSealProof_StackedDrg8MiBV1_1: {
		SectorMaxLifetime: EpochsInFiveYears,
	},
	stabi.RegisteredSealProof_StackedDrg512MiBV1_1: {
		SectorMaxLifetime: EpochsInFiveYears,
	},
	stabi.RegisteredSealProof_StackedDrg32GiBV1_1: {
		SectorMaxLifetime: EpochsInFiveYears,
	},
	stabi.RegisteredSealProof_StackedDrg64GiBV1_1: {
		SectorMaxLifetime: EpochsInFiveYears,
	},
}
```

And remove the following condition in ExtendSectorExpiration method:

```go
if !CanExtendSealProofType(sector.SealProof) {
		rt.Abortf(exitcode.ErrForbidden, "cannot extend expiration for sector %v with unsupported seal type %v",
		sector.SectorNumber, sector.SealProof)
}
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
