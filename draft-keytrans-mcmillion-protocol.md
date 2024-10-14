---
title: "Key Transparency Protocol"
category: info

docname: draft-keytrans-mcmillion-protocol-latest
submissiontype: IETF
v: 3
area: SEC
workgroup: KEYTRANS Working Group
venue:
  github: "Bren2010/draft-keytrans"
  latest: "https://Bren2010.github.io/draft-keytrans/draft-keytrans-mcmillion-protocol.html"

author:
 -
    fullname: "Brendan McMillion"
    email: "brendanmcmillion@gmail.com"
 -
    fullname: "Felix Linker"
    email: "linkerfelix@gmail.com"

normative:

informative:
  Merkle2:
    target: https://eprint.iacr.org/2021/453
    title: "Merkle^2: A Low-Latency Transparency Log System"
    date: 2021-04-08
    author:
      - name: Yuncong Hu
      - name: Kian Hooshmand
      - name: Harika Kalidhindi
      - name: Seung Jin Yang
      - name: Raluca Ada Popa


--- abstract

While there are several established protocols for end-to-end encryption,
relatively little attention has been given to securely distributing the end-user
public keys for such encryption. As a result, these protocols are often still
vulnerable to eavesdropping by active attackers. Key Transparency is a protocol
for distributing sensitive cryptographic information, such as public keys, in a
way that reliably either prevents interference or detects that it occurred in a
timely manner.

--- middle


# Introduction

End-to-end encrypted communication services rely on the secure exchange of
public keys to ensure that messages remain confidential. It is typically assumed
that service providers correctly manage the public keys associated with each
user's account. However, this is not always true. A service provider that is
compromised or malicious can change the public keys associated with a user's
account without their knowledge, thereby allowing the provider to eavesdrop on
and impersonate that user.

This document describes a protocol that enables a group of users to ensure that
they all have the same view of the public keys associated with each other's
accounts. Ensuring a consistent view allows users to detect when unauthorized
public keys have been associated with their account, indicating a potential
compromise.

More detailed information about the protocol participants and the ways the
protocol can be deployed can be found in {{!I-D.ietf-keytrans-architecture}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the TLS presentation language {{!RFC8446}} to describe the
structure of protocol messages, but does not require the use of a specific
transport protocol. As such, implementations do not necessarily need to transmit
messages according to the TLS format and can chose whichever encoding method
best suits their application. However, cryptographic computations MUST be done
with the TLS presentation language format to ensure the protocol's security
properties are maintained.


# Tree Construction

A Transparency Log is a verifiable data structure that maps *label-version
pairs* to cryptographic keys or other structured data. Labels correspond to user
identifiers, and a new version of a label is created each time the label's
associated value changes. Transparency Logs have an *epoch* counter which is
incremented every time a new set of label-version pairs are added.

~~~
  Epoch   Update      Set of logged Label-Version Pairs
* n       X:0 -> k    { ..., X:0 }
* n+1     Y:3 -> l    { ..., X:0, Y:3 }
* n+2     Z:2 -> m    { ..., X:0, Y:3, Z:2 }
* n+3     X:1 -> n    { ..., X:0, Y:3, Z:2, X:1 }
~~~
{: title="An example log history. At epoch n the user with label X submits their
first key k. As it is their first key, it is associated with version 0. At epoch
n+1, user Y updates their key to l."}

KT uses a *prefix tree* to commit to a mapping between each label-version pair
and a commitment to the label's value at that version. Every time the prefix
tree changes, its new root hash is stored in a *log tree*. The benefit of the
prefix tree is that it is easily searchable, and the benefit of the log tree is
that it can easily be verified to be append-only. The data structure powering KT
combines a log tree and a prefix tree, and is called the *combined tree
structure*.

This section describes the operation of
both prefix and log trees at a high level and the way that they're combined. More precise algorithms
for computing the intermediate and root values of the trees are given in
{{cryptographic-computations}}.

## Terminology

Trees consist of
*nodes* which have a byte string as their *hash value*. A node is either a
*leaf* if it has no children, or a *parent*
if it has either a *left child* or a *right child*. A node is the *root* of a
tree if it has no parents, and an *intermediate* if it has both children and
parents. Nodes are *siblings* if they share the same parent.

The *descendants* of a node are that node, its children, and the descendants of
its children. A *subtree* of a tree is the tree given by the descendants of a
particular node, called the *head* of the subtree.

The *direct path* of a root node is the empty list, and of any other node is the
concatenation of that node's parent along with the parent's direct path. The
*copath* of a node is the node's sibling concatenated with the list of siblings
of all the nodes in its direct path, excluding the root.

## Log Tree

Log trees are used for storing information in the chronological order that it
was added and are constructed as *left-balanced* binary trees.

A binary tree is *balanced* if its size is a power of two and for any parent
node in the tree, its left and right subtrees have the same size. A binary tree
is *left-balanced* if for every parent, either the parent is balanced, or the
left subtree of that parent is the largest balanced subtree that could be
constructed from the leaves present in the parent's own subtree. Given a list of
`n` items, there is a unique left-balanced binary tree structure with these
elements as leaves. Note also that every parent always has both a left and right
child.

~~~ aasvg
                             X
                             |
                   .---------+.
                  /            \
                 X              |
                 |              |
             .---+---.          |
            /         \         |
           X           X        |
          / \         / \       |
         /   \       /   \      |
        X     X     X     X     X

Index:  0     1     2     3     4
~~~
{: title="A log tree containing five leaves."}

Log trees initially consist of a single leaf node. New leaves are
added to the right-most edge of the tree along with a single parent node, to
construct the left-balanced binary tree with `n+1` leaves.

~~~ aasvg
                             X
                             |
                   .---------+---.
                  /               \
                 X                 |
                 |                 |
             .---+---.             |
            /         \            |
           X           X           X
          / \         / \         / \
         /   \       /   \       /   \
        X     X     X     X     X     X

Index:  0     1     2     3     4     5
~~~
{: title="Example of inserting a new leaf with index 5 into the previously
depicted log tree. Observe that only the nodes on the path from the new root to
the new leaf change."}

While leaves contain arbitrary data, the value of a parent node is always the
hash of the combined values of its left and right children.

Log trees are powerful in that they can provide both *inclusion proofs*, which
demonstrate that a leaf is included in a log, and *consistency proofs*, which
demonstrate that a new version of a log is an extension of a past version of the
log.

An inclusion proof is given by providing the copath values of a leaf. The proof
is verified by hashing together the leaf with the copath values and checking
that the result equals the root hash value of the log. Consistency proofs are a
more general version of the same idea. With a consistency proof, the prover
provides the minimum set of intermediate node values from the current tree that
allows the verifier to compute both the old root value and the current root
value. An algorithm for this is given in section 2.1.2 of {{!RFC6962}}.

~~~ aasvg
                             X
                             |
                   .---------+---.
                  /               \
                 X                 |
                 |                 |
             .---+---.             |
            /         \            |
          (X)           X         (X)
          / \         / \         / \
         /   \       /   \       /   \
        X     X     X   (X)    X     X

Index:  0     1     2     3     4     5
~~~
{: title="Illustration of a proof of inclusion. To verify that leaf 2 is
included in the tree, the server provides the client with the hashes of the
nodes on its copath, i.e., all hashes that are required for the client to
compute the root hash itself. In the figure, the copath consists of the nodes
marked by (X)."}

~~~ aasvg
                             X
                             |
                   .---------+----.
                  /                \
                (X)                 |
                 |                 |
             .---+---.             |
            /         \            |
           X           X           X
          / \         / \         / \
         /   \       /   \       /   \
        X     X     X     X    (X)   [X]

Index:  0     1     2     3     4     5
~~~
{: title="Illustration of a consistency proof. The server proves to the client
that it correctly extended the tree by giving it the hashes of marked nodes
([X] / (X)). The client can verify that it can construct the old root hash
from the hashes of nodes marked by (X), and that it can construct the new root
hash when also considering the hash of the node [X]."}

## Prefix Tree

Prefix trees are used for storing key-value pairs, in a way that provides the
ability to efficiently prove that a search key's value was looked up correctly.

Each leaf node in a prefix tree represents a specific key-value pair, while each parent
node represents some prefix which all search keys in the subtree headed by that node
have in common. The subtree headed by a parent's left child contains all search keys
that share its prefix followed by an additional 0 bit, while the subtree headed
by a parent's right child contains all search keys that share its prefix followed by
an additional 1 bit.

The root node, in particular, represents the empty string as a prefix. The
root's left child contains all search keys that begin with a 0 bit, while the right
child contains all search keys that begin with a 1 bit.

A prefix tree can be searched by starting at the root node, and moving to the
left child if the first bit of a search key is 0, or the right child if the first bit
is 1. This is then repeated for the second bit, third bit, and so on until the
search either terminates at a leaf node (which may or may not be for the desired
value), or a parent node that lacks the desired child.

~~~ aasvg
                     X
                     |
             .-------+-----.
            /               \
            0                1
            |                |
            |             .--+-.
            |            /      \
            0           0        |
           / \         / \       |
          /   \       /   \      |
Key:   00010 00101 10001 10111 11011
Value:     A     B     C     D     E
~~~
{: title="A prefix tree containing five entries."}

New key-value pairs are added to the tree by searching it according to the same process.
If the search terminates at a parent without a left or right child, a new leaf
is simply added as the parent's missing child. If the search terminates at a
leaf for the wrong search key, one or more intermediate nodes are added until the new
leaf and the existing leaf would no longer reside in the same place. That is,
until we reach the first bit that differs between the new search key and the existing
search key.

~~~ aasvg
                          X
                          |
                   .------+------.
                  /               \
                 0                 1
                 |                 |
              .--+-.            .--+-.
             /      \          /      \
            0        |        0        |
           / \       |       / \       |
          /   \      |      /   \      |
Index: 00010 00101 01101 10001 10111 11011
Value:     A     B     F     C     D     E
~~~
{: title="The previous prefix tree after adding the key-value pair: 01101 -> F."}

The value of a leaf node is the encoded key-value pair, while the value of a
parent node is the hash of the combined values of its left and right children
(or a stand-in value when one of the children doesn't exist).

A proof of membership is given by providing the leaf hash value, along with the
hash value of each copath entry along the search path. A proof of non-membership
is given by providing an abridged proof of membership that follows the
path for the intended search key, but ends either at a stand-in node or a leaf for a
different search key. In either case, the proof is verified by hashing together the
leaf with the copath hash values and checking that the result equals the root
hash value of the tree.

## Combined Tree

Log trees are desirable because they can provide efficient consistency proofs to
assure verifiers that nothing has been removed from a log that was present in a
previous version. However, log trees can't be efficiently searched without
downloading the entire log. Prefix trees are efficient to search and can provide
inclusion proofs to convince verifiers that the returned search results are
correct. However, it's not possible to efficiently prove that a new version of a
prefix tree contains the same data as a previous version with only new values
added.

In the combined tree structure, which is based on {{Merkle2}}, a prefix tree
contains a mapping where each label-version pair is a search key, and its
associated value is a cryptographic commitment to the label's new contents. A
log tree contains a record of each version of the prefix tree that's created.
With some caveats, this combined structure supports both efficient consistency
proofs and can be efficiently searched.

Note that, although the Transparency Log maintains a single logical prefix tree,
each modification of this tree results in a new root hash, which is then stored
in the log tree. Therefore, when instructions refer to "looking up a label-version pair in the
prefix tree at a given log entry," this actually means searching in the specific
version of the prefix tree that corresponds to the root hash stored at that log
entry (where a "log entry" refers to a leaf of the log tree).

~~~ aasvg
Epoch        n-1         n                       n+1

Root hash     o -------- o ---------------------- o
             / \         |                        |
            /___\     .--+--.              .------+------.
                     / \    |             / \            |
                    /___\   |            /___\   .-------+-------.
                            |                   /                 \
                    [X:0->k; h(PT_n)]  [X:0->k; h(PT_n)]  [Y:3->l; h(PT_n+1)]
~~~
{: title="An example evolution of the combined tree structure, following the
example from the beginning of the section. At every epoch, a new leaf is added
to a log tree, containing a change to a label's value and the updated
prefix tree (PT)."}

# Searching the Tree

When searching the combined tree structure described in {{combined-tree}}, the
proof provided by the Transparency Log may either be **full** or **abridged**. A
full proof must be provided if the deployment mode of the Transparency Log is
Contact Monitoring, or if the user has specifically requested it. Otherwise,
proofs are provided abridged.

A full proof follows the path of a binary search for the first log entry where
the prefix tree contains the desired label-version pair.
This ensures that all users will check the same or
similar entries when searching for the same label, allowing for
efficient client-side auditing of the Transparency Log. The binary search uses
an *implicit binary search tree* constructed over the leaves of the log tree
(distinct from the structure of the log tree itself), which allows the search to
have a complexity logarithmic in the number of the log's leaves.

An abridged proof skips this binary search, and simply looks at the most recent
version of the prefix tree to determine the commitment to
the update that the user is looking for. Abridged proofs rely on a third-party
auditor or manager that can be trusted not to collude with the Transparency Log,
and who checks that every version of the prefix tree is constructed correctly.
This is described in more detail in {{putting-it-together}}.

## Implicit Binary Search Tree

Intuitively, the leaves of the log tree can be considered a flat array
representation of a binary tree. This structure is similar to the log tree, but
distinguished by the fact that not all parent nodes have two children. In this
representation, "leaf" nodes are stored in even-numbered indices, while
"intermediate" nodes are stored in odd-numbered indices:

~~~ aasvg
                             X
                             |
                   .---------+---------.
                  /                     \
                 X                       X
                 |                       |
             .---+---.               .---+---.
            /         \             /         \
           X           X           X           X
          / \         / \         / \         /
         /   \       /   \       /   \       /
        X     X     X     X     X     X     X

Index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13
~~~
{: title="A binary tree constructed from 14 entries in a log" }

Following the structure of this binary tree when executing searches makes
auditing the Transparency Log much more efficient because users can easily
reason about which nodes will be accessed when conducting a search. As such,
only nodes along a specific search path need to be checked for correctness.

The following Python code demonstrates the computations used for following this
tree structure:

~~~ python
# The exponent of the largest power of 2 less than x. Equivalent to:
#   int(math.floor(math.log(x, 2)))
def log2(x):
    if x == 0:
        return 0
    k = 0
    while (x >> k) > 0:
        k += 1
    return k-1

# The level of a node in the tree. Leaves are level 0, their parents
# are level 1, etc. If a node's children are at different levels,
# then its level is the max level of its children plus one.
def level(x):
    if x & 0x01 == 0:
        return 0
    k = 0
    while ((x >> k) & 0x01) == 1:
        k += 1
    return k

# The root index of a search if the log has `n` entries.
def root(n):
    return (1 << log2(n)) - 1

# The left child of an intermediate node.
def left(x):
    k = level(x)
    if k == 0:
        raise Exception('leaf node has no children')
    return x ^ (0x01 << (k - 1))

# The right child of an intermediate node.
def right(x, n):
    k = level(x)
    if k == 0:
        raise Exception('leaf node has no children')
    x = x ^ (0x03 << (k - 1))
    while x >= n:
        x = left(x)
    return x
~~~

The `root` function returns the index in the log at which a search should
start. The `left` and `right` functions determine the subsequent index to be
accessed, depending on whether the search moves left or right.

For example, in a search where the log has 50 entries, instead of starting the
search at the typical "middle" entry of `50/2 = 25`, users would start at entry
`root(50) = 31`. If the next step in the search is to move right, the next index
to access would be `right(31, 50) = 47`. As more entries are added to the log,
users will consistently revisit entries 31 and 47, while they may never revisit
entry 25 after even a single new entry is added to the log.

## Binary Ladder

When executing searches on a Transparency Log, the implicit tree described in
{{implicit-binary-search-tree}} is navigated according to a binary search. At
each individual log entry, the binary search needs to determine whether it
should move left or right. That is, it needs to determine, out of the set of
label-version pairs stored in the prefix tree, whether the highest version of a
label that's present at a given log entry is greater than, equal to, or less
than a target version.

A **binary ladder** is a series of lookups in a single log entry's prefix tree,
which is used to establish whether the target version of a label is present or
not. It consists of the following lookups, stopping after the first lookup that
produces a proof of non-inclusion:

1. First, version `x` of the label is looked up, where `x` is consecutively higher
   powers of two minus one (0, 1, 3, 7, ...). This is repeated until `x` is the
   largest such value less than or equal to the target version.
2. Second, the largest `x` that was looked up is retained, and consecutively
   smaller powers of two are added to it until it equals the target version.
   Each time a power of two is added, this version of the label is looked up.

As an example, if the target version of a label to lookup is 20, a binary ladder
would consist of the following versions: 0, 1, 3, 7, 15, 19, 20. If all of the
lookups succeed (i.e., result in proofs of inclusion), this indicates that the
target version of the label exists in the log. If the ladder stops early because a
proof of non-inclusion was produced, this indicates that the target version of
the label did not exist, as of the given log entry.

When executing a search in a Transparency Log for a specific version of a label, a
binary ladder is provided for each node on the search path, verifiably guiding
the search toward the log entry where the desired label-version pair was first
inserted (and therefore, the log entry with the desired update).

Requiring proof that this series of versions are present in the prefix tree,
instead of requesting proof of just version 20, ensures that all users are able
to agree on which version of the label is *most recent*, which is discussed
further in the next section.

## Most Recent Version

Often, users wish to search for the "most recent" version of a label. That is, the
label with the highest version possible.

To determine this, users request a **full binary ladder** for each
node on the **frontier** of the log. The frontier consists of the root node of a
search, followed by the entries produced by repeatedly calling `right` until
reaching the last entry of the log. Using the same example of a search where the
log has 50 entries, the frontier would be entries: 31, 47, 49.

A full binary ladder is similar to the binary ladder discussed in the previous
section, except that it identifies the exact highest version of a label that
exists, as of a particular log entry, rather than stopping at a target version.
It consists of the following lookups:

1. First, version `x` of the label is looked up, where `x` is a consecutively
   higher power of two minus one (0, 1, 3, 7, ...). This is repeated until the
   first proof of non-inclusion is produced.
2. Once the first proof of non-inclusion is produced, a binary search is
   conducted between the highest version that was proved to be included, and the
   version that was proved to not be included. Each step of the binary search
   produces either a proof of inclusion or non-inclusion, which guides the
   search left or right, until it terminates.

For the purpose of finding the highest version possible, requesting a full
binary ladder for each entry along the frontier is functionally the same as
doing so for only the last log entry. However, inspecting the entire frontier
allows the user to verify that the search path leading to the last log entry
represents a monotonic series of version increases, which minimizes
opportunities for log misbehavior.

Once the user has verified that the frontier lookups are monotonic and
determined the highest version, the user then continues a binary search for this
specific version.

## Putting it Together

As noted at the beginning of the section, a search in the tree will either
require producing a full proof, or an abridged proof may be accepted if the user
can trust a third-party to audit and not collude with the Transparency Log.

The steps for producing a full or abridged search proof are summarized as
follows:

- Full proof:
  - If searching for the most recent version of a label, a full binary ladder
    is obtained for each node on the frontier of the log. This determines the
    highest version of the label available, which allows the search to proceed
    for this specific version.
  - If searching for a specific version, the proof follows a binary search for
    the first entry in the log where this version of the label exists. For
    each step in the binary search, the proof contains a (non-full) binary
    ladder for the targeted version, which proves whether the targeted version
    of the label existed yet or not by this point in the log. This indicates
    whether the binary search should move forwards or backwards in the log.
- Abridged proof:
  - If searching for the most recent version of a label, a full binary ladder is
    obtained only from the last (most recent) entry of the log. The prefix tree
    entry for the most recent version of the label will contain the commitment to the
    update, ending the search.
  - If searching for a specific version, a (non-full) binary ladder for this
    version is obtained only from the last entry of the log. Similar to the
    previous case, the prefix tree entry for the targeted version will contain
    the commitment to the update.


# Monitoring the Tree

As new entries are added to the log tree, the search path that's traversed to
find a specific version of a label may change. New intermediate nodes may become
established in between the search root and the leaf, or a new search root may be
created. The goal of monitoring a label is to efficiently ensure that, when these
new parent nodes are created, they're created correctly so that searches for the
same versions continue converging to the same entries in the log.

To monitor a given label, users maintain a small amount of state: a map
from a position in the log to a version counter. The version counter is the
highest version of the label that's been proven to exist at that log
position. Users initially populate this map by setting the position of an entry
they've looked up, to map to the version of the label stored in that entry. A map
may track several different versions of a label simultaneously, if a user
has been shown different versions of the same label.

To update this map, users receive the most recent tree head from the server and
follow these steps, for each entry in the map:

1. Compute the entry's direct path (in terms of the Implicit Binary Search Tree)
   based on the current tree size.
2. If there are no entries in the direct path that are to the right of the
   current entry, then skip updating this entry (there's no new information to
   update it with).
3. For each entry in the direct path that's to the right of the current entry,
   from low to high:
   1. Receive and verify a binary ladder from that log entry, for the version
      currently in the map. This proves that, at the indicated log entry, the
      highest version present is greater than or equal to the
      previously-observed version.
   2. If the above check was successful, remove the current position-version
      pair from the map and replace it with a position-version pair
      corresponding to the entry in the log that was just checked.

This algorithm progressively moves up the tree as new intermediate/root nodes
are established and verifies that they're constructed correctly. Note that users
can often execute this process with the output of Search or Update operations
for a label, without waiting to make explicit Monitor queries.

It is also worth noting that the work required to monitor several versions of
the same label scales sublinearly, due to the fact that the direct paths of the
different versions will often intersect. Intersections reduce the total number
of entries in the map and therefore the amount of work that will be needed to
monitor the label from then on.

Finally, unlike searching, there is no abridged version of monitoring.


# Ciphersuites

Each Transparency Log uses a single fixed ciphersuite, chosen when the log is
initially created, that specifies the following primitives to be used for
cryptographic computations:

* A hash algorithm
* A signature algorithm
* A Verifiable Random Function (VRF) algorithm

The hash algorithm is used for computing the intermediate and root values of
hash trees. The signature algorithm is used for signatures from both the service
operator and the third party, if one is present. The VRF is used for preserving
the privacy of labels. One of the VRF algorithms from {{!RFC9381}} must be
used.

Ciphersuites are represented with the CipherSuite type. The ciphersuites are
defined in {{kt-ciphersuites}}.


# Cryptographic Computations

## Verifiable Random Function

Each label-version pair created in a log will have a unique
representation in the prefix tree. This is computed by providing the combined
label and version as inputs to the VRF:

~~~ tls-presentation
struct {
  opaque label<0..2^8-1>;
  uint32 version;
} VrfInput;
~~~

The VRF's output evaluated on `VrfInput` is the concrete value inserted into the
prefix tree.

## Commitment

As discussed in {{combined-tree}}, commitments are stored in the leaves of the
log tree and correspond to updates. Commitments are computed
with HMAC {{!RFC2104}}, using the hash function specified by the ciphersuite. To
produce a new commitment, the application generates a random 16 byte value
called `opening` and computes:
<!-- TODO Opening size should be determined by ciphersuite -->

~~~ pseudocode
commitment = HMAC(fixedKey, CommitmentValue)
~~~

where `fixedKey` is the 16 byte hex-decoded value:

~~~
d821f8790d97709796b4d7903357c3f5
~~~
<!-- TODO This is weird but nobody has ever said anything. Keep it? Do better? -->

and CommitmentValue is specified as:

~~~ tls-presentation
struct {
  opaque opening<16>;
  opaque label<0..2^8-1>;
  UpdateValue update;
} CommitmentValue;
~~~

This fixed key allows the HMAC function, and thereby the commitment scheme, to
be modeled as a random oracle. The `label` field of CommitmentValue
contains the label being updated
and the `update` field contains the new value for the label.

The output value `commitment` may be published, while `opening` should be kept
private until the commitment is meant to be revealed.

## Prefix Tree

The leaf nodes of a prefix tree are serialized as:

~~~ tls
struct {
    opaque vrf_output<VRF.Nh>;
    uint64 update_index;
} PrefixLeaf;
~~~

where `vrf_output` is the VRF output for the label-version pair, `VRF.Nh` is the
output size of the ciphersuite VRF in bytes, and `update_index` is the index of
the log tree's leaf committing to the respective value, i.e., the log tree's
`tree_size` just after the respective label-version pair was inserted minus one.

The parent nodes of a prefix tree are serialized as:

~~~ tls
struct {
  opaque value<Hash.Nh>;
} PrefixParent;
~~~

where `Hash.Nh` is the output length of the ciphersuite hash function. The value
of a parent node is computed by hashing together the values of its left and
right children:

~~~ pseudocode
parent.value = Hash(0x01 ||
                   nodeValue(parent.leftChild) ||
                   nodeValue(parent.rightChild))

nodeValue(node):
  if node.type == emptyNode:
    return 0 // all-zero vector of length Hash.Nh
  else if node.type == leafNode:
    return Hash(0x00 || node.vrf_output || node.update_index)
  else if node.type == parentNode:
    return node.value
~~~

where `Hash` denotes the ciphersuite hash function.

## Log Tree {#crypto-log-tree}

The leaf and parent nodes of a log tree are serialized as:

~~~ tls-presentation
struct {
  opaque commitment<Hash.Nh>;
  opaque prefix_tree<Hash.Nh>;
} LogLeaf;

struct {
  opaque value<Hash.Nh>;
} LogParent;
~~~

The `commitment` field contains the output of evaluating HMAC on
`CommitmentValue`, as described in {{commitment}}. The `prefix_tree` field
contains the root hash of the prefix tree, after inserting a new label-version
pair for the `label` in `CommitmentValue`.

The value of a parent node is computed by hashing together the values of its
left and right children:

~~~ pseudocode
parent.value = Hash(hashContent(parent.leftChild) ||
                    hashContent(parent.rightChild))

hashContent(node):
  if node.type == leafNode:
    return 0x00 || nodeValue(node)
  else if node.type == parentNode:
    return 0x01 || nodeValue(node)

nodeValue(node):
  if node.type == leafNode:
    return Hash(node.commitment || node.prefix_tree)
  else if node.type == parentNode:
    return node.value
~~~

## Tree Head Signature

The head of a Transparency Log, which represents the log's most recent state, is
represented as:

~~~ tls-presentation
struct {
  uint64 tree_size;
  opaque signature<0..2^16-1>;
} TreeHead;
~~~

where `tree_size` counts the number of entries in the log tree. If the
Transparency Log is deployed with Third-party Management then the public key
used to verify the signature belongs to the third-party manager; otherwise the
public key used belongs to the service operator.

The signature itself is computed over a `TreeHeadTBS` structure, which
incorporates the log's current state as well as long-term log configuration:

~~~ tls-presentation
enum {
  reserved(0),
  contactMonitoring(1),
  thirdPartyManagement(2),
  thirdPartyAuditing(3),
  (255)
} DeploymentMode;

struct {
  CipherSuite ciphersuite;
  DeploymentMode mode;
  opaque signature_public_key<0..2^16-1>;
  opaque vrf_public_key<0..2^16-1>;

  select (Configuration.mode) {
    case contactMonitoring:
    case thirdPartyManagement:
      opaque leaf_public_key<0..2^16-1>;
    case thirdPartyAuditing:
      opaque auditor_public_key<0..2^16-1>;
  };
} Configuration;

struct {
  Configuration config;
  uint64 tree_size;
  opaque root<Hash.Nh>;
} TreeHeadTBS;
~~~


# Tree Proofs

## Log Tree

An inclusion proof for a single leaf in a log tree is given by providing the
copath values of a leaf. Similarly, a bulk inclusion proof for any number of
leaves is given by providing the fewest node values that can be hashed together
with the specified leaves to produce the root value. Such a proof is encoded as:

~~~ tls-presentation
opaque NodeValue<Hash.Nh>;

struct {
  NodeValue elements<0..2^16-1>;
} InclusionProof;
~~~

Each `NodeValue` is a uniform size, computed by passing the relevant `LogLeaf`
or `LogParent` structures through the `nodeValue` function in
{{crypto-log-tree}}. The contents of the `elements` array is kept in
left-to-right order: if a node is present in the root's left subtree, its value
must be listed before any values provided from nodes that are in the root's
right subtree, and so on recursively.

Consistency proofs are encoded similarly:

~~~ tls-presentation
struct {
  NodeValue elements<0..2^8-1>;
} ConsistencyProof;
~~~

Again, each `NodeValue` is computed by passing the relevant `LogLeaf` or
`LogParent` structure through the `nodeValue` function. The nodes chosen
correspond to those output by the algorithm in Section 2.1.2 of {{RFC6962}}.

## Prefix Tree

A proof from a prefix tree authenticates that a set of values are either members
of, or are not members of, the total set of values represented by the prefix
tree. Such a proof is encoded as:

~~~ tls
enum {
  reserved(0),
  inclusion(1),
  nonInclusionLeaf(2),
  nonInclusionParent(3),
} PrefixSearchResultType;

struct {
  PrefixSearchResultType result_type;
  select (PrefixSearchResult.result_type) {
    case inclusion:
      uint64 update_index;
    case nonInclusionLeaf:
      PrefixLeaf leaf;
  };
  uint8 depth;
} PrefixSearchResult;

struct {
  PrefixSearchResult results<0..2^8-1>;
  NodeValue elements<0..2^16-1>;
} PrefixProof;
~~~

The `results` field contains the search result for each individual value. It is
sorted lexicographically by corresponding value. The `result_type` field of each
`PrefixSearchResult` struct indicates what the terminal node of the search for
that value was:

- `inclusion` for a leaf node matching the requested value.
- `nonInclusionLeaf` for a leaf node not matching the requested value. In this
  case, the terminal node's value is provided given that it can not be inferred.
- `nonInclusionParent` for a parent node that lacks the desired child.

The `depth` field indicates the depth of the terminal node of the search, and is
provided to assist proof verification.

The `elements` array consists of the fewest node values that can be hashed
together with the provided leaves to produce the root. The contents of the
`elements` array is kept in left-to-right order: if a node is present in the
root's left subtree, its value must be listed before any values provided from
nodes that are in the root's right subtree, and so on recursively. In the event
that a node is not present, an all-zero byte string of length `Hash.Nh` is
listed instead.

The proof is verified by hashing together the provided elements, in the
left/right arrangement dictated by the tree structure, and checking that the
result equals the root value of the prefix tree.

## Combined Tree {#proof-combined-tree}

### Proofs for Contact Monitoring

A proof from a combined log and prefix tree follows the execution of a binary
search through the leaves of the log tree, as described in {{combined-tree}}. It
is serialized as follows:

~~~ tls-presentation

struct {
  opaque proof<VRF.Np>;
} VRFProof;

struct {
  PrefixProof prefix_proof;
  opaque commitment<Hash.Nh>;
} ProofStep;

struct {
  optional<uint32> version;
  VRFProof vrf_proofs<0..2^8-1>;
  ProofStep steps<0..2^8-1>;
  InclusionProof inclusion;
} SearchProof;
~~~

If searching for the most recent version of a label, the most recent version is
provided in `version`. If searching for a specific version, this field is
omitted.

Each element of `vrf_proofs` contains the output of evaluating the VRF on a
different version of the label. The versions chosen correspond either to
the binary ladder described in {{binary-ladder}} (when searching for a specific
version of a label), or to the full binary ladder described in
{{most-recent-version}} (when searching for the most recent version of a label).
The proofs are sorted from lowest version to highest version.

Each `ProofStep` structure in `steps` is one log entry that was inspected as
part of the binary search. The first step corresponds to the "middle" leaf of
the log tree (calculated with the `root` function in
{{implicit-binary-search-tree}}). From there, each subsequent step moves left or
right in the tree, according to the procedure discussed in {{binary-ladder}} and
{{most-recent-version}}.

The `prefix_proof` field of a `ProofStep` is the output of executing a binary
ladder, excluding any ladder steps for which a proof of inclusion is expected,
and a proof of inclusion was already provided in a previous `ProofStep` for a
log entry to the left of the current one.

The `commitment` field of a `ProofStep` contains the commitment to the update at
that leaf. The `inclusion` field of `SearchProof` contains a batch inclusion
proof for all of the leaves accessed by the binary search.

The proof can be verified by checking that:

1. The elements of `steps` represent a monotonic series over the leaves of the
   log, and
2. The `steps` array has the expected number of entries (no more or less than
   are necessary to execute the binary search).
<!-- TODO Specify this verification in MUCH more detail -->

Once the validity of the search steps has been established, the verifier can
compute the root of each prefix tree represented by a `prefix_proof` and combine
it with the corresponding `commitment` to obtain the value of each leaf. These
leaf values can then be combined with the proof in `inclusion` to check that the
output matches the root of the log tree.

### Proofs for Third-Party Auditing

In third-party auditing, clients can rely on the assumption that the prefix tree
is monitored to be append-only. Therefore, they need not execute the binary
ladder but the proof can directly jump to the index identified by the prefix
tree leaf.

~~~ tls-presentation
struct {
  optional<uint32> version;
  VRFProof vrf_proofs<0..2^8-1>;
  PrefixProof prefix_proof;
  InclusionProof inclusion;
} SearchProofCompact;
~~~

The semantics of the `version` field do not change.

Similarly to `SearchProof`, `vrf_proofs` contains the output of evaluating the
VRF on a different version of the label. Either one version will be included
(when requesting a specific version) or the versions to verify the full binary
ladder (when requesting the latest version).

`prefix_proof` contains the proof to either verify the inclusion of the
label-version pair (when requesting a specific version) or to verify the full
binary ladder (when requesting the latest version). Both types of proofs are for
the most recent prefix tree.

`inclusion` contains a batch inclusion of the most recent leaf and the leaf that
commits to respective value for the request label-version pair. The most recent
leaf is needed to obtain the prefix tree's root hash, and the leaf committing to
the requested value will be at the index identified in the most recent prefix
tree.

# Update Format

The updates committed to by a combined tree structure contain the new value of a
label, along with additional information depending on the deployment mode
of the Transparency Log. They are serialized as follows:

~~~ tls-presentation
struct {
  select (Configuration.mode) {
    case thirdPartyManagement:
      opaque signature<0..2^16-1>;
  };
} UpdatePrefix;

struct {
  UpdatePrefix prefix;
  opaque value<0..2^32-1>;
} UpdateValue;
~~~

The `value` field contains the new value associated with the label.

In the event that third-party management is used, the `prefix` field contains a
signature from the service operator, using the public key from
`Configuration.leaf_public_key`, over the following structure:

~~~ tls-presentation
struct {
  opaque label<0..2^8-1>;
  uint32 version;
  opaque value<0..2^32-1>;
} UpdateTBS;
~~~

The `label` field contains the label being updated, `version` contains the new
version, and `value` contains the same contents as `UpdateValue.value`. Clients
MUST successfully verify this signature before consuming `UpdateValue.value`.


# User Operations

The basic user operations are organized as a request-response protocol between a
user and the Transparency Log operator.

Users MUST retain the most recent `TreeHead` they've successfully
verified as part of any query response, and populate the `last` field of any
query request with the `tree_size` from this `TreeHead`. This ensures that all
operations performed by the user return consistent results.

~~~ tls-presentation
struct {
  TreeHead tree_head;
  optional<ConsistencyProof> consistency;
  select (Configuration.mode) {
    case thirdPartyAuditing:
      AuditorTreeHead auditor_tree_head;
  };
} FullTreeHead;
~~~

If `last` is present, then the Transparency Log MUST provide a consistency proof
between the current tree and the tree when it was this size, in the
`consistency` field of `FullTreeHead`.

## Search

<!-- TODO: Update to cover different deployment modes -->

Users initiate a Search operation by submitting a SearchRequest to the
Transparency Log containing the label that they're interested in. Users can
optionally specify a version of the label that they'd like to receive, if not the
most recent one.

~~~ tls-presentation
struct {
  optional<uint32> last;

  opaque label<0..2^8-1>;
  optional<uint32> version;
} SearchRequest;
~~~

In turn, the Transparency Log responds with a SearchResponse structure:

~~~ tls-presentation
struct {
  FullTreeHead full_tree_head;

  SearchProof search;
  opaque opening<16>;
  UpdateValue value;
} SearchResponse;
~~~

Users verify a search response by following these steps:

1. Evaluate the search proof in `search` according to the steps in
   {{proof-combined-tree}}. This will produce a verdict as to whether the search
   was executed correctly and also a candidate root value for the tree. If it's
   determined that the search was executed incorrectly, abort with an error.
2. With the candidate root value for the tree, verify the given `FullTreeHead`.
3. Verify that the commitment in the terminal search step opens to
   `SearchResponse.value` with opening `SearchResponse.opening`.

Depending on the deployment mode of the Transparency Log, the `value` field may
or may not require additional verification, specified in {{update-format}},
before its contents may be consumed.

## Update

Users initiate an Update operation by submitting an UpdateRequest to the
Transparency Log containing the new label and value to store.

~~~ tls-presentation
struct {
  optional<uint32> last;

  opaque label<0..2^8-1>;
  opaque value<0..2^32-1>;
} UpdateRequest;
~~~

If the request passes application-layer policy checks, the Transparency Log adds
a new label-version pair to the prefix tree, followed by adding a new entry to
the log tree with the associated value and updated prefix tree root. It returns
an UpdateResponse structure:

~~~ tls-presentation
struct {
  FullTreeHead full_tree_head;

  SearchProof search;
  opaque opening<16>;
  UpdatePrefix prefix;
} UpdateResponse;
~~~

Users verify the UpdateResponse as if it were a SearchResponse for the most
recent version of `label`. To aid verification, the update response
provides the `UpdatePrefix` structure necessary to reconstruct the
`UpdateValue`.

## Monitor

<!-- TODO: Update to cover different deployment modes -->

Users initiate a Monitor operation by submitting a MonitorRequest to the
Transparency Log containing information about the labels they wish to monitor.

~~~ tls-presentation
struct {
  opaque label<0..2^8-1>;
  uint32 highest_version;
  uint64 entries<0..2^8-1>;
} MonitorLabel;

struct {
  optional<uint32> last;

  MonitorLabel owned_labels<0..2^8-1>;
  MonitorLabel contact_labels<0..2^8-1>;
} MonitorRequest;
~~~

Users include each of the labels that they own in `owned_labels`. If the
Transparency Log is deployed with Contact Monitoring (or simply if the user
wants a higher degree of confidence in the log), they also include any labels
they've looked up in `contact_labels`.

Each `MonitorLabel` structure contains the label being monitored in `label`,
the highest version of the label that the user has observed in `highest_version`,
and a list of `entries` in the log tree corresponding to the keys of the map
described in {{monitoring-the-tree}}.

The Transparency Log verifies the MonitorRequest by following these steps, for
each `MonitorLabel` structure:

1. Verify that the requested labels in `owned_labels` and `contact_labels` are all
   distinct.
2. Verify that the user owns every label in `owned_labels`, and is allowed (or was
   previously allowed) to lookup every label in `contact_labels`, based on the
   application's policy.
3. Verify that the `highest_version` for each label is less than or equal to the
   most recent version of each label.
4. Verify that each `entries` array is sorted in ascending order, and that all
   entries are within the bounds of the log.
5. Verify each entry lies on the direct path of different versions of the label.

If the request is valid, the Transparency Log responds with a MonitorResponse
structure:

~~~ tls-presentation
struct {
  uint32 version;
  VRFProof vrf_proofs<0..2^8-1>;
  ProofStep steps<0..2^8-1>;
} MonitorProof;

struct {
  FullTreeHead full_tree_head;
  MonitorProof owned_proofs<0..2^8-1>;
  MonitorProof contact_proofs<0..2^8-1>;
  InclusionProof inclusion;
} MonitorResponse;
~~~

The elements of `owned_proofs` and `contact_proofs` correspond one-to-one with
the elements of `owned_labels` and `contact_labels`. Each `MonitorProof` in
`contact_proofs` is meant to convince the user that the label they looked up is
still properly included in the log and has not been surreptitiously concealed.
Each `MonitorProof` in `owned_proofs` conveys the same guarantee that no past
lookups have been concealed, and also proves that `MonitorProof.version` is the
most recent version of the label.

The `version` field of a `MonitorProof` contains the version that was used for
computing the binary ladder, and therefore the highest version of the label that
will be proven to exist. The `vrf_proofs` field contains VRF proofs for
different versions of the label, starting at the first version that's
different between the binary ladders for `MonitorLabel.highest_version` and
`MonitorProof.version`.

The `steps` field of a `MonitorProof` contains the proofs required to update the
user's monitoring data following the algorithm in {{monitoring-the-tree}}. That is, each
`ProofStep` of a `MonitorProof` contains a binary ladder for the version
`MonitorProof.version`. The steps are provided in the order that they're
consumed by the monitoring algorithm. If same proof is consumed by the
monitoring algorithm multiple times, it is provided in the `MonitorProof`
structure only the first time.

For `MonitorProof` structures in `owned_labels`, it is also important to prove
that `MonitorProof.version` is the highest version of the label available. This
means that such a `MonitorProof` must contains full binary ladders for
`MonitorProof.version` along the frontier of the log. As such, any `ProofStep`
under the `owned_labels` field that's along the frontier of the log includes a
full binary ladder for `MonitorProof.version` instead of a regular binary
ladder. For additional entries on the frontier of the log that are to the right
of the leftmost frontier entry already provided, an additional `ProofStep` is
added to `MonitorProof`. This additional `ProofStep` contains only the proofs of
non-inclusion from a full binary ladder.

Users verify a MonitorResponse by following these steps:

1. Verify that the lengths of `owned_proofs` and `contact_proofs` are the same
   as the lengths of `owned_labels` and `contact_labels`.
2. For each `MonitorProof` structure, verify that `MonitorProof.version` is
   greater than or equal to the highest version of the label that's been
   previously observed.
3. For each `MonitorProof` structure, evalute the monitoring algorithm in
   {{monitoring-the-tree}}. Abort with an error if the monitoring algorithm detects that
   the tree is constructed incorrectly, or if there are fewer or more steps
   provided than would be expected (keeping in mind that extra steps may be
   provided along the frontier of the log, if a `MonitorProof` is a member of
   `owned_labels`).
4. Construct a candidate root value for the tree by combining the
   `PrefixProof` and commitment of `ProofStep`, with the provided inclusion
   proof.
5. With the candidate root value, verify the provided `FullTreeHead`.

Some information is omitted from MonitorResponse in the interest of efficiency,
due to the fact that the user would have already seen and verified it as part of
conducting other queries. In particular, VRF proofs for different versions of
each label are not provided, given that these can be cached from the
original Search or Update query.


# Security Considerations

<!-- TODO Security -->
<!-- TODO Say that transport layer should be encrypted, provide auth -->


# IANA Considerations

This document requests the creation of the following new IANA registries:

* KT Ciphersuites ({{kt-ciphersuites}})

All of these registries should be under a heading of "Key Transparency",
and assignments are made via the Specification Required policy {{!RFC8126}}. See
{{de}} for additional information about the KT Designated Experts (DEs).

RFC EDITOR: Please replace XXXX throughout with the RFC number assigned to
this document

## KT Ciphersuites

~~~ tls-presentation
uint16 CipherSuite;
~~~

<!-- TODO -->

## KT Designated Expert Pool {#de}

<!-- TODO -->


--- back

# Acknowledgments
{:numbered="false"}

<!-- TODO acknowledge. -->
