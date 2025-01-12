---
fip: "0007"
title: h/amt-v3
author: Rod Vagg (@rvagg), Steven Allen (@Stebalien), Alex North (@anorth), Zen Ground0 (@Zenground0)
discussions-to: https://github.com/filecoin-project/FIPs/issues/38
status: Final
type: Technical
category: Core
created: 2020-11-27
specs-sections:
   - section-appendix.data_structures.hamt

---

## Simple Summary 简述
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->

Improve the filecoin HAMT and AMT in terms of performance and safety with three smaller independent proposals.

通过三个较小的独立提案，提高filecoin HAMT和AMT的性能和安全性。

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
The filecoin HAMT does unnecessary Puts and Gets while doing Set and Find operations. Sub proposal `A` eliminates these unnecessary operations improving performance of HAMT calls.

The filecoin HAMT's pointer serialization can be modified to save 3 bytes and a small amount of de/encode time. Sub proposal `B` describes the more efficient serialization.

The filecoin AMT does unnecessary Puts and Gets while doing Set, Find and ForEach operations. Sub proposal `C` eliminates these unnecessary operations improving performance of AMT calls.

## Change Motivation

These are all unambiguous performance wins. We aren't trading anything off we are strictly reducing ipld operations and serialization size which also corresponds to reduced gas cost.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

### A
The existing HAMT loads child tree nodes into in memory cache nodes located on HAMT "link" pointers (not KV pointers). It also clears these cache nodes after writing them to disk during the Flush operation. Finally the current HAMT code writes to the blockstore when it creates new node during bucket overflow as part of setting new values.

This proposal is to (1) track modification of nodes and only Flush these caches to the blockstore when the cache node has been modified (2) stop clearing cache nodes on Flush (3) Move all blockstore writes to occur on Flush.

Concretely (1) can be achieved by adding a `dirty` flag as a sibling to the cache node on the HAMT pointer. The flag defaults to false when a cache node is loaded for reading and set to true if a set or delete operation traverses through that node and modifies that node or one of its children. Then during the Flush traversal, a cache node is written to the blockstore and its owning Pointer's Link field assigned to the cid of cache node only if the dirty flag is true at the time of Flush. Otherwise the cache node is skipped.

To achieve (2) implementations only need to stop clearing the cache nodes after flushing pointers during Flush.

Finally for (3) implementations can simply add a new cache node when creating a new child node during bucket overflow and mark this node as dirty so that it gets written during the next flush.

See go-hamt-ipld implementation [here](https://github.com/filecoin-project/go-hamt-ipld/pull/74) for a concrete example. Note that the part of this specification that avoids marking nodes as dirty during Set operations that overwrite an unchanged value to a key is being implemented separately [here](https://github.com/filecoin-project/go-hamt-ipld/pull/77#discussion_r532044992).

### B
HAMT pointers are union types. A pointer can be one of a link to a HAMT node or a KV bucket containing HAMT key value pairs. To achieve this pointers are currently serialized as a cbor map with at most one key. A key of "0" means link, a key of "1" means bucket.

This proposal is to only serialize the link or bucket of kvs and use the different cbor major types (tag major type or array major type) to distinguish which type populates the union on decode.

Succinctly from @rvagg, this proposal is to change from:
```
type Pointer union {
	&Node "0"
	Bucket "1"
} representation keyed
```
i.e. `{"0": CID}` or `{"1": [KV...]}`

to:
```
type Pointer union {
	&Node link
	Bucket list
} representation kinded
```
i.e. `CID` or `[KV...]`

See @rvagg's original proposal in go-hamt-ipld [here](https://github.com/filecoin-project/go-hamt-ipld/issues/53#issue-663418352) and the open PR implementing this in go-hamt-ipld [here](https://github.com/filecoin-project/go-hamt-ipld/pull/60) for more information.

### C
This proposal is analagous to sub proposal A except on the AMT instead of the HAMT. Implementation is [here](https://github.com/filecoin-project/go-amt-ipld/pull/30). Additionally the proposal includes rearranging order of load during `ForEach` traversals that do not walk all keys to only load child nodes if they contain keys within the traversal range. See implementation [here](https://github.com/filecoin-project/go-amt-ipld/pull/37/files) for full details.

## Design Rationale

These designs follow directly from understanding the inefficiencies of the current HAMT. They have only not been introduced sooner because they are breaking changes and carry implementation risk.

## Backwards Compatibility

### A & C
These changes are consensus breaking because they reduces the gas cost of HAMT Set and Find operations and AMT Set, Find and ForEach operations . So running HAMT and AMT code with this feature will not result in the same state transitions as in the existing protocol. Therefore this change requires a network version update.

### B
This change requires modifying the serialized bytes of all HAMT nodes. This will require a state tree migration migrating all HAMT data.

## Test Cases

Test cases are / will be accompanying implementation

## Security Considerations
All significant implementation changes carry risk. On their own each sub proposal is a simple well understood change. Increased HAMT tests as well as existing validation practices are expected to be sufficient safeguards. 

## Incentive Considerations
On the whole this change should directly and indirectly decrease both real validation time and state storage cost as well as gas cost while otherwise maintaining the same behavior. It should be incentive compatible to accept this change for all parties in the network. It should not introduce changes to the protocol's incentive structure.

## Product Considerations
This change should directly and indirectly decrease the costs of state machine operations. This will not change any aspect of the network as a product other than the positive impacts of improved performance of existing behavior.

## Implementation
Sub proposal A implemented and optimistically merged to go-hamt-ipld [here](https://github.com/filecoin-project/go-hamt-ipld/pull/74)

Sub proposal B implemented and optimistically merged in go-hamt-ipld [here](https://github.com/filecoin-project/go-hamt-ipld/pull/60)

Sub proposal C implemented and optimistically merged in go-amt-ipld [here](https://github.com/filecoin-project/go-amt-ipld/pull/30) and [here](https://github.com/filecoin-project/go-amt-ipld/pull/37)
