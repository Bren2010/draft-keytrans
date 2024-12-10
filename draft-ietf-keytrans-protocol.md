---
title: "Key Transparency Protocol"
category: std

docname: draft-ietf-keytrans-protocol-latest
submissiontype: IETF
v: 3
area: SEC
workgroup: KEYTRANS Working Group
keyword:
  - key transparency
venue:
  group: "Key Transparency"
  type: "Working Group"
  mail: "keytrans@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/keytrans/"
  github: "ietf-wg-keytrans/draft-protocol"
  latest: "https://ietf-wg-keytrans.github.io/draft-protocol/draft-ietf-keytrans-protocol.html"

author:
 -
    fullname: "Brendan McMillion"
    email: "brendanmcmillion@gmail.com"
 -
    fullname: "Felix Linker"
    email: "linkerfelix@gmail.com"

normative:

informative:
  CONIKS:
    target: https://eprint.iacr.org/2014/1004
    title: "CONIKS: Bringing Key Transparency to End Users"
    date: 2014-04-27
    author:
      - name: Marcela S. Melara
      - name: Aaron Blankstein
      - name: Joseph Bonneau
      - name: Edward W. Felten
      - name: Michael J. Freedman
  SEEMLess:
    target: https://eprint.iacr.org/2018/607
    title: "SEEMless: Secure End-to-End Encrypted Messaging with less trust"
    date: 2018-06-18
    author:
      - name: Melissa Chase
      - name: Apoorvaa Deshpande
      - name: Esha Ghosh
      - name: Harjasleen Malvai
  OPTIKS:
    target: https://eprint.iacr.org/2023/1515
    title: "OPTIKS: An Optimized Key Transparency System"
    date: 2023-10-04
    author:
      - name: Julia Len
      - name: Melissa Chase
      - name: Esha Ghosh
      - name: Kim Laine
      - name: Radames Cruz Moreno
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

KT uses a *prefix tree* to establish a mapping between each label-version pair
and a commitment to the label's value at that version. Every time the prefix
tree changes, its new root hash and the current timestamp are stored in a *log
tree*. The benefit of the prefix tree is that it is easily searchable, and the
benefit of the log tree is that it can easily be verified to be append-only. The
data structure powering KT combines a log tree and a prefix tree, and is called
the *combined tree*.

This section describes the operation of both prefix and log trees at a high
level and the way that they're combined. More precise algorithms for computing
the intermediate and root values of the trees are given in
{{cryptographic-computations}}.

## Terminology

Trees consist of
*nodes*, which have a byte string as their *hash value*. A node is either a
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

The *size* of a tree or subtree is defined as the number of leaf nodes it
contains.

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
added to the right-most edge of the tree along with a single parent node to
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
value.

However, the protocol described in this document generally doesn't provide pure
inclusion or consistency proofs to verifiers. Instead, it relies on the more
general idea of providing the smallest number of intermediate node values that are
necessary to allow a verifier to compute the tree's root hash from
intermediate and leaf values that they already have. The verifier may already have
intermediate and leaf values because they were provided by the Transparency Log as
part of its response to the current query, or because they were retained from
responses to previous queries. This pattern of providing proofs ensures that
there is minimal on-the-wire redundancy.

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

Prefix trees store a mapping between search keys and their corresponding values,
with the ability to efficiently prove that a search key's value was looked up
correctly.

Each leaf node in a prefix tree represents a specific mapping from search key to
value, while each parent node represents some prefix which all search keys in
the subtree headed by that node have in common. The subtree headed by a parent's
left child contains all search keys that share its prefix followed by an
additional 0 bit, while the subtree headed by a parent's right child contains
all search keys that share its prefix followed by an additional 1 bit.

The root node, in particular, represents the empty string as a prefix. The
root's left child contains all search keys that begin with a 0 bit, while the right
child contains all search keys that begin with a 1 bit.

A prefix tree can be searched by starting at the root node and moving to the
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
convince verifiers that nothing has been removed from a log that was present in a
previous version. However, log trees can't be efficiently searched without
downloading the entire log. Prefix trees are efficient to search and can provide
inclusion proofs to convince verifiers that the returned search results are
correct. However, it's not possible to efficiently prove that a new version of a
prefix tree contains the same data as a previous version with only new values
added.

In the combined tree structure, based on {{Merkle2}}, a prefix tree
contains a mapping where each label-version pair has a search key, and the
search key's value is a cryptographic commitment to the label's new contents. A
log tree contains a record of each version of the prefix tree and
the timestamp that it was created. With some caveats, this combined structure
supports both efficient consistency proofs and can be efficiently searched.

Note that, although the Transparency Log maintains a single logical prefix tree,
each modification of this tree results in a new root hash which is then stored
in the log tree. And, as part of the protocol, the Transparency Log is often
required to perform lookups in different versions of the prefix tree. Therefore,
when instructions refer to "looking up a label-version pair in the prefix tree
at a given log entry," this means performing the search in the specific version
of the prefix tree whose root hash is stored at that log entry (where a "log
entry" refers to a leaf of the log tree).

~~~ aasvg
Epoch:          n                               n+1
                                ==>
Log tree:       o                                o
           o----+----.                o----------+----------o
          / \         \              / \            .-------+-----.
         /   \         |            /   \          /               \
        /_____\   [T_n; h(PT_n)]   /_____\   [T_n; h(PT_n)]   [T_n+1; h(PT_n+1)]
~~~
{: title="An example evolution of the combined tree structure. At every epoch, a
new leaf is added to a log tree containing the timestamp T_n, and the new
prefix tree root hash PT_n."}

# Fixed-Version Searches

When searching the combined tree structure described in {{combined-tree}}, users
do a binary search for the first log entry where the prefix tree at that entry
contains the desired label-version pair. This ensures that all users will check
the same or similar entries when searching for the same label, allowing for
efficient client-side auditing of the Transparency Log. The binary search uses
an *implicit binary search tree* constructed over the leaves of the log tree
(distinct from the structure of the log tree itself), which allows the search to
have a complexity logarithmic in the number of the log's leaves.

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

Following the structure of this binary tree when executing searches means that
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

It is also frequently useful to refer to the **frontier** of the log. The
frontier consists of the root node of a search, followed by the entries produced
by repeatedly calling `right` until reaching the last entry of the log. Using
the same example of a search where the log has 50 entries, the frontier would be
entries: 31, 47, 49.

## Binary Ladder

When executing searches, the implicit tree described in
{{implicit-binary-search-tree}} is navigated according to a binary search. At
each individual log entry, the binary search needs to determine whether it
should move left or right. That is, it needs to determine, out of the set of
label-version pairs stored in the prefix tree, whether the highest version of a
label that's present at a given log entry is greater than, equal to, or less
than the target version.

A **binary ladder** is a series of lookups in a single log entry's prefix tree
that determines the highest version of a label that exists, and it proceeds as
follows:

1. First, version `x` of the label is looked up, where `x` is a consecutively
   higher power of two minus one (0, 1, 3, 7, ...). This is repeated until the
   first proof of non-inclusion is produced.
2. Once the first proof of non-inclusion is produced, a binary search is
   conducted between the highest version that was proved to be included, and the
   version that was proved to not be included. Each step of the binary search
   produces either a proof of inclusion or non-inclusion, which guides the
   search left or right, until it terminates.

As an example, if the highest version of a label is 6, a binary ladder would
consist of the following: proofs of inclusion for versions 0, 1, 3, terminated
by a proof of non-inclusion for version 7. Then, proofs of inclusion for
versions 5 and 6 would conclude the proof that 6 is the highest version
available.

While being able to determine the exact highest version of a label is sometimes
useful (as discussed in {{greatest-version-searches}}), this is not always the case. Users
searching for a specific version of a label only need to know whether
the highest version at a given log entry is greater than or less than their
target version -- not its exact value. A **truncated binary ladder** is one which
stops after the first proof of inclusion for a version greater than or equal to
the target version, or the first proof of non-inclusion for a version less than
the target version.

## Maximum Lifetime

A Transparency Log operator MAY define a maximum lifetime for log entries. If
defined, it MUST be greater than zero seconds. Whether a log entry has surpassed
its maximum lifetime is determined by subtracting the timestamp of the rightmost
log entry from the timestamp of the log entry in question.

A user executing a search may arrive at a log entry which is past its maximum
lifetime by either of two ways. The user may have inspected a log entry which is
**not** expired and decided to recurse to the log entry's left child, which is
expired. Alternatively, the root log entry may be expired, in which case the
user would've started their search at an expired root log entry.

When a user's search proceeds from a log entry which is not expired to a log
entry which is expired, the user is provided with a binary ladder from the
expired log entry as usual. If the user's search would recurse further into the
expired portion of the tree (to the log entry's left child), the search is
aborted. If the user's search would recurse away from the expired portion of the
tree (to the log entry's right child), the user continues as normal.

When the root and potentially multiple frontier log entries are expired, the
user skips to the furthest-right expired frontier log entry without receiving
binary ladders from any of its parents. Similar to the previous case,
the user is provided with a binary ladder from this log entry. If the user
determines that its search would recurse to the left (further into the expired
portion of the tree), it aborts; to the right (into the unexpired portion of the
tree), it continues.

This allows the Transparency Log to prune data which is sufficiently old, as
only a small amount of the log tree and prefix tree outside of the maximum
lifetime need to be retained. Specifically, only a logarithmic number of log
entries that have passed their maximum lifetime will still be needed by users;
the rest can be discarded. Pruning is explained in more detail in TODO.

## Algorithm {#fixed-version-algorithm}

The algorithm for performing a fixed-version search (a search for a specific
version of a label) is described below as a recursive algorithm. It starts with
the root log entry, as defined by the implicit binary search tree, and then
recurses to left or right children, each time starting back at step 1.

1. Verify that the log entry's timestamp is consistent with the timestamps of
   all ancestor log entries. That is, if the log entry is in the ancestor's left
   subtree, then its timestamp is less than or equal to the ancestor's. If the
   log entry is in the ancestor's right subtree, then its timestamp is greater
   than or equal to the ancestor's.
2. If the log entry has surpassed its maximum lifetime and is on the frontier,
   determine whether its right child has also surpassed its maximum lifetime. If
   so, recurse to the right child; otherwise, continue to step 3. Note that a
   right child always exists, as the rightmost log entry cannot exceed its
   maximum lifetime by definition.
3. Obtain a truncated binary ladder from the current log entry for the target
   version. Verify that the binary ladder terminates in a way that is consistent
   with previously-inspected log entries. Specifically, verify that it indicates
   a maximum version greater than or equal to any log entries to the left, and
   less than or equal to any log entries to the right.
4. If the binary ladder stops short of proving the target version is present,
   recurse to the log entry's right child. Otherwise, check if the log entry has
   surpassed its maximum lifetime. If so, abort the search with an error
   indicating that the desired version of the label has expired and is no longer
   available. If not, recurse to the log entry's left child. If, in either case,
   the child doesn't exist:
5. This largely concludes the search. However, there are some additional
   technicalities to address. First, it's possible for the binary search to
   conclude successfully even if the label-version pair that the user is
   interested in has expired. Out of the log entries touched by the binary
   search, identify which log entry was first to contain the desired
   label-version pair. If it is a log entry that is past its maximum lifetime,
   abort the search and return an error to the user.
6. It's also possible at this point that a commitment to the contents of the
   desired label-version pair has not been provided by the Transparency Log.
   This can happen, for example, if multiple versions of a label were inserted
   in a single epoch and the binary ladder skipped the target version. If this
   has happened, obtain a search proof for the target label-version pair from
   the prefix tree in the first log entry to contain it (identified in step 5).
7. Finally, if the binary search failed to find a log entry containing the
   desired label-version pair, or if the search proof in step 6 returned a proof
   of non-inclusion, return an error to the user indicating that the version of
   the label does not exist. Otherwise, terminate the search successfully.

The most important goal of this algorithm is correctly identifying the first log
entry that contains the target label-version pair. The fundamental purpose for
doing this is to make monitoring more efficient for the label owner. If a label
has a large number of versions, it can become prohibitively expensive for its
owner to repeatedly check that every single version is represented correctly in
multiple log entries. Instead, the label owner can check that the version was
created correctly in the one epoch where it was first added and then enforce
that binary searches for that version will always converge back to that same log
entry.

Once users successfully complete the search and the specified verification, they
move on to opening the target label-version pair's commitment.


# Greatest-Version Searches

Users often wish to search for the "most recent" version, or the greatest
version, of a label. Unlike searches for a specific version, label owners
regularly verify that their most recent version is correctly represented in the
log. This enables a simpler, more efficient approach to searching.

{{reasonable-monitoring-window}} and {{distinguished-log-entries}} define the
concept of a **distinguished** log entry, which is a specific log entry that
label owners are required to check for correctness. To perform a
greatest-version search, users start at the rightmost distinguished log entry.
This log entry will always be on the frontier of the log and will never be past
its maximum lifetime. If there is no distinguished log entry yet, users start at
the root instead.

Users receive a non-truncated binary ladder from their starting log entry. This
identifies the greatest version of the label that it contains. Users then
proceed down the remainder of the frontier: the starting log entry's right
child, the right child's right child, and so on until reaching the last log
entry. From each of these log entries, the user also receives a non-truncated
binary ladder identifying the greatest version of the label that each contains.

As in the previous section, users verify that the log entry timestamps and the
greatest versions identified in each log entry each represent a
monotonically-increasing series. If this verification is successful, users move
on to opening the commimtent for whichever label-version pair was identified as
the greatest.


# Monitoring the Tree

As new entries are added to the log tree, the search path that's traversed to
find a specific version of a label may change. New intermediate nodes may be
established between the search root and the log entry, or a new search root may be
created. The goal of monitoring a label is to efficiently ensure that, when
these new parent nodes are created, they're created correctly such that searches
for the same versions continue converging to the same entries in the log.

Monitoring is performed both by the users that own a label (meaning they are the
authoritative source for the label's content) and the users that lookup a label.
Owners monitor their labels to ensure that past (expected) versions of a label
are still correctly stored in the log and that no new (unexpected) versions have
been added. Users that looked up a label may sometimes need to monitor it
afterwards to ensure that the version they observed isn't later concealed by the
Transparency Log.

## Reasonable Monitoring Window

Label owners MUST monitor their labels regularly, ensuring that past versions of
the label are still correctly represented in the log and that any new versions
of the label are permissible (alerting the user if not). Service Providers
define a duration, referred to as the **Reasonable Monitoring
Window** (RMW), which is the frequency with which the Transparency Log generally
expects label owners to perform monitoring. The log entry maximum lifetime, if
defined, MUST be greater than the RMW.

**Distinguished** log entries are chosen such that there is roughly one per
every interval of the RMW. If a user looks up a label, either through a
fixed-version or greatest-version search, and finds that the first log entry
containing their desired label-version pair is to the right of the rightmost
distinguished log entry, they MUST regularly monitor the label-version pair
until its monitoring path intersects a distinguished log entry. That is, until a
new distinguished log entry is established to its right and the two log entries
are verified to be consistent. The purpose of this monitoring is to ensure that
the label-version pair is not removed or obscured by the Transparency Log before
the label owner has had an opportunity to detect it.

However, if a user looks up a label and finds that the first log entry
containing the label-version pair is either a distinguished log entry or to the
left of any distinguished log entry, they are not required to monitor it
afterwards. The only state that would be retained from the query would be the
tree head, as discussed in {{updating-views-of-the-tree}}.

"Regular" monitoring SHOULD be performed at least as frequently as the RMW and
MUST (if at all possible) happen more frequently than the log entry maximum
lifetime.

## Distinguished Log Entries

Distinguished log entries are chosen according to the following recursive
algorithm:

1. Take as input: a log entry, the timestamp of a log entry to its left, and the
   timestamp of a log entry to its right.
2. If the right timestamp minus the left timestamp is less than the Reasonable
   Monitoring Window, terminate the algorithm. Otherwise:
3. Assert that the log entry is distinguished.
4. If the given log entry has a left child in the implicit binary search tree,
   then recurse to its subtree by executing this algorithm with: the given log
   entry's left child, the given left timestamp, and the timestamp of the given
   log entry.
5. If the given log entry has a right child, then recurse to its right subtree
   by executing this algorithm with: the given log entry's right child, the
   timestamp of the given log entry, and the given right timestamp.

While the algorithm is recursive, it is initiated with these parameters: the
root node in the implicit binary search tree, the timestamp of the leftmost log
entry (regardless of its expiration status), and the timestamp of the rightmost
log entry. Note that step 2 is specifically "less than" and not "less than or
equal to"; this ensures correct behavior when the RMW is zero.

This process for choosing distinguished log entries ensures that they are
**regularly spaced**. Having irregularly-spaced distinguished log entries risks
either overwhelming label owners with a large number of them, or delaying
consensus between users by having arbitrarily few. Distinguished log entries
must reliably occur at roughly the same interval as the Reasonable Monitoring
Window regardless of variations in the tree's growth rate or other factors.

This process also ensures that distinguished log entries are **stable**. Once a
log entry is chosen to be distinguished, it will never stop being distinguished.
This is important because it means that, if a user looks up a label and checks
consistency with some distinguished log entry, this log entry can't later avoid
inspection by the label owner by losing its distinguished status.

## Lower-Bound Binary Ladder

Similar to the algorithm for searching the tree, the algorithm for monitoring
the tree requires a way to prove that the highest version of a label stored in a
particular log entry is greater than or equal to a target version. Since we
already know that the target version exists, we only need to prove that there
has not been an unexpected downgrade. As such, the steps of the binary ladder
for monitoring are slightly different than for search:

1. First, version `x` of the label is looked up, where `x` is consecutively higher
   powers of two minus one (0, 1, 3, 7, ...). This is repeated until `x` is the
   largest such value less than or equal to the target version.
2. Second, the largest `x` that was looked up is retained, and consecutively
   smaller powers of two are added to it until it equals the target version.
   Each time a power of two is added, this version of the label is looked up.

This could be equivalently computed as a search binary ladder, excluding lookups
for any versions greater than the target.

## Algorithm

To monitor a given label, users maintain a small amount of state: a map from a
position in the log to a version counter. The version counter is the highest
version of the label that's been proven to exist at that log position. Users
initially populate this map by setting the position of the first log entry to
contain the label-version pair they've looked up to map to that version. A map
may track several different versions of a label simultaneously if a user has
been shown different versions of the same label.

To update this map, users receive the most recent tree head from the server and
follow these steps for each entry in the map, from rightmost to leftmost log
entry:

1. Determine if the log entry has been declared distinguished. If so, leave
   the position-version pair in the map and move on to the next map entry.
2. Compute the log entry's direct path, in terms of the implicit binary search
   tree, based on the current tree size. Remove all entries that are to the left
   of the log entry. If any of the remaining log entries are distinguished,
   terminate the list just after the first distinguished log entry.
3. If the computed list is empty, leave the position-version pair in the map
   and move on to the next map entry.
4. For each log entry in the computed list, from left to right:
   1. Check if this log entry already has an entry in the map with a greater
      version. If so, this version of the label no longer needs to be monitored.
      Remove the current position-version pair (the one with the lesser version)
      from the map and move on to the next map entry.
   2. Receive and verify a lower-bound binary ladder, described in
      {{lower-bound-binary-ladder}}, from this log entry for the version
      currently in the map. This proves that, at the indicated log entry, the
      highest version present is greater than or equal to the
      previously-observed version.
   3. If the above check fails, return an error to the user. Otherwise, remove
      the current position-version pair from the map and replace it with a new
      one, with the position of the log entry the binary ladder came from.

Once the map entries are updated according to this process, the final step of
monitoring is to remove all mappings where the position corresponds to a
distinguished log entry.

In summary, monitoring works by progressively moving up the tree as new
intermediate/root nodes are established and verifying that they're constructed
correctly. Once a distinguished log entry is reached and successfully verified,
monitoring is no longer necessary and the relevant entry is removed from the
map.

Note that users can often execute this process with the output of Search or
Update operations for a label without waiting to make explicit Monitor queries.
It is also worth noting that the work required to monitor several versions of
the same label scales sublinearly due to the fact that the direct paths of the
different versions will often intersect. Intersections reduce the total number
of entries in the map and therefore the amount of work that will be needed to
monitor the label from then on.

### Owner Algorithm

If the user owns the label being monitored, they will additionally need to
retain the rightmost distinguished log entry where they've verified that the
most recent version of the label is correct. Users advertise this log entry's
position in their Monitor request and the Transparency Log provides a
non-truncated binary ladder, described in {{binary-ladder}}, demonstrating the
greatest version of the label in a number of subsequent distinguished log
entries. The user inspects each of these for unexpected changes; if there are
none, the user updates their retained state.

Users MUST verify that they receive a binary ladder for the distinguished log
entry immediately following the one they've advertised, and the distinguished
log entry immediately following that one, and so on. The Transparency Log will
provide whichever intermediate timestamps are necessary to demonstrate that this
is the case. To avoid overloading, the Transparency Log MAY limit the number of
distinguished log entries it provides binary ladders for in a single response.

If the user is monitoring the label for the first time since it was created (or
for the first time since they took ownership of it), they advertise no log entry
to the Transparency Log and initialize their state by setting it to be the
distinguished log entry that terminates monitoring in step 4 above. The
Transparency Log provides binary ladders for additional subsequent distinguised
log entries.


# Updating Views of the Tree

The previous sections on searching and monitoring always operate on a single,
fixed version of the combined tree. However, as users interact with the tree
over time, they will see many different root hashes as the contents of the tree
changes. It's necessary for users to guarantee that the root hashes they observe
are consistent with respect to a few important properties:

- If root hash A is shown after root hash B, then root hash A contains all the
  same log entries as B (with some new ones potentially added).
- If the rightmost log entry of the tree with root hash A has timestamp
  T_a, and the rightmost log entry of the tree with root hash B has timestamp
  T_b, then T_a is greater than or equal to T_b.
  - Furthermore, all log entries between the rightmost log entry of A and the
    rightmost log entry of B have monotonically increasing timestamps.

The first property is necessary to ensure that the Transparency Log never
removes a log entry after showing it to a user, as this would allow the
Transparency Log to remove evidence of its own misbehavior. The second two
properties ensure that all users have a consistent view of when each portion of
the tree was created. Disagreement on when portions of the tree were created is
functionally a fork and it introduces the same security issues. This is because
users rely on log entry timestamps to decide whether to continue monitoring
certain labels, or which portions of the tree to skip when searching.

To address this, users retain select information about the last tree head
they've observed:

1. The "size" of the log tree, defined as the number of log entries it contains.
2. The root hashes of the "full subtrees" of the log tree. The full subtrees are
   those which are the largest powers of two possible. This allows the
   Transparency Log to provide more space-efficient consistency and inclusion
   proofs than if just the root hash was retained.
3. The timestamps of the log entries along the frontier. As will be described
   below, this will be used to ensure that subsequent log entry timestamps are
   correct.

When users make queries to the Transparency Log, they advertise the size of the
last tree head observed. If the Transparency Log responds with an updated tree
head, it provides additional information in its query response to prove that the
advertised tree head and the more recent one are consistent.

Proving the latter two properties, that newly added log entries have
monotonically increasing timestamps, is done as follows:

- Starting with the log entry that was advertised by the user, compute its
  direct path in the new tree in terms of the implicit binary search tree.
  Provide the log entry timestamp and prefix tree root hash for each element
  of the direct path that's to the right of the advertised log entry.
- Exactly one of these log entries will lie on the new tree's frontier. From
  this log entry, compute the remainder of the frontier. That is, compute the
  log entry's right child, and the right child's right child, and so on. Provide
  the timestamps and prefix tree root hashes for these log entries as well.

Users verify that the presented timestamps represent a monotonic series. While
this only requires users to verify a logarithmic number of the newly added log
entries' timestamps, it guarantees that two users with overlapping views of the
tree will detect any violations. Note that retaining only the rightmost log
entry's timestamp would be sufficient for this purpose, but users retain the
timestamps of all log entries along the frontier. The additional timestamps are
retained so that the Transparency Log doesn't have to provide them again in
future queries if they've exceeded their maximum lifetime.

Proving the first property, that the tree is append-only, is done by providing a
consistency proof combined with an inclusion proof for the log entries that were
provided to prove the latter two properties. That is, the minimum set of
intermediate hashes from the log tree are provided that are necessary to compute
the new root hash from the given log entries and the full subtrees retained by
the user.

Additionally, the Transparency Log defines two durations: how old of a view of
the tree is acceptable, and how far ahead of the current time a view of the tree
may claim to be. Users verify that the rightmost log entry's timestamp falls
within this range according to their local clock.

For users which have never interacted with the Transparency Log before and don't
have a previous tree head to advertise, the Transparency Log simply provides the
timestamps and prefix tree root hashes of the log entries on the frontier, along
with an inclusion proof. Same as above, the user verifies that the timestamps
are monotonic.


# Ciphersuites

Each Transparency Log uses a single fixed ciphersuite, chosen when the log is
initially created, that specifies the following primitives and parameters to be used for
cryptographic computations:

* A hash algorithm
* A signature algorithm
* A Verifiable Random Function (VRF) algorithm
* `Nc`: The size in bytes of commitment openings
* `Kc`: A fixed string of bytes used in the computation of commitments

The hash algorithm is used for computing the intermediate and root values of
hash trees. The signature algorithm is used for signatures from both the service
operator and the third party, if one is present. The VRF is used for preserving
the privacy of labels. One of the VRF algorithms from {{!RFC9381}} must be
used.

Ciphersuites are represented with the CipherSuite type. The ciphersuites are
defined in {{kt-ciphersuites}}.


# Cryptographic Computations

## Tree Head Signature

The head of a Transparency Log, which represents the log's most recent state, is
represented as:

~~~ tls-presentation
struct {
  uint64 tree_size;
  opaque signature<0..2^16-1>;
} TreeHead;
~~~

where `tree_size` counts the number of leaves of the log tree. If the
Transparency Log is deployed with Third-party Management, then the public key
used to verify the signature belongs to the third-party manager; otherwise the
public key used belongs to the service operator.

The signature itself is computed over a `TreeHeadTBS` structure which
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

  uint64 max_ahead;
  unit64 max_behind;
  uint64 reasonable_monitoring_window;
  optional<uint64> maximum_lifetime;
} Configuration;

struct {
  Configuration config;
  uint64 tree_size;
  opaque root<Hash.Nh>;
} TreeHeadTBS;
~~~

The `max_ahead` and `max_behind` fields contain the maximum amount of time in
milliseconds that a tree head may be ahead of or behind the user's local clock
without being rejected. The `reasonable_monitoring_window` contains the
Reasonable Monitoring Window, defined in {{reasonable-monitoring-window}}, in
milliseconds. If the Transparency Log has chosen to define a maximum lifetime
for log entries, per {{maximum-lifetime}}, this duration in milliseconds is
stored in the `maximum_lifetime` field.

Finally, `Hash.Nh` is the output size of the ciphersuite hash function in bytes.

## Update Format

The updates committed to by the prefix tree contain the new value of a label,
along with additional information depending on the deployment mode of the
Transparency Log. They are serialized as follows:

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

## Commitment

As discussed in {{combined-tree}}, commitments are stored in the leaves of the
prefix tree rather than raw `UpdateValue` structures. Commitments are computed
with HMAC {{!RFC2104}}, using the hash function specified by the ciphersuite. To
produce a new commitment, the application generates a random `Nc`-byte value
called `opening` and computes:

~~~ pseudocode
commitment = HMAC(Kc, CommitmentValue)
~~~

where `Kc` is defined by the ciphersuite and CommitmentValue is specified as:

~~~ tls-presentation
struct {
  opaque opening<Nc>;
  opaque label<0..2^8-1>;
  UpdateValue update;
} CommitmentValue;
~~~

The `label` field of CommitmentValue contains the label being updated and the
`update` field contains the new value for the label. The output value
`commitment` may be published, while `opening` should only be revealed to users
that are authorized to receive the label's contents.

The Transparency Log MAY generate `opening` in a programmatic way. However, it
SHOULD ensure that `opening` can later be deleted and not feasibly recovered.
This preserves the Transparency Log's ability to delete certain information in
compliance with privacy laws.

## Verifiable Random Function

Each label-version pair created will have a unique
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

## Log Tree {#crypto-log-tree}

The leaf and parent nodes of a log tree are serialized as:

~~~ tls-presentation
struct {
  uint64 timestamp;
  opaque prefix_tree<Hash.Nh>;
} LogLeaf;

struct {
  opaque value<Hash.Nh>;
} LogParent;
~~~

The `timestamp` field contains the timestamp that the leaf was created in
milliseconds since the Unix epoch. The `prefix_tree` field contains the new root
hash of the prefix tree after inserting any new desired entries.

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
    return Hash(node.timestamp || node.prefix_tree)
  else if node.type == parentNode:
    return node.value
~~~

where `Hash` denotes the ciphersuite hash function.

## Prefix Tree

The leaf nodes of a prefix tree are serialized as:

~~~ tls
struct {
    opaque vrf_output<VRF.Nh>;
    opaque commitment<Hash.Nh>;
} PrefixLeaf;
~~~

where `vrf_output` is the VRF output for the label-version pair, `VRF.Nh` is the
output size of the ciphersuite VRF in bytes, and `commitment` is the commitment
to the corresponding `UpdateValue` structure.

The parent nodes of a prefix tree are serialized as:

~~~ tls
struct {
  opaque value<Hash.Nh>;
} PrefixParent;
~~~

The value of a parent node is computed by hashing together the values of its
left and right children:

~~~ pseudocode
parent.value = Hash(0x01 ||
                   nodeValue(parent.leftChild) ||
                   nodeValue(parent.rightChild))

nodeValue(node):
  if node.type == emptyNode:
    return 0 // all-zero vector of length Hash.Nh
  else if node.type == leafNode:
    return Hash(0x00 || node.vrf_output || node.commitment)
  else if node.type == parentNode:
    return node.value
~~~


# Tree Proofs

## Log Tree

The inclusion of some leaf in a tree head, or consistency between two tree
heads, is proven by providing the smallest set of intermediate nodes from the
log tree that allows the user to compute the tree's root hash from the leaf or
other intermediate node values they already have. Such a proof is encoded as:

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

Additionally, each `NodeValue` corresponds to a full subtree of the log tree. If
an intermediate node's value needs to be provided to satisfy the verifier, but
the node represents a non-full subtree, the values of the node's descendant full
subtrees are provided instead.

## Prefix Tree

A proof from a prefix tree authenticates that a search was done correctly for a
given search key. Such a proof is encoded as:

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
sorted lexicographically by search key (which is always a VRF output in this
protocol). The `result_type` field of each `PrefixSearchResult` struct indicates
what the terminal node of the search for that value was:

- `inclusion` for a leaf node matching the requested value.
- `nonInclusionLeaf` for a leaf node not matching the requested value. In this
  case, the terminal node's value is provided since it can not be inferred.
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
left/right arrangement dictated by the bits of the search keys, and checking
that the result equals the root value of the prefix tree.

## Combined Tree {#proof-combined-tree}

As users execute the algorithms for searching, monitoring, or updating their
view of the tree, they inspect a series of leaves in the log tree. For some of
these leaves, only the timestamp of the log entry is needed. For other leaves,
both the timestamp and a `PrefixProof` from the log entry's prefix tree are
needed.

The general pattern used for constructing proofs for these algorithms is to
output the timestamp or `PrefixProof` from each log entry in the order that it's
consumed by the algorithm the user is running. For the purpose of this protocol,
the user always executes the algorithm to update its view of the tree, described
in {{updating-views-of-the-tree}}, followed immediately by one of the algorithms
to search or monitor the current tree.

Proofs are encoded as follows:

~~~ tls
struct {
  uint64 timestamps<0..2^8-1>;
  PrefixProof prefix_proofs<0..2^8-1>;
  NodeValue prefix_roots<0..2^8-1>;
} CombinedTreeProof;
~~~

The elements of the `timestamps` field are the timestamps of log entries, and
the elements of the `prefix_proofs` field are search proofs from the prefix
trees at particular log entries. There is no explicit indication as to which log
entry the elements correspond to, as they are provided in the order that they're
meant to be consumed. The elements of the `prefix_roots` field contain, in
left-to-right order, the prefix tree root hashes for any log entries whose
timestamp was provided in `timestamps` but a search proof was not provided in
`prefix_proofs`.

If a log entry's timestamp is referenced multiple times, it is only added to the
`timestamps` the first time. Additionally, when a user advertises a
previously-observed tree size in their request, log entry timestamps which the
user is expected to have retained are excluded from `timestamps`.

The `prefix_proofs` array differs from the `timestamps` array in two important
ways. First, it is possible for the same log entry to have multiple entries in
`prefix_proofs` if it is referenced multiple times. Users MUST verify that all
`PrefixProof` structures corresponding to the same log entry compute the same
prefix tree root hash. Second, while every log entry with a timestamp in
`timestamps` will also be referenced in either `prefix_proofs` or
`prefix_roots`, the inverse is not true. There may be elements of
`prefix_proofs` or `prefix_roots` that correspond to log entries that are not in
`timestamps` if the user is expected to have retained the log entry's timestamp.

Users processing a `CombinedTreeProof` MUST verify that each field contains
exactly the expected number of entries -- no more and no less.

### Updating View

For a user to update their view of the tree, the following is provided:

- If the user has not previously observed a tree head, the timestamp of each log
  entry along the frontier.
- If the user has previously observed a tree head, the timestamps of each log
  entry from the list computed in {{updating-views-of-the-tree}}.

Users verify that the timestamps represent a monotonic series, and that the
rightmost timestamp is within the bounds defined by `max_ahead` and
`max_behind`.

### Fixed-Version Search

For a user to search the combined tree for a specific version of a label, the
following is provided:

- For each log entry touched by the algorithm in {{fixed-version-algorithm}}:
  - The log entry's timestamp.
  - If the log entry has surpassed its maximum lifetime and is on the frontier,
    the right child's timestamp.
  - If it is not the case that the log entry has surpassed its maximum lifetime,
    is on the frontier, and the log entry's right child has also surpassed its
    maximum lifetime, then a `PrefixProof` corresponding to a binary ladder in
    the log entry's prefix tree is provided.
- If the `PrefixProof` from the first log entry to contain the label-version
  pair didn't touch the target version, provide a second `PrefixProof` from this
  log entry specifically looking up the target version.

Users verify the output as specified in {{fixed-version-algorithm}}.

### Greatest-Version Search

For a user to search the combined tree for the greatest version of a label, the
following is provided:

- For each log entry along the frontier, starting from the log entry identified
  in {{greatest-version-searches}}: a `PrefixProof` corresponding to a
  non-truncated binary ladder.

Note that the log entry timestamps are already provided as part of updating the
user's view of the tree, and that no additional timestamps are necessary to
identify the starting log entry.

TODO Add back reduced binary ladder

### Monitor

For a user to monitor a label in the combined tree, the following is provided:

<!--
### Fixed-Version

Searching the combined tree for a specific version of a label follows the
execution of a binary search through the leaves of the log tree, as described in
{{fixed-version-searches}}. It is serialized as follows:

~~~ tls-presentation

struct {
  opaque proof<VRF.Np>;
  opaque commitment<Hash.Nh>;
} BinaryLadderStep;

struct {
  uint64 timestamp;
  PrefixProof prefix_proof; // TODO: May just need to provide root hash
} ProofStep;

struct {
  BinaryLadderStep binary_ladder<0..2^8-1>;
  ProofStep steps<0..2^8-1>;
  InclusionProof inclusion;
} SearchProof;
~~~

Each element of `binary_ladder` corresponds to the steps of the binary ladder
described in {{binary-ladder}}. The `proof` field contains the contains the
output of evaluating the VRF on a specified version of the label, and
`commitment` contains the commitment to the `UpdateValue` of the label-version
pair. The elements of `binary_ladder` are in the same order as they were
computed in {{binary-ladder}}.

Each `ProofStep` structure in `steps` is one log entry that was inspected as
part of the binary search. The first step corresponds to the "middle" leaf of
the log tree (calculated with the `root` function in
{{implicit-binary-search-tree}}). From there, each subsequent step moves left or
right in the tree according to the procedure discussed in
{{fixed-version-searches}}.

The `timestamp` field of a `ProofStep` contains the log entry's timestamp, and
the `prefix_proof` field contains the output of executing a binary ladder in the
version of the prefix tree stored in the log entry. The binary ladder is
truncated, meaning that it stops after the first proof of inclusion for a
version greater than or equal to the target version, or the first proof of
non-inclusion for a version less than the target version.

Furthermore, for  any ladder steps for which a proof of inclusion is expected
and a proof of inclusion was already provided in a previous `ProofStep` for a
log entry to the left of the current one, these ladder steps are omitted from
`prefix_proof`.

Finally, the `inclusion` field of `SearchProof` contains a batch inclusion
proof for all of the leaves accessed by the binary search.

The proof can be verified by the following steps:

1. Verify that the `binary_ladder` array has the expected number of entries and
   compute the VRF output for each label-version pair in the ladder.
2. Follow the algorithm in {{fixed-version-searches}}.
3. Verify that the `steps` array has the exact number of entries necessary to
   execute the binary search (no more or less).
4. Compute the root of each prefix tree represented by a `prefix_proof` and
   combine it with the corresponding `timestamp` to obtain the value of each
   leaf.
5. Combine the leaf values with the proof in `inclusion` to compute a candidate
   value for the root of the log tree. Verify the root is correct.

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

-->


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
  select (Configuration.mode) {
    case thirdPartyAuditing:
      AuditorTreeHead auditor_tree_head;
  };
} FullTreeHead;
~~~

Modifications to a user's state MUST only be persisted once the query response
has been fully verified. Queries that fail full verification MUST NOT modify the
user's protocol state in any way.

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
  opaque proof<VRF.Np>;
  opaque commitment<Hash.Nh>;
} BinaryLadderStep;

struct {
  FullTreeHead full_tree_head;

  BinaryLadderStep binary_ladder<0..2^8-1>;
  CombinedTreeProof search;
  InclusionProof inclusion;

  opaque opening<16>;
  UpdateValue value;
} SearchResponse;
~~~

Each `BinaryLadderStep` structure contains information related to one version of
the label that's in the binary ladder. The `proof` field contains the VRF
proof, and `commitment` contains the commitment to the label's value at that
version. The `binary_ladder` field contains these structures in the same order
that the versions are output by the algorithm in {{binary-ladder}}.

The `search` field contains the output of updating the user's view of the tree
to match `FullTreeHead.tree_head.size` followed by either a fixed-version or
greatest-version search for the requested label, depending on whether `version`
was provided in `SearchRequest` or not.

The `inclusion` field contains a proof of inclusion for all of the log tree
leaves where either a search proof was provided in
`CombinedTreeProof.prefix_proofs` or the prefix tree root hash was provided
directly in `CombinedTreeProof.prefix_roots`. If the user advertised a
previously-observed tree size in `last`, the proof in `inclusion` also functions
as a proof of consistency.

Users verify a search response by following these steps:

1. Compute the VRF output for each version of the label from the proofs in
   `binary_ladder`.
2. Verify the proof in `search` as described in {{proof-combined-tree}}.
3. Compute a candidate root value for the tree from the proof in `inclusion`,
   the hashes of log entries used in `search`, and any previously-retained full
   subtrees of the log tree.
4. With the candidate root value for the tree, verify `FullTreeHead`.
5. Verify that the commitment to the target version of the label opens to
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
the log tree with an updated timestamp and prefix tree root. It returns
an UpdateResponse structure:

~~~ tls-presentation
struct {
  FullTreeHead full_tree_head;

  BinaryLadderStep binary_ladder<0..2^8-1>;
  CombinedTreeProof search;
  InclusionProof inclusion;

  opaque opening<16>;
  UpdatePrefix prefix;
} UpdateResponse;
~~~

Users verify the UpdateResponse as if it were a SearchResponse for the most
recent version of `label`. To aid verification, the update response
provides the `UpdatePrefix` structure necessary to reconstruct the
`UpdateValue`.

<!-- TODO: This could probably be a lot more efficient -->

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
