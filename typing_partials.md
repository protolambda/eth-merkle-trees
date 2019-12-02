# Typing and Partials

One of the main reasons for SSZ is the typed structured convenience over complex merkleized state.
Issues have been raised about the `BeaconState` "god object": a state so large that it looks ridiculous to use.
However, *this is merely typing*: the real state can be represented as a binary tree,
 making each modification and new hash-tree-root cost only `O(log(N))`, and so can ligth-clients keep track of subsets, or partials, of the state.

And then the `BeaconState`, and future phase 1+ parts of the state, are optimized so that the tree does not change by too widely: this is optimal for cached hash-tree-roots and light-client proofs.

## Problem

The difficult part here is usability: the tree is supposed to be immutable, and can have any unbalanced shape. Yet we want to mutate the state, and expect properties of a typed structure to be there.

To handle this, **the "backing" is separated from the typing layer**:
unlike existing SSZ implementations that run hash-tree-root through their typed language-specific structure,
 it runs on a bare binary merkle tree to preserve the immutability and caching ease.

The typing is then applied by overlaying a view on the backing: views track the backing anchor node, and when a view mutates, its rebinds to the resulting new backing root node.
This enables you to fork the backing into many different states, and only pay the cost for the modified parts. And this applies to both storage as hashing cost.

## Tree

### Structure

The tree part structure is fairly simple:
- `Node`: interface: a node in the binary merkle tree.
    - `hash_tree_root()` to get the merkle root of this node. 
- `Root`: a single `bytes32`, no child nodes.
    - `hash_tree_root()` returns the node itself.
- `Commit`: a pair of a left and right `Node`.
    - `hash_tree_root()`, simply `H(left.hash_tree_root(), right.hash_tree_root())`. And cache the resulting root in the `Commit`, to never do the work over again.

### Merkleization "`hash_tree_root()`"

- `Root`: returns the node itself.
- `Commit`: simply `H(left.hash_tree_root(), right.hash_tree_root())`. And cache the resulting root in the `Commit`, to never do the work over again.


### Navigation, "`getter(target: gindex) -> Node`"

Notes:
- `anchor = 1 << (gindex.bit_length() - 1)`: the generalized index, but with everything zeroed except the leading `1`.
- `pivot = anchor >> 1`: the anchor of the left subtree.
- `target ^ anchor | pivot`: very common operation, but note that it only changes the first two bits: it moves the leading bit one down, to navigate left or right subtree.

Implementation:
- `Root`: the node itself if `target == 1`, error otherwise.
- `Commit`:
    - `target < 1`: error
    - `target == 1`: return self
    - `target == 2`: return left child
    - `target == 3`: return right child
    - `target < pivot`: return `left.getter(target ^ anchor | pivot)`
    - `target >= pivot`: return `right.getter(target ^ anchor | pivot)`

`target == 2, target == 3` are optional, but save a little bit of time.

Now to get a node `p` at generalized index position `x`: `p = tree_root_node.getter(x)`

### Facilitating mutations, "`setter(target: gindex) -> Link`"

To change the tree, which is immutable, we need to introduce *"rebinding"*: creating new commits that partially reference the unchanged child subtree, and partially whatever new node is inserted.

Rebinding functions are `Link`: they take an input, and when called with a value, they return the new tree anchor that reference this value in its newly linked together position,
 re-using the unmodified subtrees under the previous anchor.
 
A `Commit` allows you to rebind with two different `Link` functions:
- `rebind_left(v)`: `Commit(v, self.right)`
- `rebind_right(v)`: `Commit(self.left, v)`

A `Node` can always be rebound with `identity`: simply a `Link` that returns the input as output.

To rebind a node to a specific position, a `Link` is composed of smaller links leading from the entry anchor to the position:

`compose(inner: Link, outer: Link) -> Link: return lambda v: outer(inner(v))`

Implementation:
 - `Root`: `identity` if `target == 1`, error otherwise.
 - `Commit`:
     - `target < 1`: error
     - `target == 1`: return `identity`
     - `target == 2`: return `rebind_left`
     - `target == 3`: return `rebind_right`
     - `target < pivot`: return `compose(left.setter(target ^ anchor | pivot), rebind_left)`
     - `target >= pivot`: return `compose(right.setter(target ^ anchor | pivot), rebind_right)`

Again, `target == 2, target == 3` are optional but save some time.

Now, to set a node `q` at generalized index position `x`:
```
setter_x = tree_root_node.setter(x)
new_tree_root_node = setter_x(q)
```

### Expansions as setters, `expand_into(target: gindex) -> Link`

Another more advanced feature is to expand a tree.
This could be done to expand SSZ summaries, but is also essential to implement `append/pop` of lists:

It's exactly like `setter` (but call `expand_into` for `left`/`right`), except for the  `Root`:
- `if target == 1`: return `identity`
- `else`:
  - `child = zero_node(target.bit_length() - 1)`
  - return `Commit(child, child).expand_into(target)`

The idea here is that instead of a `Root` being a dead end, a temporary commit with zeroes is made (of the 1 below the `target` anchor depth).
Either the `left` or `right` will be replaced by calling the returned `Link`, and the new commit is bound into the new tree anchor.

#### Expansions with proofs

To expand a node into zeroes, mismatching the possible previous value,
 works great for `append/pop` to implement a `List` with, but not for expanding summary nodes.
Instead, we can define a `expand_from_proof(target, branch)`,
 now passing along a `branch` with the target, again only changing behavior of `Root`:
- `if target == 1`: return `identity`
- `else`:
  - `child = branch[0]`
  - `inner_link = Commit(child, child).expand_from_proof(target, branch[1:])`
  - ```python
    def secure_link(v: Node):
        out_node = inner_link(v)
        assert out_node.hash_tree_root() == self # we are operating on a summary Root node
        return out_node
    ```

Again, either the `left` or `right` will be replaced, but this time a proof is supplied to make it hash to the same summary root.


### Factories

Often you will want to create subtrees filled with commits, and partially empty (e.g. a list with a huge limit, but only a few elements), or efficiently filled with a default (e.g. super large vector).
One of the benefits of the immutable approach is that it works really well with defaults: subtree filled with millions of the same item can be represented by a single `O(log(N))` branch of nodes!
This is similar to the `zero_hashes`: a `x_{i+1} = H(x_i, x_i)` does the job.

```python
def subtree_fill_to_depth(bottom: Node, depth: int) -> Node:
    node = bottom
    while depth > 0:
        node = Commit(node, node)
        depth -= 1
    return node
```

```python
def subtree_fill_to_length(bottom: Node, depth: int, length: int) -> Node:
    if length > (1 << depth): raise Exception("too many nodes")
    if length == (1 << depth): return subtree_fill_to_depth(bottom, depth)
    if depth == 0:
        if length == 1: return bottom
        else: raise NavigationError
    if depth == 1: return Commit(bottom, bottom if length > 1 else zero_node(0))
    else:
        anchor = 1 << depth
        pivot = anchor >> 1
        if length <= pivot:
            return Commit(subtree_fill_to_length(bottom, depth-1, length), zero_node(depth))
        else:
            return Commit(
                subtree_fill_to_depth(bottom, depth-1),
                subtree_fill_to_length(bottom, depth-1, length - pivot)
            )
```

```python
def subtree_fill_to_contents(nodes: List[Node], depth: int) -> Node:
    if len(nodes) > (1 << depth): raise Exception("too many nodes")
    if depth == 0:
        if len(nodes) == 1: return nodes[0]
        else: raise NavigationError
    if depth == 1: return Commit(nodes[0], nodes[1] if len(nodes) > 1 else zero_node(0))
    else:
        anchor = 1 << depth
        pivot = anchor >> 1
        if len(nodes) <= pivot:
            return Commit(subtree_fill_to_contents(nodes, depth-1), zero_node(depth))
        else:
            return Commit(
                subtree_fill_to_contents(nodes[:pivot], depth-1),
                subtree_fill_to_contents(nodes[pivot:], depth-1)
            )
```

