% Title = "The Stellar Consensus Protocol (SCP)"
% abbrev = "scp"
% category = "exp"
% docName = "draft-mazieres-dinrg-scp-00"
% ipr= "trust200902"
% area = "Internet"
% workgroup = ""
% keyword = ["consensus"]
%
% date = 2018-03-05T00:00:00Z
%
% [[author]]
% initials="N."
% surname="Barry"
% fullname="Nickolas Barry"
% #role="editor"
% organization = "Stellar"
%   [author.address]
%   email = "nicolas@stellar.org"
%   [author.address.postal]
%   street = "170 Capp St., Suite A"
%   city = "San Francisco, CA"
%   code = "94110"
%   country = "US"
%
% [[author]]
% initials="D."
% surname="Mazieres"
% fullname="David Mazieres"
% #role="editor"
% organization = "Stanford University"
%   [author.address]
%   email = "dm@uun.org"
%   [author.address.postal]
%   street = "353 Serra Mall, Room 290"
%   city = "Stanford, CA"
%   code = "94305"
%   country = "US"
%
% [[author]]
% initials="J."
% surname="McCaleb"
% fullname="Jed McCaleb"
% #role="editor"
% organization = "Stellar"
%   [author.address]
%   email = "jed@stellar.org"
%   [author.address.postal]
%   street = "170 Capp St., Suite A"
%   city = "San Francisco, CA"
%   code = "94110"
%   country = "US"
%
% [[author]]
% initials="S."
% surname="Polu"
% fullname="Stanislas Polu"
% #role="editor"
% organization = "Stripe Inc."
%   [author.address]
%   email = "stan@stripe.com"
%   [author.address.postal]
%   street = "185 Berry Street, Suite 550"
%   city = "San Francisco, CA"
%   code = "94107"
%   country = "US"

.# Abstract

SCP is a protocol for Internet-level Byzantine agreement.

{mainmatter}

# Introduction

Various aspects of Internet infrastructure depend on irreversible and
transparent updates to data sets such as authenticated mappings
[cite Li-Man-Watson draft].  Examples include public key certificates
mapping domain names to public keys, transparency logs [@?RFC6962],
HSTS [@?RFC6797] and HPKP [@?RFC7469] preload lists.

The Stellar Consensus Protocol (SCP) specified in this draft allows
various Internet infrastructure stakeholders to collaborate in
applying irreversible transactions to public state.  SCP is an open
Byzantine agreement protocol that resists Sybil attacks by allowing
individual parties to specify minimum quorum memberships in terms of
specific trusted peers.  Hence, participants can choose specific peers
they know to be trustworthy and enjoy safety so long as the peers they
depend on honestly obey the protocol.

An more detailed abstract description of the SCP protocol and
English-language proof of safety is available in [@?SCP].  This
document specifies the end-system logic and wire format of the
messages.

# The Model

## Configuration

Each participant or _node_ in the SCP protocol has a digital signature
key and is named by the corresponding public key, which we term a
`NodeID`.

Each node also selects one or more groups of nodes (each of which
includes itself) called _quorum slices_.  A quorum slice represents a
large or important enough set of peers that the node selecting the
quorum slice believes the slice collectively speaks for the whole
network.

A _quorum_ is a non-empty set of nodes containing at least one quorum
slice of each of its members.  For instance, suppose `v1` has the
single quorum slice `{v1, v2, v3}`, and each of `v2`, `v3`, and `v4`
has the single quorum slice `{v2, v3, v4}`.  In this case, `{v2, v3,
v4}` is a quorum, because it contains a slice for each member.  On the
other hand `{v1, v2, v3}` is not a quorum, because it does not contain
a quorum slice for `v2` or `v3`.  The smallest quorum including `v1`
in this example is the set of all nodes `{v1, v2, v3, v4}`.

Unlike traditional Byzantine agreement protocols, nodes in SCP only
care about quorums to which they belong themselves (and hence which
contain at least one of their quorum slices).  Intuitively, this is
what protects nodes from Sybil attacks.  In the example above, if `v3`
deviates from the protocol maliciously invents 96 Sybils `v5, v6, ...,
v100`, the honest nodes quorums will all still include one another.

Every message in the SCP protocol specifies the sender's quorum
slices.  Hence, by collecting messages a node dynamically learns what
constitutes a quorum, and can decide when a particular message has
been sent by a quorum to which it belongs.  (Again, nodes to not care
about quorums to which they do not belong themselves.)

## Input and output

The SCP protocol outputs a series of _values_ each associated with a
consecutively numbered _slot_.  5 seconds after SCP outputs the value
for one slot, nodes restart the protocol to select a value for the
next slot.  The time between slots is used to construct candidate
values for the next slot, as well as to amortize the cost of consensus
over an arbitrary-sized batch of operations.

There must exist a deterministic combining function that reduces
multiple candidate values into a single _composite_ value.  Hence,
should nodes nominate multiple values, they can still
deterministically converge on the same composite value.  Note that
this does not mean that every node has a say in the composite value
for every slot.  Whether or not a node can in practice influence a
given slot depends on how many other nodes include that node in their
quorum slices as well as how the current slot number perturbs a
cryptographic hash of nodes' public keys.

# Protocol

The protocol consists of exchanging digitally-signed messages
containing nodes' quorum slices.  When multiple nodes send the same
messages, two thresholds have significance throughout the protocol for
each node `v`.

* quorum threshold:  When `v` node is a member of a quorum in which
  every member (including `v`) has issued some signed statement.

* blocking threshold:  When every one of `v`'s quorum slices has a
  member issuing some signed statement (which doesn't necessarily
  include `v`).

Message formats are specified using XDR [@!RFC4506].

## Basic types

SCP employs 32- and 64-bit integers, as defined below.

~~~~~ {.xdr}
typedef unsigned int uint32;
typedef int int32;
typedef unsigned hyper uint64;
typedef hyper int64;
~~~~~

SCP uses the SHA-256 cryptograhpic hash function [@!RFC6234], and
represents hash values as a simple array of 32 bytes.

~~~~~ {.xdr}
typedef opaque Hash[32];
~~~~~

SCP employes the Ed25519 digital signature algorithm [@!RFC8032].  For
purposes of cryptographic agility, however, public keys are
represented as a union type that can be compatibly extended with other
key types.

~~~~~ {.xdr}
typedef opaque uint256[32];

enum PublicKeyType
{
    PUBLIC_KEY_TYPE_ED25519 = 0
};

union PublicKey switch (PublicKeyType type)
{
case PUBLIC_KEY_TYPE_ED25519:
    uint256 ed25519;
};

// variable size as the size depends on the signature scheme used
typedef opaque Signature<64>;
~~~~~

Nodes are public keys, while values are simply opaque arrays of byes.

~~~~~ {.xdr}
typedef PublicKey NodeID;

typedef opaque Value<>;
~~~~~

## Quorum slices

Theoretically a quorum slice can be an arbitrary set of sets of nodes.
However, it would not be possible to represent an arbitrary predicate
on sets concisely.  Instead we specify quorum slices as any set of
k-of-n members, where each of the n members can either be an
individual node ID, or, recursively, another k-of-n predicate.

~~~~~ {.xdr}
// supports things like: A,B,C,(D,E,F),(G,H,(I,J,K,L))
// only allows 2 levels of nesting
struct SCPQuorumSet
{
    uint32 threshold;            // the n in k-of-n
    PublicKey validators<>;
    SCPQuorumSet innerSets<>;
};
~~~~~

## Nomination

Nomination message are sent in the first phase of each slot in an
attempt to select a value on which to try to agree.

~~~~~ {.xdr}
struct SCPNomination
{
    Hash quorumSetHash; // D
    Value votes<>;      // X
    Value accepted<>;   // Y
};
~~~~~

As with all messages, the nomination protocol includes a SHA-256 hash
of the sender's XDR-encoded SCPQuorumSet.  It also includes two
monotonically-growing sets of values.

`votes` consists of candidate values nominated by the sender.  Each
node progresses through a series of nomination _rounds_ in which it
potentially adds more and more values to `votes` by adding `votes` for
valid values received from a growing set of peers to its own set of
`votes`.  Each node computes the set of peers whose nomination `votes`
it should echo based on its quorum slices as follows for round `n` of
slot `i`:

* Let `Gi(m) = SHA-256(i || output[i-1] || m)`, where `output[i-1]` is
  the consensus output of slot i-1 or the zero-byte value for slot 0.
  (Recall values are encoded as an XDR opaque vector, with a 32-byte
  length followed by bytes padded to a multiple of 4 bytes.)  Treat
  the output of `Gi` as a 256-bit binary number in big-endian format.

* For each peer `v`, define `weight(v)` as the faction of quorum
  slices containing `v`.

* Define the set of nodes `neighbors(n)` as the set of nodes v for
  which `Gi("N" || n || v) < 2^{256}-1 * weight(v)`.

* Define `priority(n, v)` as `Gi("P" || n || v)`.

For each round `n` until nomination starts, a node picks the available
peer `v` with the highest value of priority in the given round from
among the nodes in `neighbors(n)`, and adds any values in that node's
`votes` set to its own `votes` set.  Round `n` lasts for `2+n`
seconds, after which if nomination has not finished (see below), a
node proceeds to round `n+1`.

If a particular valid value `x` reaches quorum threshold in the
messages sent by peers (meaning that every node in a quorum contains
`x` either in the `votes` or the `accepted` field), then a node moves
`x` from its `votes` field to its `accepted` field and broadcasts a
new `SCPNomination` message.  Similarly, if `x` reaches blocking
threshold in a node's peers' `accepted` field (meaning every one of a
node's quorum slices contains at least one node with `x` in its
`accepted` field), then the node adds `x` to its `accepted` field
(removing it from `votes` if applicable).

The nomination process terminates at a node when any value `x` reaches
quorum threshold in the `accepted` fields, meaning every node in a
quorum that includes the node has sent a signed `SCPNomination`
message in which the `accepted` field contains `x`.  Once nomination
terminates, nodes stop adding new values to their `votes` sets, but
potentially continues adding new values to `accepted` as previously
described.

## Prepare messages

Once the nomination process is complete, meaning at least one
`accepted` value has reached quorum threshold, a node moves on to the
balloting phase of the protocol.  A ballot is a pair of a counter
(`n`) and candidate value (`x`):

~~~~~ {.xdr}
struct SCPBallot
{
    uint32 counter; // n
    Value value;    // x
};
~~~~~

The counter beings at 1, and is incremented to higher numbers if
ballot 1 fails to reach consensus on an output value.  The value `x`
begins as the output of the deterministic combining function on all
values that have achieved quorum threshold in the `accepted` fields of
`SCPNomination` messages (meaning all values for which the local node
is part of a quorum in which every node includes the value in the
`accepted` field of an `SCPNomination` message).  Note, however, that
`x` may change in subsequent ballots if more values reach this quorum
threshold (since nomination keeps running in the background, even
though a node does not add new values to its `votes` field in
`SCPNomination`).  Moreover, as described below, `x` may change if a
particular ballot makes it a certain way through the protocol before
failing.

Once they have selected `x` for a particular ballot, nodes attempt to
`prepare` the ballot by sending the following message:

~~~~~ {.xdr}
struct SCPPrepare
{
    Hash quorumSetHash;       // D
    SCPBallot ballot;         // b
    SCPBallot* prepared;      // p
    SCPBallot* preparedPrime; // p'
    uint32 nC;                // c.n
    uint32 nH;                // h.n
};
~~~~~

The fields have the following meaning:

* `quorumSetHash` - the SHA-256 hash of the sending node's
  `SCPQuorumSet` structure.

* `ballot` - the current ballot as described above.

* `prepared` - the highest ballot `<n,x>` for which one of the
  following two thresholds has been met:
    * The sending node is part of a quorum in which each node included
      `<n',x>` for `n' >= n` in one of the `ballot`, `prepared` or
      `preparedPrime` fields of an `SCPPrepare` message.
    * Every one of the sending node's quorum slices contains at least
      one node that sent an `SCPPrepare` containing some `<n',x>` with
      `n' >= n` in either the `prepared` or `preparedPrime` field.
  If no such ballot exists, then `prepared` is `NULL`.

* `preparedPrime` - the highest ballot `<n,x>` that satisfies the same
  criteria as `prepared`, but has a different value `x` from that in
  the `prepared` field.  `NULL` if no such ballot exists.

* `nH` - the counter from the highest ballot `<n,x>` for which the
  sender is in a quorum in which each member has sent an `SCPPrepare`
  message with one of the following properties:
    * The `prepared` field contains `<n',x>` where `n' >= n`, or
    * The `preparedPrime` field contains `<n',x'>` where `n' >= n` and
      `x'` can be any value.

## Confirm messages

~~~~~ {.xdr}
struct SCPConfirm
{
    SCPBallot ballot;   // b
    uint32 nPrepared;   // p.n
    uint32 nCommit;     // c.n
    uint32 nH;          // h.n
    Hash quorumSetHash; // D
};
~~~~~

This message conveys an implicit `SCPPrepare` message with the
following fields:

* `quorumSetHash` - same
* `ballot` - `<infinity, x>` where `x` is the value in the
  `SCPConfirm message's `ballot`.
* `prepared` - same
* `preparedPrime` - `NULL`
* `nC` - same
*  `nH` - `infinity`

(Note that `infinity` here just represents 2^{32} -- an integer one
greater than the largest representable value on the wire.)

## Externalize messages

~~~~~ {.xdr}
struct SCPExternalize
{
    SCPBallot commit;         // c
    uint32 nH;                // h.n
    Hash commitQuorumSetHash; // D used before EXTERNALIZE
} externalize;
~~~~~

An `SCPExternalize` message conveys two implicit `SCPConfirm`
messages.  The first one has the following fields:

* `ballot` - `<infinity,x>` where `x` is the value from the
  `SCPExternalize` ballot.
* `nPrepared` - `infinity`
* `nCommit` - the `counter` field from the `SCPExternalize` messages
  `commit` field.
* `nH` - `infinity`
* `quorumSetHash` - the `commitQuorumSetHash` from the
  `SCPExternalize` message.

The second one has the same fields as the first, except for the last
two:

* `nH` - the `nH` value from the `SCPExternalize` message
* `quorumSetHash` - a trivial `SCPQuorumSet` that contains a single
  slice whose only member is the sender of the `SCPExternalize`
  message


## Message envelopes

In order to provide full context for each signed message, all signed
messages are part of an `SCPStatement` union type that includes the
`slotIndex` naming the slot to which the message applies, as well as
the `type` of the message.  A signed message and its signature are
packed together in an `SCPEnvelope` structure.

~~~~~ {.xdr}
enum SCPStatementType
{
    SCP_ST_PREPARE = 0,
    SCP_ST_CONFIRM = 1,
    SCP_ST_EXTERNALIZE = 2,
    SCP_ST_NOMINATE = 3
};

struct SCPStatement
{
    NodeID nodeID;    // v (node signing message)
    uint64 slotIndex; // i

    union switch (SCPStatementType type)
    {
    case SCP_ST_PREPARE:
        SCPPrepare prepare;
    case SCP_ST_CONFIRM:
        SCPConfirm confirm;
    case SCP_ST_EXTERNALIZE:
        SCPExternalize externalize;
    case SCP_ST_NOMINATE:
        SCPNomination nominate;
    }
    pledges;
};

struct SCPEnvelope
{
    SCPStatement statement;
    Signature signature;
};
~~~~~

# Security considerations

If nodes do not pick quorum slices well, the protocol will not be
safe.

# Acknowledgments

The Stellar development foundation supported development of the
protocol and produced the first production deployment of SCP.  The
IRTF DIN group including Dirk Kutscher, Sydney Li, Colin Man, Melinda
Shore, and Jean-Luc Watson helped with the framing and motivation for
this specification.

{{reference.scp.xml}}

{backmatter}


