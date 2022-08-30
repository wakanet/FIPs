---
fip: 0019
title: Snap Deals
author: @Kubuxu, @lucaniz, @nicola, @rosariogennaro, @irenegia
discussions-to: https://github.com/filecoin-project/FIPs/issues/145
status: Final
type: Core
created: 2021-08-24
spec-sections: 
  - miner actors
  - lotus mining

---

## Simple Summary 简述

A one-message protocol for updating any sector with new data without re-sealing.

一种单消息协议，用于在不重新密封的情况下用新数据更新任何扇区。

## Abstract 摘要

Storage provider embeds deal data into existing sector replica with some unpredictable randomness by "xor"-ing.

存储提供商通过“异或”将交易数据嵌入到现有扇区副本中，具有一些不可预测的随机性。

The miner generates one message which proves that sector was updated with new data.

矿工生成一条消息，证明扇区已用新数据更新。

## Change Motivation 改变动机

Since 90+% of sectors in the Filecoin Network are CC sectors, having a protocol that allows for updating CC sectors to store real data without incurring in a full re-sealing would massively improve our network in terms of the amount of real data stored through it. 

由于Filecoin网络中90%以上的扇区是CC扇区，因此有一个协议允许更新CC扇区以存储真实数据，而不会导致完全重新密封，这将极大地改善我们的网络中通过它存储的真实数据量。

1. It would allow **decoupling sealing latency from deal-making speed** - offering storage clients an improved experience for how quickly their data can land in on-chain deals
1. 它将允许**将封存延迟与交易制定速度分离** - 为存储客户提供更好的体验，让他们的数据能够以多快的速度进入链上交易
2. It would **unlock the 8EiB of storage already committed to the Filecoin network** to be quickly used for deals - enabling a 100PiB+ client to make deals for their entire dataset with a single miner like [f0127595](https://filfox.info/en/address/f0127595) which already has 120PiB of committed capacity.
2. 它将**解锁已经提交给Filecoin网络的8EiB存储空间**，以便快速用于交易-使100PiB+客户端能够使用这样的单个矿工像[f0127595](https://filfox.info/en/address/f0127595)为其整个数据集进行交易其已具有120PiB的承诺容量。
3. It makes **utilizing existing committed capacity much cheaper for miners** (since they’ve already invested the sealing cost), increasing the chances they pay clients to add FIL+ data to these sectors.
3. 这使得**利用现有的承诺产能对采矿者来说更加便宜**（因为他们已经投资了密封成本），增加了他们向客户支付将FIL+数据添加到这些扇区的机会。

What this FIP does not support (see Future Work section):
* Deal updates or updating sectors with existing deals
* Moving deals across sectors


## Specification 规范

### Nomenclature 术语
**SectorKeyCID** - on-chain commitment of CC sector replica

**UnsealedSectorCID** - commitment of deal data generated from PieceCIDs

**SealedSectorCID** - commitment of sector replica with deals embedded in it

**EncodingRands** - collection of **HashShards** randomness values used for encoding process

**Poseidon-128** - SNARK friendly hash function


### Parameters 参数

**ChallengesNumber** = 1376 - number of challenges to be proven

**HashShards** = 512 - number of hashes used for randomness in encoding process


### One-messages Update Protocol 消息更新协议
1. **Storage Provider accepts and publishes storage deals (using `PublishStorageDeals`)**
2. **If necessary storage provider extends sector expiration so that sector is active long enough to contain deals with `miner.ExtendSectorExpiration`**
3. **Storage Provider updates an existing replica with deals**:
	1. Pre-generate HashShards possible number of `EncodingRand(ComputeUnsealedCID(P1, P2, P3, ...), i)`
	where `P1, P2, P3, ...` are pieces corresponding to deals that will included in this sector.
	2. Encode the deal data into existing replica using `newReplica[i] = Enc(sectorKey[i], data[i], EncodingRand(o))` function:
		- `newReplica` is the vector of `Fr` elements of the new replica
		- `sectorKey` is the vector of `Fr` elements of the CC sector replica data
		- `data` is the vector of `Fr` elements of the deals data ordered and padded as done in the sealing subroutine
		- `NodeSectorSize` is size of the sector in nodes (32 byte chunks)
	3. Compute SealedSectorCID of the `newReplica`.
4. **Storage Provider produces a proof of sector update.**
	Generate a SNARK that proves:
	1. Generation of ChallengesNumber challenges in batches of 8:
		1. For `j=0..ChallengesNumber//8`, `[_, c[8*j+7], c[8*j+6], ..., , c[8*j] = Split-31bit(Poseidon-128(Poseidon-128(UnsealedSectorCID, SealedSectorCID), j)`.
		The `Split-31bit` splits the 255bit output of Poseidon-128 hash into 8 31 bit chunks and discards 7 high bits.
		2. `c[i] = c[i] mod NodeSectorSize`
	2. For each challenge `i=0..ChallengesNumber`,  `c = c[i]`
		1. Encoding: the following we correcty computed: `newReplica[c] = Enc(sectorKey[c], data[c], Poseidon-128(UnsealedSectorCID, c*HashNumber//NodeSectorSize)`
		2. Inclusion proofs:
			1. `newReplica[c]` is the opening of `UpdatingSectorCID` at position `c`
			2. `sectorKey[c]` is the opening of `CommRLast` from `SealedSectorCID` at position `c`
			3. `data[c]` is the opening of `UnsealedSectorCID` at position `c`
5. **Storage Provider publishes `Miner.ReplicaUpdate(SectorID, NewSealedSectorCID, deals []DealID, Proof)`**:
   	1. Validate input parameters
	2. Activate Deals
	3. Generate `NewUnsealedSectorCID` (CommD) based on DealIDs and market actor state
	4. Verify the proof of correct data update:
		- Verify that `proof` is valid for using inputs: `NewSealedSectorCID`, `NewUnsealedSectorCID`, `SectorKey`.
	5. Update SectorInfo

### Algorithms 算法


#### Encoding Randomness 编码随机性

`EncodingRand(UnsealedSectorCID, i) = Poseidon-128(UnsealedSectorCID, i*HashShards//NodeSectorSize)`

Intuitively the sector nodes (32byte chunks) are split into **HashShards** number of continuous segments with
the same encoding randomness.

#### Encoding 编码

The current encoding algorithm is the following:

`Enc(sectorKey[i], data[i]) = sectorKey[i] + data[i] * EncodingRand(i)`

The Encoding function is cheap and allows for parallel encoding. 

#### Decoding 解码

`Dec(sectorKey[i], replica[i]) = (replica[i] - sectorKey[i]) * (EncodingRand(i)<sup>-1</sup>)`

The modular inverse of `EncodingRand` can be computed only **HashShards** times per whole operation.

### Retrieving updated data from encoded replica 从编码副本检索更新的数据

1. From the `replica`:
   2. Regenerate `sectorKey` by re-sealing the sector
   3. For each `i`: `Dec(sectorKey[i], replica[i], EncodingRand)`

### Deprecation of current CC Upgrade 当前CC升级的弃用

`PreCommitSector` will no longer be allowed to schedule current cc upgrades during prove commit.  The data on chain tracking cc upgrades and the miner actor logic doing current cc upgrades will be removed. PreCommit message parameters will no longer include fields for specifying cc upgrades.

在证明承诺的期间`预提示扇区`将不再被允许安排当前cc升级。跟踪cc升级的链上数据和进行当前cc升级的miner actor逻辑将被删除。预提交消息参数将不再包括用于指定cc升级的字段。

### ProveReplicaUpdates Actor Method 

#### Params 参数
```
type SnapDealsUpdate struct {
     SectorID SectorID
     Deadline int
     Partition int
     Deals []DealID
     Proof ReplicaUpdateProofs
}
```
`Miner.ProveReplicaUpdates(Updates []SnapDealsUpdate)`

#### Spec 规范
Unless stated assume that each step operates on all inputs updates in a loop

1. **Param validation**
   1. Assert that `SectorID` has no deals
   2. Assert proof size is not above limits
   3. Assert batch size bounds are not above limits
   4. Assert that sector's power is active. Fail if sector is in unproven, faulty or terminated states
2. **Market Activation and CommD generation**
   1. Call `market.ActivateDeals` on all deals in the update
   2. Call `market.ComputeDataCommittment` with batched inputs to generate all CommDs for all updates
3. **Proof verification**
   1. Extend runtime with VerifyReplicaUpdate method
4. **Update Miner State**
   1. `SectorNumber` i.e. the sector ID, `Expiration` and `SealProof` fields remain unchanged
   2. `SealedCID`, `DealIDs`, `DealWeight`, `VerifiedDealWeight` are all modified to match the upgraded sector with deals
   3. `InitialPledge` is the max of CC sector's InitialPledge and the value computed with the upgrade power at the upgrade epoch
   4. `ReplacedDayReward` is assigned the value of `ExpectedDayReward` and `ExpectedDayReward` is recalculated, as in current CC upgrade
   5. `ReplacedSectorAge` is set to `upgradeEpoch - Activation` as in current CC upgrade
   6. `ExpectedStoragePledge` is recalculated
   7. If `SectorKeyCID` is null, set it to `SealedCID` of the CC sector being replaced
   8. `Activation` is set to current epoch 
   9. Call `ReplaceSectors` on all partitions with updates
5. **Update Power**
   1. `partition.ReplaceSectors` will update miner deadline state to correct power
   2. Call `power.UpdateClaimedPower` to update power actor state.

#### Batching and Error Handling 批处理和错误处理

This method accepts a batch of updates to be processed in one message call.  As much as possible error handling should ignore updates that do not pass validation and only fail if all updates fail validation or if the entire operation fails.

此方法接受在一个消息调用中处理的一批更新。错误处理应尽可能忽略未通过验证的更新，并且仅在所有更新未通过验证或整个操作失败时才失败。

### Upgrade and State Migration 升级和状态迁移
1. Migration removing precommit info about cc upgrades in the PreCommit map
2. PreCommit params will lose the fours cc upgrade fields
3. Add `SectorKey` a new field to the sector info.  It should be a pointer to a cid serializing to cbor null when no snap deals upgrade has occurred and becoming the original `SealedCID` after the first snap deals update

## Design Rationale 设计原理

### Encoding Function 编码方法

The aim of the encoding function is to allow encoding data into an existing sector key without breaking
the compressibility assumption of PoRep.
...

### Parameters choice 参数选项

* `ChallengeNumber`: 1376 - to facilitate required security factor with Fiat-Shamir (TODO expand in Security Considerations).
* `HashShards`: 512 - to reach required spacegap fraction without neadlessly increasing the cost of conducting protocol (TODO explanation).

### Limitations 局限性

The update protocol is limited to CC sectors as the `SectorKey` commitment is only avaliable for sectors with no data. The SectorKey commitment is currently included as `CommRLast` commitment inside the on-chain `SealedSectorCID`  when the sector is CC. When the sector contains deals natively the `SealedSectorCID` includes only `CommRLast+Data` commitment.

更新协议仅限于CC扇区，因为`SectorKey`承诺仅适用于没有数据的扇区。当扇区为CC时，SectorKey承诺当前作为`CommRLast`承诺包含在链上`SealedSectorCID`中。当该扇区包含本地交易时，`SealedSectorCID`仅包括`CommRLast+Data`承诺。

## Backwards Compatibility 向后兼容性

This change is not backwards compatible and will require a new actors version and a network upgrade.

此更改不向后兼容，需要新的actors版本和网络升级。

## Test Cases

TODO

## Security Considerations 安全考虑

### Loosing epsilon-Replication 丢失ε复制证明

This section is a summary of more detailed [report](../resources/fip-0019/SnapDeals-Theory_Report.pdf).

本节总结了更详细的[报告](../resources/fip-0019/SnapDeals-Theory_report.pdf)。

**Assumption 1**: We work under the assumption that for each bit grinded on the `{\rhoi}i=1,...,N`  an adversary can flip one bit of her choice on the original `R`.

**假设1**：我们工作的假设是，对于在`{\rhoi}i=1,...,N`上研磨的每一位，对手可以在原始`R`上翻转其选择的一位。

We consider `\lambda = 80` as the security parameter. This means that we give the adversary a bit flip budget of 80 bits on `R` (by crafting `R*` such that these bits are flipped, keeping all the rest untouched) in order to make `R*` more compressible than `R`. 

我们将`\lambda = 80`作为安全参数。这意味着我们给对手一个`R`上80位的位翻转预算（通过制作`R*`，使这些位被翻转，其余的保持不变），以使`R*`比`R`更可压缩。

Observation: As we will see in what follows, in the case of our actual encoding (where a single `\rhoi` is shared among _K_ distinct nodes Dj), we'll be either more conservative than we should, according to Kolmogorov bounds.

观察：正如我们将在下文中看到的，在实际编码的情况下(在_K_个不同节点Dj之间共享一个`\rhoi`)，根据科尔莫戈罗夫边界，我们要么比我们应该更保守。

In order to stay on the safe side, we have two independent security analysis, following two different paths that give us similar results. 

为了安全起见，我们有两个独立的安全分析，遵循两条不同的路径，得出了相似的结果。

The first analysis is a probabilistic one, and we refer to it as the "security against the flipping adversary". The second one is using an information theoretical argument based on Kolmogorov complexity.

第一种分析是概率分析，我们将其称为“对抗翻转对手的安全”。第二种是基于Kolmogorov复杂性的信息理论论证。


#### Analysis 1: Security against "2 bytes for 1 bit" Adversary 

**Preliminaries:**
Given a predetermined string of L bits, we take for granted that a random string has probability 2<sup>-L</sup> to contain it.
We will consider bit strings of K_consecutive `0`s (or `1`s). Note that it is the same when considering patterns like `01` or `10`, the only difference is that those sequences would allow for less compression.

**预备工作：**
给定一个L位的预定字符串，我们认为随机字符串包含它的概率为2<sup>-L</sup>。
我们将考虑K_consecutive `0`(或`1`)这样的位字符串时。注意像`01`或者`10`这是相同的，唯一的区别是这些序列将允许更少的压缩。

**Assumption 2**: An adversary can compress ~2 consecutive bytes (we would consider 2 bytes per node, but it can happen anywhere) by flipping a single bit.
This means having substrings s that are of the form  `s = 0000000100000000` (or its negation).
Note that the probability of `s` (or negation of `s`) to happen is 2<sup>-15</sup>. 
Since the adversary can adaptively choose the bits to flip, we are just assuming he finds 80 of these strings in a 32GiB string. 2 bytes out of 32 represent the 6.25% of a single node

**假设2**：对手可以通过翻转一个位来压缩~2个连续字节（我们会考虑每个节点2个字节，但它可以发生在任何地方）。
这意味着具有形式为`s = 000000100000000`（或其否定）的子字符串's'。
注意，`s`（或`s`的否定）发生的概率为2<sup>-15</sup>。
由于对手可以自适应地选择要翻转的位，我们假设他在32GiB字符串中找到了80个这些字符串。32个字节中的2个字节表示单个节点的6.25%

Note 1: Remember that each randomness `\rhoi` is shared among K nodes, have 2<sup>30</sup>/K different hashes.
As a consequence, we consider each compression performed on a node D to propagate on all the K nodes that share the same randomness with D.

注1：请记住，每个随机性`\rhoi`在K个节点之间共享，具有2个＜sup＞30个</sup>/K个不同的散列。
因此，我们考虑在节点D上执行的每个压缩在与D共享相同随机性的所有K个节点上传播。

**Assumption 3**: An adversary A is able to find a string of the form of `s = 000000010000000` (or equivalent) in 80 different nodes, each of those uses a different `\rhoi`

**假设3**：对手A能够在80个不同的节点中找到`s = 000000010000000`（或等效值）形式的字符串，每个节点使用不同的`\rhoi`

Note 2: Assumption 3 maximizes the propagation of the compression made by the adversary since all the local compression propagates to K nodes each. This results in a total compression of 80*0.0625*K nodes out of 2<sup>30</sup>

注2：假设3最大化了对手进行的压缩传播，因为所有本地压缩都传播到每个节点的K个节点。这导致总共压缩了80*0.0625*K个节点（共2个＜sup＞30个</sup＞节点）

With H = 2<sup>9</sup> and K = 2<sup>21</sup> we obtain a total compression under 1%.

当H = 2<sup>9</sup>和K = 2<sup>21</sup>时，我们得到的总压缩率低于1%。

Assume we have 2<sup>9</sup> different hashes. This means that we have a total compression, according to Assumption 2 and Assumption 3 of ~1%.

假设我们有两个不同的散列。这意味着，根据假设2和假设3，我们的总压缩率约为1%。

#### Analysis 2: Based on Kolmogorov Complexity (Information Theoretic Argument)
**Information Theoretic Argument**: Given a string `S` of size N with incompressibility V, for any adversary that modifies `S` into `S'` by flipping T bits of his choice, we have that the incompressibility of `S'` is at least _V - T*(log(N) + log(log(N))+1)_.
**Proof**:  Assume by contradiction that there exists a string `S''` which is more compressible than `S'`. 
Then it means that it is possible to compress `S` into a string `S*` of length  _| S'' | + T*(log(N) + 2*log(log(N))+1) < V_ .

Note: _T*(log(N) + 2*log(log(N))+1)_ are the bits needed in order to be able to store the T bit positions where the adversary flipped the bits, using self delimiting codes (References: this [paper](https://eprint.iacr.org/2021/162.pdf), page 9; this [book](https://core.ac.uk/download/pdf/301633813.pdf), Section 2.2).

Given the following:
We assume `R` to be incompressible, and we then set the incompressibility of `R` to `N`. 
We assume an adversary can flip 80 bits of his choice in `R`.
To be conservative, given that we are considering 2<sup>30</sup>/K different hashes, we apply the compressibility lower bound on a vector of size N' = 2<sup>30</sup>/K nodes

According to the lower bound, we get that an adversary that can flip 80 bits of his choice on `R` can not compress `R` to a string that has compressibility smaller than _N' - 80(log(N') + 2*log(log(N')) + 1)_. 

In order to be even more conservative, we are using `N` instead of `N'` in the compressibility reduction factor. We are then considering that R can not be compressed to a string which is less compressible than 2<sup>30</sup>/K  - 80(log(2<sup>30</sup>) + log(log(2<sup>30</sup>)) + 1) = 2<sup>30</sup>/ K - 80(30+11) .

As a consequence, we have that an adversary can compress a total of `80(30+11)*K` bits out of 2<sup>30</sup>.

This means that, for K = 2<sup>21</sup> and 2<sup>9</sup> = 512 different hashes, we have that the additional spacegap is < 2.51%. Given in our case the distribution of the last bit of each node is not uniform, we also considered the case of nodes of 255 bits rather than 256. Spacegap in this case is < 2.5123%.
If we allow for 1024 hashes, additional spacegap is bounded by < 1.26%. If as above we consider nodes of bit length 255 instead of 256 we still obtain that an additional spacegap bounded by  < 1.26%  


### Poseidon-128

Poseidon-128 is zk-SNARK friendly hash function which is already widely used in Filecoin protocol.
Both uses of Poseidon-128 include Domain Separation Tags such that the usage here does not clash with possible future use cases.

Domain Separation Tags used:
 - Encoding Randomness Generation: `2^40`
 - Challenge Generation: `2^40`

Domain Separate Tags for both are the same as their inputs are different.

## Incentive Considerations

SnapDeals does not allow storage providers to release currently held collateral for the sector but it allows
to reuse that already held collateral if collateral required is greater, for example due to verified deal.

## Product Considerations

Snap Deals protocol significantly reduces the time needed for data to be included in a sector
and confirmed on-chian. The process was designed with cost for storage providers in mind.

## Implementation

## Future work

* DeclareDeals to support deal transfer: allow moving deals from sector to sector
* CapacityDeals: allow purchasing a capacity of the storage of a storage provider instead of the storage of a specific deal
* Update protocol that does not require to perform an operation on a full sector.
* DealUpdates: a license to terminate a deal in place for a new one (e.g. a user wants to update a portion of their files)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
