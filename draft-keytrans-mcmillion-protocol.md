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

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the TLS presentation language {{!RFC8446}} to describe the
structure of protocol messages, but does not require the use of a specific
transport protocol. As such, implementations do not necessarily need to transmit
messages according to the TLS format, and can chose whichever encoding method
best suits their application.

TODO Say that transport layer should be encrypted etc

# Tree Construction

KT relies on two combined hash tree structures: log trees and prefix trees. This
section describes the operation of both at a high level and the way that they're
combined. More precise algorithms for computing the intermediate and root values
of the trees are given in {{cryptographic-computations}}.

Both types of trees consist of *nodes* which have a byte string as their
*value*. A node is either a *leaf* if it has no children, or a *parent* if it
has either a *left child* or a *right child*. A node is the *root* of a tree if
it has no parents, and an *intermediate* if it has both children and parents.
Nodes are *siblings* if they share the same parent.

The *descendants* of a node are that node, its children, and the descendants of
its children. A *subtree* of a tree is the tree given by the descendants of a
node, called the *head* of the subtree.

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

Log trees initially consist of a single leaf node. New leaves are added to the
right-most edge of the tree along with a single parent node, to construct the
left-balanced binary tree with `n+1` leaves.

While leaves contain arbitrary data, the value of a parent node is always the
hash of the combined values of its left and right children.

Log trees are powerful in that they can provide both *inclusion proofs*, which
demonstrate that a leaf is included in a log, and *consistency proofs*, which
demonstrate that a new version of a log is an extension of a past version of the
log.

An inclusion proof is given by providing the copath values of a leaf. The proof
is verified by hashing together the leaf with the copath values and checking
that the result equals the root value of the log. Consistency proofs are a more
general version of the same idea. With a consistency proof, the prover provides
the minimum set of intermediate node values from the current tree that allows
the verifier to compute both the old root value and the current root value. An
algorithm for this is given in section 2.1.2 of {{!RFC6962}}.

## Prefix Tree

TODO

## Combined Tree


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

## Commitment

As discussed in {{preserving-privacy}}, commitments are stored in the leaves of
the log tree and correspond to updated key-value pairs. Commitments are computed
with HMAC {{RFC2104}}, using the hash function specified by the ciphersuite. To
produce a new commitment, the application generates a random 16 byte value
called `opening` and computes:

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

TODO

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
  uint64 timestamp;
  opaque signature<0..2^16-1>;
} TreeHead;
~~~

where `tree_size` counts the number of entries in the log tree and `timestamp`
is the time that the structure was generated in milliseconds since the Unix
epoch. If the Transparency Log is deployed with Third-party Management then the
public key used to verify the signature belongs to the third-party manager;
otherwise the public key used belongs to the service operator.

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
  uint64 timestamp;
  opaque root_value<Hash.Nh>;
} TreeHeadTBS;
~~~


# Security Considerations

TODO Security


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

TODO

## KT Designated Expert Pool {#de}

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
