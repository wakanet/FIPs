---
fip: "0016"
title: Pack arbitrary data in CC sectors
author: donghengzhao (@1475)
discussions-to: https://github.com/filecoin-project/FIPs/issues/57
status: Draft
type: Technical
category: Core
created: 2021-30-03
spec-sections:
   - https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go
---

## Simple Summary 简述

For CC sectors, allow miners to choose if they want to seal empty data or pack data pieces which could be real for CC
sector, and it can be verified accordingly.

对于CC扇区，允许矿工选择是否要密封空数据或打包对于CC扇区可能是真实的数据段，并相应地进行验证。

## Abstract 摘要

Currently in seal process, CC sectors are kind of wasteful in terms of storage. In the CC scenario, miners pack one big piece (all zeros) that the size of entire sector, after porep finished, create precommit messages with empty dealID information. In verification of commit messages, verifier will get the DealID from corresponding precommit info, and if DealID is empty, verifier recover the unsealed id by all zeros locally, which limit the CC sector to seal only zeroed data.
This proposal is aiming for raising this constrain, by changing the verification against arbitrary data miners choose to seal in the sectors, and keep the piece ID(s) in it for furhter use cases.

目前，在密封过程中，CC扇区在存储方面有点浪费。在CC场景中，矿工打包了一大块（全零），即整个扇区的大小，在porep完成后，创建带有空dealID信息的预提交消息。在验证提交消息时，验证器将从相应的预提交信息中获取DealID，如果DealID为空，验证者将在本地通过全零恢复未密封的id，这将限制CC扇区仅密封零数据。
本提案旨在通过改变针对任意数据矿工选择在扇区中进行密封的验证，并在其中保留工件ID，以用于更多用例，从而提高这一限制。

## Change Motivation

We propose design to allow miners to pack real data for CC sectors, add piece CID(s) in precommit message and keep it on chain. It introduce the possibility for CC upgrade without re-seal, and turn the piece into verified data afterwards (
Separated FIPs).

## Specification

1. Add data field “piece CIDs” in precommit.info and populate value in it in if it's a CC sector with non zero peice data.
4. Add Optional data field “piece CIDs” in SectorOnChainInfo (for further CC upgrade verification between unsealed id
   and piece CID)/SealVerifyStuff struct,
5. In getVerifyInfo, if “piece CIDs” of SealVerifyStuff is not empty and dealIDs is empty, than use it for commD computation, otherwise keep the logic as is.
6. In ConfirmSectorProofsValid, populate “piece CIDs” in the additional field of SectorOnChainInfo, this is for future
   verification

## Design Rationale

Extra onchain storage for “unsealed ID” in SectorOnChainInfo will be required. As this field provides the possibility
for post-verification between data pieces and unsealed ID.

## Backwards Compatibility

Fully backwards compatibility is expected

## Security Considerations

There will be no difference between "all zero piece data" or "pieces with meaningfull data" in terms of seal process. i.e. AddPiece, PreCommit1/PreCommit2, Commit1/Commit2 will keep as is. If we look it from seal/post perspective, The CC sectors with meaningful piece data will be exactly same with sectors with deal. There will be no advantage for miners possess this type of sectors. Therefore there is no extra security risk will be introduced by this FIP.

## Future Improvements

This proposal introduces the possibilities for further improvement (which could be in other FIPs) as
1, Integrate with Filecoin Plus program to complement deal and upgrade the CC sectors to be "verified" without reseal
2, Offchain deal system for save the block capacity
3, Miner commitment of multiple backups for one deal
4, Support retrieve pieces in CC in retrieval market

## Incentive Considerations

1, Miners have the option to leverage the committed capacity to store meaningful data, i.e, miners business data, chain store data, or offline shipped client data, instead of just plain "all zero" data. Therefore miners business model could be expanded and their hardware investment will be utilized in a more efficient manner
2, Filecoin network stored piece CIDs amount expected to be boomed. The value of whole network will be uplifted, and retrieval market could be more prosperous as the "Commodities" will be increased in it

## Implementation

As the design descripted above, we need a new package version of v6 and network migration to v14.

As we determined to limit same data type in one sector, either pieces with deals or pieces without deals, but cannot combines all together, I adjusted the method of unsealedCID to more simplicity.

1. add PieceInfos field to SectorPrecommitInfo and SectorOnChainInfo struct to store piece data cid and piece size for forward verify.
```
type SectorPreCommitInfo struct {
	SealProof       abi.RegisteredSealProof
	SectorNumber    abi.SectorNumber
	SealedCID       cid.Cid `checked:"true"` // CommR
	SealRandEpoch   abi.ChainEpoch
	DealIDs         []abi.DealID
	Expiration      abi.ChainEpoch
	PieceInfos      []abi.PieceInfo
	...
}

type SectorOnChainInfo struct {
	SectorNumber          abi.SectorNumber
	SealProof             abi.RegisteredSealProof // The seal proof type implies the PoSt proof/s
	SealedCID             cid.Cid                 // CommR
	DealIDs               []abi.DealID
	Activation            abi.ChainEpoch  // Epoch during which the sector proof was accepted
	Expiration            abi.ChainEpoch  // Epoch during which the sector expires
	DealWeight            abi.DealWeight  // Integral of active deals over sector lifetime
	VerifiedDealWeight    abi.DealWeight  // Integral of active verified deals over sector lifetime
	PieceInfos            []abi.PieceInfo //PieceInfos
	...
}
```

Rightnow, the `PieceInfos  []abi.PieceInfo` just present meaning of this field, for real environmnet it will be a cid.Cid to save onchain space. 
```
PieceInfos     cid.Cid 
```

2. change getVerifyInfo for diffrent way to get unsealedCIDs of cc sectors with piece data inside, if in single sector proven verify
```
builtin.RequireState(rt, len(params.DealIDs) > 0 && len(params.PieceInfos) > 0, "deals and raw pieces cannot exist at the same time in one sector")

commDs := requestUnsealedSectorCIDs(rt, &market.SectorDataSpec{
	SectorType: params.RegisteredSealProof,
	DealIDs:    params.DealIDs,
	Pieces:     params.PieceInfos,
})
```

3. change market.ComputeDataCommitment to allow map struct param to passed in.
```
type SectorDataSpec struct {
	DealIDs    []abi.DealID
	Pieces     []abi.PieceInfo
	SectorType abi.RegisteredSealProof
}
```
add a Pieces field to generate unsealed CID for raw piece sectors

```
func (a Actor) ComputeDataCommitment(rt Runtime, params *ComputeDataCommitmentParams) *ComputeDataCommitmentReturn {
	rt.ValidateImmediateCallerType(builtin.StorageMinerActorCodeID)

	var st State
	rt.StateReadonly(&st)
	proposals, err := AsDealProposalArray(adt.AsStore(rt), st.Proposals)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deal dealProposals")
	commDs := make([]cbg.CborCid, len(params.Inputs))
	for i, commInput := range params.Inputs {
		pieces := make([]abi.PieceInfo, 0)
		for _, dealID := range commInput.DealIDs {
			deal, err := getDealProposal(proposals, dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get dealId %d", dealID)

			pieces = append(pieces, abi.PieceInfo{
				PieceCID: deal.PieceCID,
				Size:     deal.PieceSize,
			})
		}
		for _, pieceInfo := range commInput.Pieces {
			pieces = append(pieces, abi.PieceInfo{
				PieceCID: pieceInfo.PieceCID,
				Size:     pieceInfo.Size,
			})
		}
		commD, err := rt.ComputeUnsealedSectorCID(commInput.SectorType, pieces)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to compute unsealed sectorCID: %s", err)
		commDs[i] = (cbg.CborCid)(commD)
	}
	return &ComputeDataCommitmentReturn{
		CommDs: commDs,
	}
}
```

if no deal in sectors, try generate unsealed cid from raw piece data


4. additionaly we limit the raw piece number in one sector like deals number and check this when precommit message verifying
```
// Determine maximum number of pieces miner's sector can hold
func SectorPiecesMax(size abi.SectorSize) uint64 {
	return max64(256, uint64(size/DealLimitDenominator))
}
```

```
if len(precommit.PieceInfos) > 0 && len(precommit.DealIDs) > 0 {
	rt.Abortf(exitcode.ErrIllegalArgument, "deals and raw pieces cannot exist at the same time in one sector")
}

if len(precommit.PieceInfos) > int(SectorPiecesMax(abi.SectorSize(precommit.SealProof))) {
	rt.Abortf(exitcode.ErrIllegalArgument, "too many pieces data in one sector")
}
```

5. so miners can insert data they need, to preform cc sector sealing and get disk space more useful

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
