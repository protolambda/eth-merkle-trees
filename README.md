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

### Tree-offsets idea

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

Opportunities:
- SSZ merkle trees are already binary. MPT can be translated from hexary to binary.
- SSZ Bottom nodes are always pairs, since we have both leaves and witnesses in a proof.
- SSZ merke paths are well established as bitlists, or "generalized indices"
- MPT paths are a static depth, but start to differ from siblings after some smaller prefix length.
- Binary unbalanced tree reasoning: parent size = left size + right size. When the parent is known, only one child size needs to be encoded to express the other.
- Consistent encoding + consistent paths -> consistent and elegant lookups.
- SSZ bottom nodes have a static size (32 bytes per chunk), and so offsets can be translated easily between the offsets indexing, and the proof data indexing.

#### Tree offsets

##### Definition

```
<offsets><bottom chunks>
```

**`<offsets>`**: First encode the total number of bottom nodes,
then encode the size (count of bottom nodes) of the left-hand subtree,
starting from the root, going depth-first through the tree.
Bottom nodes themselves do not have child nodes, so no left-hand size is encoded for these.

When reducing the set of bottom nodes by factor 2 until 1 is left,
 which holds for any unbalanced tree shape, you spent `N - 1` operations.
So `N - 1` offsets. add `1` field for the total number of offsets, and it results in a nice exact `N` total to encode.
For reasonable proof size the size of an offset could be 2 bytes (`2**16 = 65536` bottom nodes in the proof. The real tree can be a lot bigger however)
This would be a `2 / 32` = `6.25%` memory overhead, to both deterministically describe the proof and provide an efficient lookup.

**`<bottom chunks>`**: The bottom nodes in the tree packed together, each as a 32 byte chunk, from left to right, 
with witnesses and leaves mixed (there is really no notion of a difference here).

##### Lookup

To get the buffer position of a bottom node chunk with a given generalized index, run through the offsets.  
```
inputs: offsets, g_index
# skip the leading root bit.
i = 1
# skip the offsets length encoding.
pos = 1
res = 0
for i < len(g_index):
    if g_index[i]:
        skip = offsets[pos]
        res += skip
        pos += skip
    else:
        pos += 1
    i += 1
return res
```

`chunks[res]` would be the lookup result. Or `buffer[offsets[0] << BYTES_PER_OFFSET + res << BYTES_PER_CHUNK]` to get the chunk from the raw data.

##### Partials

A partial could then modify the chunk data in-place. And after a set of modifications, the proof chunks can be re-hashed for a new root.

Appending/popping to subtrees would result in new chunks. This would incur a `depth` increments/decrements to update the offsets, and one memory copy to move the chunk bytes.

Note that appending/popping can be trivially batched.

#### Typed trees

For some cases, it is known to only have 1 type of element, e.g. a transaction tree.
In this case one could not encode the duplicate offsets for each of these transactions, and effectively change `BYTES_PER_CHUNK` to `BYTES_PER_BOTTOM_TYPE`.

#### Multi-proof verification

To run a multi-proof efficiently, one would want to:

- Maintain a stack of subtree roots to merge sibling subtree roots with.
- Navigate the tree efficiently. No stack-frames or memory copies.
- Combine pre-determined hops (static slices within the generalized index bitlist):
 flatten the loop (based on static generalized index size), and sum changes by known consecutive `g_index[i]` bits.
 (This could result in a 100% compile time lookup optimization for static structures!)

Multi-proof verification based on offsets and a packed bottom-chunks array:

```
chunk_i = 0
pos = 0
size_stack = [0] * max_depth
chunk_stack = [chunk(0)] * max_depth
stack_i = 0
for pos < count:
    if offsets[pos] > 1: # if the sub-tree is not empty, we need to go deeper. (also handles empty proof case)
        size_stack[stack_i] = offsets[pos]
        stack_i += 1
        pos += 1
    else:
        assert offsets[pos] != 0  # no empty offsets, all pairs should have a left-hand.

        # hit a botom chunk on left hand. Put it into the stack
        chunk_stack[stack_i] = chunks[chunk_i]


// TODO: merging
        merge_chunk = chunk[pos]
        chunk_i += 1

        start_stack_i = stack_i
        # one branch node on right-hand left to merge? -> merge back up as far as possible
        for size_stack[stack_i] + 1 == size_stack[stack_i - 1]
            # when done with a subtree, make the parent get closer to completion.
            size_stack[stack_i - 1] += size_stack[stack_i] + 1
            # merge right-hand bottom chunk back up with left-hand from stack
            merge_chunk = hash(chunk_stack[stack_i] ++ merge_chunk)
            stack_i -= 1
        
        # save merge results if any
        if start_stack_i != stack_i:
            chunk_stack[stack_i - 1] = merge_chunk
```

Stacks are allocated with space for max depth, e.g. `chunk_stack` is 128 chunks for a tree with the lowest node at depth 127. If unknown, use a generous margin.

// TODO: work in progress multi-proof code.