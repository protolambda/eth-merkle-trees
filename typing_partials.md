# Typing and Partials

One of the main reasons for SSZ is the typed structured convenience over complex merkleized state.
Issues have been raised about the `BeaconState` "god object": a state so large that it looks ridiculous to use.
However, *this is merely typing*: the real state can be represented as a binary tree,
 making each modification and new hash-tree-root cost only `O(log(N))`, and so can ligth-clients keep track of subsets, or partials, of the state.

And then the `BeaconState`, and future phase 1+ parts of the state, are optimized so that the tree does not change too widely (lots of elements at the same time): this is optimal for cached hash-tree-roots and light-client proofs.

## Problem

The difficult part here is usability: the tree is supposed to be immutable, and can have any unbalanced shape. Yet we want to mutate the state, and expect properties of a typed structure to be there.

To handle this, **the "backing" is separated from the typing layer**:
unlike existing SSZ implementations that run hash-tree-root through their typed language-specific structure,
 it runs on a bare binary merkle tree to preserve the immutability and caching ease.

The typing is then applied by overlaying a view on the backing: views track the backing anchor node, and when a view mutates, its rebinds to the resulting new backing root node.
This enables you to fork the backing into many different states, and only pay the cost for the modified parts. And this applies to both storage as hashing cost.

Note that this is not new, libraries like [`pyrsistent`](https://pyrsistent.readthedocs.io/en/latest/) already implement this kind of mutable-like interface over immutable tree-based backings.
The difference here is that **it does not have to be "magic"**: we can implement it ourselves, and bake in good support for:
- binary trees, with navigation. Your favorite data structure by the end of this read.
- merkle caching by default, `O(loh(N))` hashing cost, nevermind SSZ performance.
- types are compatible with partial trees, run things on merkle proofs.
- ownership of the backing tree, manage and store the tree, 
 e.g. fork an older state into a new state with a few modifications, with bare minimum cost.


## Tree

### Structure

The tree part structure is fairly simple:
- `Node`: interface: a node in the binary merkle tree.
- `Root`: a single `bytes32`, no child nodes.
- `Commit`: a pair of a left and right `Node`.

### Merkleization "`hash_tree_root()`"

- `Root`: returns the node itself.
- `Commit`: simply `H(left.hash_tree_root(), right.hash_tree_root())`. And cache the resulting root in the `Commit`, to never do the work over again.


### Navigation, "`getter(target: gindex) -> Node`"

Notes:
- `anchor = 1 << (gindex.bit_length() - 1)`: the generalized index, but with everything zeroed except the leading `1`.
- `pivot = anchor >> 1`: the anchor of the left subtree.
- `target ^ anchor | pivot`: very common operation, but note that it only changes the first two bits: it moves the leading bit one down, to navigate left or right subtree.
- `target < target | pivot`: check if the target is in the left subtree

Implementation:
- `Root`: the node itself if `target == 1`, error otherwise.
- `Commit`:
    - `target < 1`: error
    - `target == 1`: return self
    - `target == 2`: return left child
    - `target == 3`: return right child
    - `target < target | pivot`: return `left.getter(target ^ anchor | pivot)`
    - `target >= target | pivot`: return `right.getter(target ^ anchor | pivot)`

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
     - `target < target | pivot`: return `compose(left.setter(target ^ anchor | pivot), rebind_left)`
     - `target >= target | pivot`: return `compose(right.setter(target ^ anchor | pivot), rebind_right)`

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

## Views

Now the interesting part: how do we overlay a type on an immutable binary tree and make it behave like any regular mutable object?
The answer: views!

Views are ephemeral objects that reference a backing node, and provide an interface to *modify their reference, not the backing itself*.
Some views are simple, others require elaborate type information.

### Type definitions

Meta-programming for type definitions is really nice, but not strictly necessary.
E.g. a Golang implementation can define a type definition as yet another struct/slice type, and embed it into a view struct to provide the necessary type parameters.

In Python things can be more elegant:
- a `TypeDef` is is a `type` subclass.
- Then a `TypeBase` has `TypeDef` as metaclass, and can be subclassed to implement types.
- A `View` has `TypeBase` as metaclass, and can be subclassed to implement views.

 E.g. `Uint64` is an `int` and `View`, its class is described by a `Uint64Type`,
  which is a specialized `UintType`/`BasicType`/`TypeBase`. And this `Uint64Type` is a `TypeDef` that can be passed around and required by other composite type definitions to e.g. declare an element type.

#### Interface

Type definitions should provide:
- `default_node() -> Node`: create the default backing for this type.
- `view_from_backing(node: Node) -> View`: wrap a backing in a type-specific view. 

And then logically follows: `def default() -> View: return view_from_backing(default_node())`

And then views should provide:
- `get_backing() -> Node`
- `set_backing(node: Node)`
But these can be internal to the view implementation instead of fully exposed.

#### View-hooks

The pass-by-value vs pass-by-reference can be translated to views as well:

A getter of a view (the "superview") returns a new view (the "subview"), with its own subtree-backing:
- pass-by-value: detached from the superview, not propagating changes
- pass-by-reference: attached to the superview, it has a `ViewHook` to call to propagate changes.

A `ViewHook` is a lot like a `Link`, but the mutable typed version: `ViewHook = function(v View)`
So whenever a `View` changes its backing (in `set_backing(node)`), it should call `hook(new_backing)` to make the superview aware of the change of the element.
This `hook` can simply be created on the `get(key)` call on the superview to map it to a `set(key, v)` call: `lambda v: self.set(key, v)`

To pass by reference, the `view_from_backing` is extended:
- `view_from_backing(node: Node, hook: ViewHook) -> View`: wrap a backing in a type-specific view, setup to propagate changes to the given `hook`. 

Note however that putting the same view in multiple superviews will not work: the hook only updates one superview.
Alternatively you compose the hook to call multiple hooks, but at some point it is a better idea to simply work with copies of the view:
 mutability is nice, but unintended side-effects are not! Mutating a copy of the view (with different/no viewhook) will not affect the original.  
Not that view-copies are very efficient: all data is in the referenced immutable tree, the view is just a typed pointer.
Views are ephemeral, so embedding a view in multiple superviews shouldn't be possible.

### View implementations

#### Subtrees

With e.g. a `VectorType` a `Vector` (`VectorView`) can be described.
Again, in Golang simply embed the `VectorType` information into the `VectorView` instance, in Python `type(VectorView) == VectorType`.

Now one thing that stands out is that a lot of the types are very similar:
*a sequential series of elements in subtree of some fixed depth*.

Defining a `SubtreeType` and `SubtreeView` to generalize this is useful:

`SubtreeType`:
 - `depth`: the type is parametrized with a tree-depth.

`SubtreeView`:
 - `get(i: int) -> View`:
 ```python
    elem_type = ...
    elem_type.view_from_backing(get_backing().getter(to_gindex(i, depth)),
                      hook=lambda v: self.set(i, v))
 ```
 - `set(i: int, v: View)`:
 ```python
    setter_link = self.get_backing().setter(to_gindex(i, depth))
    set_backing(setter_link(v.get_backing()))
 ```

Now a `VectorType` can simply define `depth` as a function of `length`,
 and `VectorView` can limit `get` and `set` to `length` bounds in the subtree.

Similarly, a `Container` would be exactly the same, just with `len(fields)` as `length` and `elem_type` mapped to a `field_type`.

`List` could also be implemented on a `SubtreeView`: derive `depth` from `limit`, add `1` for the length mix in, limit `get`/`set` based on `length`, which reads the mix-in node contents (and optionally also checks against the `limit`).

##### `append` and `pop`

`List` is extra interesting since it also uses expansions naturally:
- `append` expands into the item at element index `length` of the list. (if not already at `limit`)
- `pop` summarises the item at element index `length - 1` of the list. (if not already empty)

Expansions and summaries here work on zero nodes. 
Instead of summarising, just setting it to a zero node is also valid if space is not an issue. 

And then do not forget to update the length mixin node of course.

### Basic types and views

Basic types get packed in homogeneously typed subtrees, so the 1 Node = 1 object fails to hold.
And then there is the problem of a basic object being only represented by raw value (i.e. type information, but no instance specific properties).
This means that basic views cannot have a `ViewHook`, and will need be sliced out and into a regular root node.
However, basic types can also still be used as field elements, taking a full node.
This means that basic types are *an extension* of regular types.
And basic views provide *extended functionality*.

#### Basic type

Extend the type definition interface with:
- `byte_length() -> `: get the type size in number of bytes
- `basic_view_from_backing(base: Root, i: int) -> BasicView`: get a basic view from a root node, and an index that tells you where in this root the element is placed.
  - return `from_bytes(base[i*byte_length() : (i+1)*byte_length()])` can be the default.
- `from_bytes(bytez)`: to construct a basic view of just the applicable bytes. 

And now some of the regular type-def functionality can be described in terms of that of the basic type def:
- `default_node()` will always be `zero_node(0)`
- `view_from_backing(node, hook)` will ignore the `hook` and require `node` to be a `Root`. Then return `from_bytes(node[0:byte_length()])`.

#### Basic view

Extend the view interface with:
- `backing_from_base(base: Root, i: int) -> Root`: plug in the basic view contents into the root at the given position, return the new root.
  - return `base[:i*byte_length()] + to_bytes() + base[(i+1)*byte_length():]` can be the default.
- `to_bytes() -> bytes`: encode the basic view as bytes

And implement `View`:
- `get_backing() -> Node`: return `pad_right(to_bytes(), length=32)`
- `set_backing(b: Node)`: error, basic views are pass by value only.

#### Basic vector and basic list

To pack the actual `BasicView`s in lists and vectors, we need a type that is aware of the `basic_view_from_backing` and `backing_from_base`.
One option is to add extra conditions to the regular `Vector`/`List`, another option is to define `BasicVector` and `BasicList`:
- `elem_type` must be a `BasicType`
- `get` and `set` return `BasicView` instead. And use `byte_length()` to get the right root, and then derive the sub-index to slice out the `BasicView` from the backing root.

#### Future

This pattern can be improved to `HookedView` (mutates superview, backed by node), `BackedView` (pass by value, backed by node), and `BasicView` (pass by value, view is converted back end forth to node).
 To further rule out errors on compile-time by using more types to express intentions more precisely.

### Bitvector and Bitlist

These are more difficult, and have rather interesting use-cases:
- `BitVector[4]` for the justification data fits within a single root just fine. No need to complicate logic.
- `Bitlist`s used for pending attestations are bigger, but also don't make much sense to partially represent. Caching the root of the bitlist as a whole would be best.

However, for compatibility with loading merkle-proofs from general bytes32 backings, we still need to define how the bitvector and bitlists are loaded from and into trees.
So here this is one way of representing it as a tree, when necessary:
- use `SubtreeType`/`SubtreeView`, but for a `depth` derived from `bit_length/256` (Or `limit` and the mix-in for lists, as with regular `List`s).
- Define a `bit_view_from_backing(base: Root, i: int) -> BoolView` and `backing_from_bitfield_base(base: Root, i: int) -> Root`, similar to basic types, but for 1 bit at a time.
- `BitVector` and `BitList` are like the basic vectors/lists, but without element-type param (always `BoolType`), and using above methods to interpret and output roots.

## Results

This has been implemented in Go, see [`ztyp`](https://github.com/protolambda/ztyp), although experimental, and more verbose/unpolished as Go does not have meta-programming. But interfaces and embedding get your relatively close.
And in Python, see [`pymerkles`](https://github.com/protolambda/pymerkles) much less verbose, and nicely put together with meta-programming (metaclasses, not compile time). Work in progress, not complete.

Also, ZRNT has a `tree-state` branch that introduces `ztyp` to represent state, however, error handling verbosity, lack of generics, or any meta-programming at all, makes this transition a big pain.
Relatively far, but no encoding/decoding on top of tree-state defined yet, and state-transition has not completely migrated yet.

Word of advice here is to:
- Pick a language which abstracts these error-cases better. Partial trees with unexpanded elements shouldn't result in a 2x lines of code blow-up because of passing errors around.
    - think of exceptions, or Rust early-returns on errors/null.
- Pick a language which allows you to parametrize types well. Defining types as objects as benefits of writing code around the type properties more easily,
 but defining both a view and a type definition for each consensus type gets tedious.
    - Python metaclasses solve this exact issue.
    - Nim may be powerfull enough to do the same, or even better during compile time.
    - Rust gets close enough with deriving types/properties from other types etc.
    - Golang is just very, very, very verbose. Almost to a point where things get so repetitive that it is more likely to write bugs, instead of less likely. 
- Start with `uint64`, `bytes32`, `Container` and `Vector`. This is what most of the Eth2 spec uses, and good for initial numbers.

Some initial numbers on `pymerkles` and `ztyp`:

Benchmark:
- Pre: A validator registry filled with 100,000 validators.
- Bench op: append a new validator, and run the hash-tree-root of the registry.
    - the new validator is a new copy, not a reference to the same immutable validator.

Golang with ztyp: `0.057` ms/op.
Python with pymerkles: `0.2` ms/op.

Note: with a new validator, and 41 hashes for the registry, that is about ~57 hashes, so Go matches roughly 1000 ns / hash. So that's ~2x overhead over the regular merkle hash speed, to navigate the tree and allocate new memory for the new validator on the heap, etc.
And then Python is about 4x slower (pymerkles in active dev, number is anywhere between `0.1 to 0.25` depending on features/optimizations balance).

