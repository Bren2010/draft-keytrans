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

KT allows clients of a service to query the keys of other clients of the same
service. To do so, KT maintains two maps: (i) one associates key-update counters
with key updates, (ii) the other associates anonymized pairs of pseudonyms and
version numbers with key-update counters. When clients query a KT service, they
require a means to authenticate the responses of the KT service. To provide for
this, the KT service maintains a *combined hash tree structure*, which commits
to both these maps. Such a tree hash structure is associated with a *root hash*.
As long as clients use a consistent root hash, they are guaranteed to always
query the same underlying data structure, thus are guaranteed to always see the
same results to their queries.

The combined hash tree structure consists of two types of trees: log trees and
prefix trees. The log tree commits to map (i) and the most recent prefix tree,
the prefix tree commits to map (ii). This section describes the operation of
both at a high level and the way that they're combined. More precise algorithms
for computing the intermediate and root values of the trees are given in
{{cryptographic-computations}}.

Both types of trees commit to a set of index-value pairs. They consist of
*nodes* which have a byte string as their *hash value*. A node is either a
*leaf* (committing to an index-value pair) if it has no children, or a *parent*
if it has either a *left child* or a *right child*. A node is the *root* of a
tree if it has no parents, and an *intermediate* if it has both children and
parents. Nodes are *siblings* if they share the same parent.

The *descendants* of a node are that node, its children, and the descendants of
its children. A *subtree* of a tree is the tree given by the descendants of a
node, called the *head* of the subtree.

The *direct path* of a root node is the empty list, and of any other node is the
concatenation of that node's parent along with the parent's direct path. The
*copath* of a node is the node's sibling concatenated with the list of siblings
of all the nodes in its direct path, excluding the root.

## Log Tree

Log trees are used for storing information in the chronological order that it
was added and are constructed as *left-balanced* binary trees. The log tree that
we utilize here will have incrementing counters as its indices and pairs of
key-updates and prefix-tree hashes as its values. The log tree's leaves are
ordered by their respective index (counter value) ascending from left to right.

A binary tree is *balanced* if its size is a power of two and for any parent
node in the tree, its left and right subtrees have the same size. A binary tree
is *left-balanced* if for every parent, either the parent is balanced, or the
left subtree of that parent is the largest balanced subtree that could be
constructed from the leaves present in the parent's own subtree. Given a list of
`n` items, there is a unique left-balanced binary tree structure with these
elements as leaves. Note also that every parent always has both a left and right
child.

<!-- TODO diagram of example tree -->

Log trees initially consist of a single leaf node with index 0. New leaves are
added to the right-most edge of the tree along with a single parent node, to
construct the left-balanced binary tree with `n+1` leaves.

<!-- TODO diagram of adding a new leaf to a tree (specifically 5 leaves to 6 leaves to show level-skipping) -->

The hash value of a leaf node is a hash of its value, and the hash value of a
parent node is the hash of the combined hash values of its left and right
children.

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

<!-- TODO diagram of inclusion and consistency proofs -->

## Prefix Tree

Prefix trees are used for storing a set of values while preserving the ability
to efficiently produce proofs of membership and non-membership in the set. The
prefix trees that we utilize here will have anonymized pairs of pseudonyms and
version counters as its indices and the log tree's indices as values.

The prefix tree is ordered by the shortest prefixes of its leaves. The prefix
tree's root represents the empty prefix. All indices of the root's left subtree
begin with a 0 bit, all indices of the root's right subtree start with a 1 bit.
More generally speaking. The subtree headed by a parent's left child contains
all values that share its prefix followed by an additional 0 bit, while the
subtree headed by a parent's right child contains all values that share its
prefix followed by an additional 1 bit.

A prefix tree can be searched by starting at the root node, and moving to the
left child if the first bit of an index is 0, or the right child if the first
bit is 1. This is then repeated for the second bit, third bit, and so on until
the search either terminates at a leaf node (which may or may not be for the
desired value), or a parent node that lacks the desired child.

<!-- TODO diagram -->

New index-value pairs are added to the tree by searching it according to the
same process. If the search terminates at a parent without a left or right
child, a new leaf is added as the parent's missing child. If the search
terminates at a leaf for the wrong index, one or more intermediate nodes are
added until the new leaf and the existing leaf would no longer reside in the
same place. That is, until we reach the first bit that differs between the new
index and the existing index.

<!-- TODO diagram of adding new value -->

The hash value of a leaf node is the hash of its value, while the hash value of
a parent node is the hash of the combined hash values of its left and right
children (or a stand-in value when one of the children doesn't exist).

A proof of membership is given by providing the leaf hash value, along with the
hash value of each copath entry along the search path. A proof of non-membership
is given by providing an abridged proof of membership that follows the search
path for the intended value, but ends either at a stand-in value or a leaf for a
different value. In either case, the proof is verified by hashing together the
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

In the combined tree structure, which is based on {{Merkle2}}, a log tree
maintains a record of each time any key's value is updated, while a prefix tree
maintains the set of pseudonym-version pairs. Importantly, the root hash value of the
prefix tree after adding a new pseudonym-version pair is stored in a leaf of the log
tree alongside a privacy-preserving commitment to the update. With some caveats,
this combined structure supports both efficient consistency proofs and can be
efficiently searched.

Note that, although the Transparency Log maintains a single logical prefix tree,
each modification of this tree results in a new root hash, which is then stored
in the log tree. Therefore, when instructions refer to "looking up a key in the
prefix tree at a given log entry," this actually means searching in the specific
version of the prefix tree that corresponds to the root hash stored at that log
entry.


# Searching the Tree

To search the combined tree structure described in {{combined-tree}}, users do a
binary search for the first log entry where the prefix tree at that entry
contains the desired key-version pair. As such, the entry that a user arrives at
through binary search contains the update that they're looking for, even though
the log itself is not sorted.

Following a binary search also ensures that all users will check the same or
similar entries when searching for the same key, which is necessary for the
efficient auditing of a Transparency Log. To maximize this effect, users rely on
an implicit binary tree structure constructed over the leaves of the log tree
(distinct from the structure of the log tree itself).

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
key-version pairs stored in the prefix tree, whether the highest version present
at a given log entry is higher than, equal to, or lower than a target version.

A **binary ladder** is a series of lookups in a single log entry's prefix tree,
which is used to establish a bound on the highest version of a key that's
available at that log entry.

To determine the highest version of a key that's present at a given log entry,
the following lookups are done:

1. First, version `x` of the key is looked up, where `x` is a consecutively
   higher power of two minus one (0, 1, 3, 7, ...). This is repeated until the
   first proof of non-inclusion is produced.
2. Once the first proof of non-inclusion is produced, a binary search is
   conducted between the highest version that was proved to be included, and the
   version that was proved to not be included. Each step of the binary search
   produces either a proof of inclusion or non-inclusion, which guides the
   search left or right, until it terminates.

However, note that it's not always necessary to find the *exact* highest version
of the key that exists at each log entry. It's sufficient just to determine
whether that version is greater than or less than the target version. As such, a
binary ladder consists of this series of lookups, truncated at the point where
further lookups would no longer impact the binary search's decision.

When executing a search in a Transparency Log for a specific version of a key, a
binary ladder is provided for each node on the search path, verifiably guiding
the search toward the log entry where the desired key-version pair was first
inserted (and therefore, the log entry with the desired update).

## Most Recent Version

Often, users wish to search for the "most recent" version of a key. That is, the
key with the highest counter possible.

To determine this, users request a full (non-truncated) binary ladder for each
node on the **frontier** of the log. The frontier consists of the root node of a
search, followed by the entries produced by repeatedly calling `right` until
reaching the last entry of the log. Using the same example of a search where the
log has 50 entries, the frontier would be entries: 31, 47, 49.

For the purpose of finding the highest version possible, inspecting each entry
along the frontier is functionally the same as inspecting only the last entry.
However, inspecting the frontier allows the user to verify that the entire
search path leading to the last entry represents a monotonic series of version
increases, thereby preventing log misbehavior.

Once the user has verified that the frontier lookups are monotonic and
determined the highest version, the user then continues a binary search for this
specific version.

## Monitoring

As new entries are added to the log tree, the search path that's traversed to
find a specific version of a key may change. New intermediate nodes may become
established in between the search root and the leaf, or a new search root may be
created. The goal of monitoring a key is to efficiently ensure that, when these
new parent nodes are created, they're created correctly so that searches for the
same versions continue converging to the same entries in the log.

To monitor a given search key, users maintain a small amount of state: a map
from a position in the log to a version counter. The version counter is the
highest version of the search key that exists in the log at that position. Users
initially populate this map by setting the position of an entry they've looked
up, to map to the version of the key stored in that entry. A map may track
several different versions of a search key simultaneously, if a user has been
shown different versions of the same search key.

To update this map, users receive the most recent tree head from the server and
follow these steps, for each entry in the map:

1. Compute the entry's direct path (in terms of the Implicit Binary Search Tree)
   based on the current tree size.
2. If there are no entries in the direct path that are to the right of the
   current entry, then skip updating this entry (there's no new information to
   update it with).
3. For each entry in the direct path that's to the right of the current entry,
   from low to high:
   1. Obtain a proof from the server that the highest version of the search key
      present in the prefix tree at that entry is greater than or equal to the
      current version.
   2. If the above check was successful, remove the current position-version pair
      from the map and replace it with a position-version pair corresponding to
      the entry in the log that was just checked.

This algorithm progressively moves up the tree as new intermediate/root nodes
are established and verifies that they're constructed correctly. Note that users
can often execute this process with the output of Search or Update operations
for a key, without waiting to make explicit Monitor queries.

It is also worth noting that the work required to monitor several versions of
the same key scales sublinearly, due to the fact that the direct paths of the
different versions will often intersect. Intersections reduce the total number
of entries in the map and therefore the amount of work that will be needed to
monitor the key from then on.

<!-- TODO This algorithm will need to be updated to handle partial knowledge of the
highest version of a key at each entry. -->


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
the privacy of lookup keys. One of the VRF algorithms from {{!RFC9381}} must be
used.

Ciphersuites are represented with the CipherSuite type. The ciphersuites are
defined in {{kt-ciphersuites}}.


# Cryptographic Computations

## Verifiable Random Function

Each version of a search key that's inserted in a log will have a unique
representation in the prefix tree. This is computed by providing the combined
search key and version as inputs to the VRF:

~~~ tls-presentation
struct {
  opaque search_key<0..2^8-1>;
  uint32 version;
} VrfInput;
~~~

## Commitment

As discussed in {{combined-tree}}, commitments are stored in the leaves of the
log tree and correspond to updates of a key's value. Commitments are computed
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
  opaque search_key<0..2^8-1>;
  UpdateValue update;
} CommitmentValue;
~~~

This fixed key allows the HMAC function, and thereby the commitment scheme, to
be modeled as a random oracle. The `search_key` field of CommitmentValue
contains the search key being updated (the search key provided by the user, not
the VRF output) and the `update` field contains the value of the update.

The output value `commitment` may be published, while `opening` should be kept
private until the commitment is meant to be revealed.

## Prefix Tree

The leaf nodes of a prefix tree are serialized as:

~~~ tls
struct {
    opaque key_version<VRF.Nh>;
} PrefixLeaf;
~~~

where `key_version` is the VRF output for the key-version pair, and `VRF.Nh` is
the output size of the ciphersuite VRF in bytes.

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
    return make([]byte, Hash.Nh)
  else if node.type == leafNode:
    return Hash(0x00 || node.key_version)
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
  opaque root_value<Hash.Nh>;
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
    case nonInclusionLeaf:
      PrefixLeaf leaf;
  };
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

The `elements` array consists of the fewest node values that can be hashed
together with the provided leaves to produce the root. The contents of the
`elements` array is kept in left-to-right order: if a node is present in the
root's left subtree, its value must be listed before any values provided from
nodes that are in the root's right subtree, and so on recursively. In the event
that a node is not present, an all-zero byte string of length `Hash.Nh` is
listed instead.

The proof is verified by hashing together the provided elements, in the
left/right arrangement dictated by the search key, and checking that the result
equals the root value of the prefix tree.

## Combined Tree {#proof-combined-tree}

A proof from a combined log and prefix tree follows the execution of a binary
search through the leaves of the log tree, as described in {{combined-tree}}. It
is serialized as follows:

~~~ tls-presentation

struct {
  opaque proof<VRF.Np>;
} VRFProof;

struct {
  uint8 successful;
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

If searching for the most recent version of a key, the most recent version is
provided in `version`. If searching for a specific version, this field is
omitted.

Each element of `vrf_proofs` contains the output of evaluating the VRF on a
different version. The versions chosen correspond to the binary ladder described
in {{binary-ladder}}.

Each `ProofStep` structure in `steps` is one leaf that was inspected as part of
the binary search. The first step corresponds to the "middle" leaf of the log
tree (calculated with the `root` function in {{implicit-binary-search-tree}}).
From there, each subsequent step moves left or right in the tree, based on
previous results.

The `successful` field of a `ProofStep` contains the number of steps of the
binary ladder where the result of looking up the corresponding VRF output in the
prefix tree yields the expected result.

The `prefix_proof` field of a `ProofStep` is the output of searching the prefix
tree for the first `successful+1` steps of the binary ladder, excluding:

- Any ladder entries for which a **proof of inclusion** is expected, and a proof
  of inclusion was already provided in a previous `ProofStep` for a log entry to
  the **left** of the current one.
- Any ladder entries for which a **proof of non-inclusion** is expected, and a
  proof of non-inclusion was already provided in a previous `ProofStep` for a
  log entry to the **right** of the current one.

Whether a ladder entry is expected to produce a proof of inclusion or
non-inclusion, for the purpose of computing this exclusion, includes the
assumption that the ladder entry with index `successful+1` (if it exists) will
produce the opposite result.
<!-- TODO Does this ever actually result in excluding any ladder entries? -->

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


# Update Format

The updates committed to by a combined tree structure contain the new value of a
search key, along with additional information depending on the deployment mode
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

The `value` field contains the new value of the search key.

In the event that third-party management is used, the `prefix` field contains a
signature from the service operator, using the public key from
`Configuration.leaf_public_key`, over the following structure:

~~~ tls-presentation
struct {
  opaque search_key<0..2^8-1>;
  uint32 version;
  opaque value<0..2^32-1>;
} UpdateTBS;
~~~

The `search_key` field contains the search key being updated (the search key
provided by the user, not the VRF output), `version` contains the new key
version, and `value` contains the same contents as `UpdateValue.value`. Clients
MUST successfully verify this signature before consuming `UpdateValue.value`.


# User Operations

The basic user operations are organized as a request-response protocol between a
user and the Transparency Log operator.

<!-- Generally, users MUST retain the most
recent `TreeHead` they've successfully verified as part of any query response,
and populate the `last` field of any query request with the `tree_size` from
this `TreeHead`. This ensures that all operations performed by the user return
consistent results.

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
`consistency` field of `FullTreeHead`. -->

## Search

Users initiate a Search operation by submitting a SearchRequest to the
Transparency Log containing the key that they're interested in. Users can
optionally specify a version of the key that they'd like to receive, if not the
most recent one.

~~~ tls-presentation
struct {
  optional<Consistency> consistency;

  opaque search_key<0..2^8-1>;
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
Transparency Log containing the new key and value to store.

~~~ tls-presentation
struct {
  optional<Consistency> consistency;

  opaque search_key<0..2^8-1>;
  opaque value<0..2^32-1>;
} UpdateRequest;
~~~

If the request passes application-layer policy checks, the Transparency Log adds
the new key-value pair to the log and returns an UpdateResponse structure:

~~~ tls-presentation
struct {
  FullTreeHead full_tree_head;

  SearchProof search;
  opaque opening<16>;
  UpdatePrefix prefix;
} UpdateResponse;
~~~

Users verify the UpdateResponse as if it were a SearchResponse for the most
recent version of `search_key`. To aid verification, the update response
provides the `UpdatePrefix` structure necessary to reconstruct the
`UpdateValue`.


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
