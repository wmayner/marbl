Marbl specification
===================

_Version 1.0.1_

**Marbl** is a specification for a normalized representation of a node in a
Bayesian network together with its Markov blanket: a **marbl**.

Recall that the Markov blanket consists of the node's parents, its children,
and its children's other parents. The joint distribution of the variables in
the Markov blanket of a node is sufficient knowledge for calculating the
distribution of the node.

### Implementations ###

* **Python**: [Marbl-Python](https://github.com/wmayner/marbl-python)


Motivation
----------

We would like to be able to determine whether two nodes are “equivalent”
without needing to know the global structure of the network they belong to, or
how the network is implemented in a program. This allows for aggressive
memoization of expensive functions whose output depends only on the causal
properties of a node.

For example, let's say we have an expensive function `f(node)`, whose output
depends on the `node`'s Markov blanket. Cleverly, we memoize `f`, and lookup
precomputed values using `hash(node)`. But suppose that a few months later, the
implementation of the `node` object changes, and `hash(node)` is no longer the
same. The previously cached values will no longer be returned.

In addition, we might be doing computation on networks that contain many nodes
that are causally equivalent, even though their transition probability matrices
are not identical.

Normalizing the representation of a Markov blanket allows us to avoid
recomputing `f` for the equivalent nodes, and isolates the cache from changes
in implementation.


A normal form
-------------

The structure of a Bayesian network can be represented as the underlying
digraph together with a set of transition probability mappings, one for each
node `n`, which map the current state of `n`'s parents to the probability of
its next state. For example, if `n` is a binary node with two parents, it might
have the following TPM:

| Current state of parents, `(p0, p1)` | Probability of `n` being on |
| :----------------------------------: | :-------------------------: |
| (0, 0)                               | 0                           |
| (0, 1)                               | 0                           |
| (1, 0)                               | 0.5                         |
| (1, 1)                               | 0                           |

### Representing a TPM ###

There is not a unique way of implementing the mapping described above.
Typically, the mappings will be given as tables, but the tables can be
formatted in different ways.

For ease of implementation, we define the canonical representation of a TPM as
a `p`-dimensional array, where `p` is the number of parents, such that _e.g._
`TPM[i][j][k]` gives a list of the probabilities of the node's next state given
that the current state of the parents is `(i, j, k)`.

For example, the first TPM above would be represented as

    [[0.0, 0.5],
     [0.0, 0.0]]

Probabilities in TPMs must always be represented as floating-point values.

### Equivalence relation on TPMs ###

However, in order to represent a node as a TPM in this way, we are forced to
choose an ordering of its parents. The same node `n` that's given by the TPM
above could be represented by the following TPM, in which the ordering of the
parents is switched:

| `(p1, p0)` | `n` |
| :--------: | :-: |
| (0, 0)     | 0   |
| (0, 1)     | 0.5 |
| (1, 0)     | 0   |
| (1, 1)     | 0   |

We need a way of representing TPMs that is invariant under rearranging the
labels of the parent nodes.

So, we need to define a canonical representation for a member of the
equivalence class of TPMs where the equivalence relation is permutation of the
parent labeling. 

Note that because of how we defined TPM arrays, permuting the order of the
parents corresponds to permuting the dimensions of the array.

### Normal form of a TPM ###

Each parent permutation corresponds to some ordering of the entries in the TPM.
We define the **normal form** of the TPM to be the **lexicographically-least
such ordering**. 

In our example, we have two equivalent TPMs:

    [[0.0, 0.0],
     [0.5, 0.0]]
    
and

    [[0.0, 0.5],
     [0.0, 0.0]]

The first is the lexicographically-least of the two, so it is the normal form
of `n`'s TPM.

**Note on time complexity:**
Unfortunately, finding the normal form of a TPM is not efficient, because it
involves finding all the permutations of the parent nodes, which is `O(p!)`.

### Normal form of a Markov Blanket ###

We now have a normal form for a node as given by a TPM, but the goal is to
define a normal form for a full Markov blanket.

Since a given node's TPM encodes all the information that its parents give
about its state, to represent the Markov blanket with TPMs we need only the
covered node's TPM and the TPMs of the covered node's children. The information
that the children's other parents give about `n` is encoded in the children's
TPMs.

However, when normalizing a child's TPM, information is lost unless we keep
track of which of the child's parents is the covered node. To do this, we store
the index of the dimension of the child's TPM that corresponds to the covered
node after permuting the dimensions to obtain the normal form.

Note that it is possible that more than one permutation of the dimensions
yeilds the TPM's normal form, in which case the new index of covered-node's
dimension is not well-defined. So, of the parent permuations that yield the
normalized TPM, we take the lexicographically-least, where permutations are
represented as permutations of the sequence `[0, 1, 2, ..., n]`. The new dimension
index is defined to be the image of the old dimension index under this
lexicographically-least permutation.

Dimension indices must be represented as integers (not floating-point values,
as in TPMs).

Thus for each child of the covered node, we have an array where the first
element is the image of the index of the covered node's dimension in the
child's un-normalized TPM under the lexicographically-least permutation that
yeilds its normalized TPM, and the second element is the normalized TPM itself.
We'll call this structure a **normalized augmented child TPM**.

At this point, a Markov blanket in our TPM representation is just the TPM of
the node, and the unordered collection of its augmented child TPMs. So, in
order to specify a Markov blanket's normal form, we need only specify a
canonical ordering for the set of normalized augmented child TPMs. 

As usual, we simply choose the lexicographic ordering.

**Note**: All implementations of this specification must use zero-based
indexing.


Example
-------

We have now specified a canonical representation for a Markov blanket.

Consider the following network:


                   +----+         +---+  
                   |cp_0|-------->|c_0|
                   +----+         +---+
                                   ^
     +-----+                      /
     | p_0 |-----+       +-------+
     +-----+      \     /
                   +---+
                   | n |
                   +---+
     +-----+      /     \        
     | p_1 |-----+       +-------+
     +-----+                      \
                                   v
                   +----+         +---+  
                   |cp_1|-------->|c_1|
                   +----+         +---+

Where `n` has the TPM

| `(p0, p1)` | `n` |
| :--------: | :-: |
| (0, 0)     | 0   |
| (0, 1)     | 1   |
| (1, 0)     | 0   |
| (1, 1)     | 0   |

`c_0` has the TPM

| `(n, cp_0)` | `c_0` |
| :---------: | :---: |
| (0, 0)      | 0     |
| (0, 1)      | 0.9   |
| (1, 0)      | 0.1   |
| (1, 1)      | 0     |

and `c_1` has the TPM

| `(n, cp_1)` | `c_1` |
| :---------: | :---: |
| (0, 0)      | 0     |
| (0, 1)      | 0     |
| (1, 0)      | 0.2   |
| (1, 1)      | 0.8   |

We compute the normal form of `n`'s Markov blanket as follows:

1. First we normalize the covered node's TPM. Swapping the labeling of `n`'s
   parents yeilds the array

        [[0.0, 0.0],
         [1.0, 0.0]]
   
   which is lexicographically less than the given one,

        [[0.0, 1.0],
         [0.0, 0.0]]

   so the former is the normal form of `n`'s TPM.

2. Then we normalize the TPMs of the children, keeping track of which dimension
   corresponds to the covered node. Swapping the labeling of `c_0`'s parents
   yields the lexicographically-least array:
   
        [[0.0, 0.1],
         [0.9, 0.0]]
    
    With this new dimension permutation, `n` corresponds to the second
    dimension (in this small example, there can only be one permutation of the
    dimensions that yields the normalized TPM, so we don't need to worry about
    finding the right new dimension index). With zero-based indexing, that
    gives us a new covered-node dimension index of `1`:

        [ 1, [[0.0, 0.1],
              [0.9, 0.0]] ]

3. Repeating the process for the next child, we find that swapping the labeling
   of `c_1` yeilds 

        [[0.0, 0.2],
         [0.0, 0.8]]

   which is lexicographically greater than 
   
        [[0.0, 0.0],
         [0.2, 0.8]]

   so we keep the given ordering, which gives

        [ 0, [[0.0, 0.0],
              [0.2, 0.8]] ]

4. We then lexicographically sort the normalized augmented child TPMs:

        [  
            [ 0, [[0.0, 0.0],
                  [0.2, 0.8]] ],
            [ 1, [[0.0, 0.1],
                  [0.9, 0.0]] ]
        ]

5. We now have our normal form, consisting of the normalized TPM of `n`, and an
   array of the normalized TPM of `c_1`, and `c_0`, in the canonical order:

        [ 
            [[0.0, 0.0],
             [1.0, 0.0]],
            [  
                [ 0, [[0.0, 0.0],
                      [0.2, 0.8]] ],
                [ 1, [[0.0, 0.1],
                      [0.9, 0.0]] ]  
            ]   
        ]


Multisets of marbls
-------------------

We define the canonical ordering of an multiset of marbls to be the
**lexicographic ordering of their normal forms**.


Serialization
-------------

There are many serialization formats that can represent arrays of
floating-point values.

**[MessagePack](http://msgpack.org), I choose you!**

So, a marbl should be serialized as appropriately-nested
[arrays](https://github.com/msgpack/msgpack/blob/master/spec.md#array-format-family)
of
[floats](https://github.com/msgpack/msgpack/blob/master/spec.md#formats-float)
and
[ints](https://github.com/msgpack/msgpack/blob/master/spec.md#int-format-family).

Hashing
-------

There are also many cryptographic hash functions. 

We define the hash of a marbl or multiset of marbls to be the **SHA1 hash of
its serialization**.

We serialize it first so that the hashes are independent of programming
language.
