# Experiments with merkle trees


## Problems

- [Binary Multi-merkle-proofs](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/light_client/merkle_proofs.md) can be:
    - Very unbalanced; e.g. a light-client beacon-state subset, along with some sparse historic data.
    - Shaped by run-time; e.g. transaction batching, or merkleized events.
    - Difficult to read with good performance:
        - When leaves and witnesses are split (status quo), the verification target needs to be known to make the difference in hashing.
        - Unbalanced trees result in either:
           - (very) sparse but balanced encoded proof data
           - packed but unbalanced proof data; hard to locate the leaf nodes from a packed structure.
              - reading and reasoning about the tree from left to right is nice, but not when you cannot do so efficiently (i.e. skip ignored subtrees).
    - Hard to describe their shape. See work by Alexey on merkle-patricia-trees, with the best encoding solution so far being a new instruction set to describe the tree...
        - Now consider that values can have any unbalanced position in a binary tree, determined during runtime based on complicated indices (e.g. an item in a list in a list).
        - The depth of the tree makes generalized indices bigger. So unbalanced trees encode very poorly as packed generalized indices (we want to avoid encoding generalized indices completely).
- Eth2 EWASM does not force one storage/memory abstraction like the EVM. An EE with SSZ fundamentals can be a developed with the right optimizations:  
    - Storing SSZ trees efficiently
    - Cheap verification of proofs, one-go multi-proof verification for batched transfers.
    - An efficient interface to read proofed data can be the base-layer for SSZ-partials in an EWASM context.
        - Modification in-place could be relatively cheap. Appending/popping can be more costly.
    - Could implement the Eth2 spec itself, fun for nesting experiments.
    - Can be nice to work with consensus data.
- Merkle-patricia-trees need to be revisited:
    - Optimize MPT performance of Go-ethereum. Current storage is slow and inefficient.
      Turbo-geth is one way to solve it, but the solution path of the other problems may be translated to another new approach.
    - One of the essential building blocks for putting Eth1 in an Eth2 EE.
 
## Towards a solution

SSZ already knows offsets for encoding variable length data, thanks to @karalabe for the initial
 [SOS](https://gist.github.com/karalabe/3a25832b1413ee98daad9f0c47be3632) and Piper for [PR 787](https://github.com/ethereum/eth2.0-specs/pull/787).

Now these offsets are for looking up **naturally sequential** data **by flat index**.
We are trying to lookup **forced  sequential** data **by generalized index**.
And since the data is unbalanced, the regular logarithmic structure does not do the job (`|0|1|10,11|100,101,110,111|....`).
Also note that for SSZ proofs, bottom nodes are always 32 bytes. Consistency is key to make lookups efficient and easy, as opposed to the Merkle Patricia Tree specializations.
But subtrees are not always full, so variable size.

### Tree-offsets

This is a design idea, not a proposal. It can be applied anywhere.
It is undecided on what to encode into messages, and what to derive during compile time or run time.

Goals:
- Lookup any bottom node in `O(depth of node)` operations.
- Describe the tree deterministically
- Scale the size of this data linearly with the amount of bottom-nodes.
- Back a SSZ-partial with a multi-proof
- Forget about the witness-leave difference. It seems nice, but breaks a boundary between application level (what meaning a proof has) and system level (how the proof is structured and verified).
- Elegant lookups
- Sequential lookup steps should be easy to combine if known during compile time.

WIP -- tree-offsets write up soon :)

