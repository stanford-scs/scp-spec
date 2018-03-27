% Title = "The Stellar Consensus Protocol (SCP)"
% abbrev = "scp"
% category = "exp"
% docName = "draft-mazieres-dinrg-scp-01"
% ipr= "trust200902"
% area = "Internet"
% workgroup = ""
% keyword = ["consensus"]
%
% date = 2018-03-22T00:00:00Z
%
% [[author]]
% initials="N."
% surname="Barry"
% fullname="Nicolas Barry"
% #role="editor"
% organization = "Stellar Development Foundation"
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
% organization = "Stellar Development Foundation"
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

SCP is an open Byzantine agreement protocol resistant to Sybil
attacks.  It allows Internet infrastructure stakeholders to reach
agreement on a series of values without unanimous agreement on what
constitutes the set of important stakeholders.  A big differentiator
from other Byzantine agreement protocols is that, in SCP, nodes
determine the composition of quorums in a decentralized way:  each
node selects quorum slices they consider large or important enough to
speak for the whole network, and a quorum must satisfy the
requirements of each of its members.

{mainmatter}

# Introduction

Various aspects of Internet infrastructure depend on irreversible and
transparent updates to data sets such as authenticated mappings
[cite Li-Man-Watson draft].  Examples include public key certificates
and revocations, transparency logs [@?RFC6962], and preload lists for
HSTS [@?RFC6797] and HPKP [@?RFC7469].

The Stellar Consensus Protocol (SCP) specified in this draft allows
Internet infrastructure stakeholders to collaborate in applying
irreversible transactions to public state.  SCP is an open Byzantine
agreement protocol that resists Sybil attacks by allowing individual
parties to specify minimum quorum memberships in terms of specific
trusted peers.  Each participants chooses combinations of peers on
which to depend such that these combinations can be trusted in
aggregate.  The protocol guarantees safety so long as these dependency
sets transitively overlap and contain sufficiently many honest nodes
correctly obeying the protocol.

Though bad configurations are theoretically possible, several
analogies provide an intuition for why transitive dependencies overlap
in practice.  For example, given multiple entirely disjoint
Internet-protocol networks, people would have no trouble agreeing on
the fact that network containing the world's top web sites is _the_
Internet.  Such a consensus can hold even without unanimous agreement
on what constitute the world's top web sites.  Similarly, if network
operators listed all the ASes from whom they would accept peering or
transit (at the right price), the transitive closures of these sets
would contain significant overlap, even without unanimous agreement on
the "tier-1 ISP" designation.  Finally, while different browsers and
operating systems have slightly different lists of valid certificate
authorities, there is significant overlap in the sets, so that systems
requiring hypothetical validation from "all CAs" would be unlikely to
diverge.

A more detailed abstract description of SCP and its rationale,
including an English-language proof of safety, is available in
[@?SCP].  In particular, that reference shows that a necessary
property for safety, termed _quorum intersection despite ill-behaved
nodes_, is sufficient to guarantee safety under SCP, making SCP
optimally safe against malicious nodes for any given configuration.

This document specifies the end-system logic and wire format of the
messages.

# The Model

This section describes the configuration and input/output values of
the consensus protocol.

## Configuration

Each participant or _node_ in the SCP protocol has a digital signature
key and is named by the corresponding public key, which we term a
`NodeID`.

Each node also selects one or more sets of nodes (each of which
includes itself) called _quorum slices_.  A quorum slice represents a
large or important enough set of peers that the node selecting the
quorum slice believes the slice collectively speaks for the whole
network.

A _quorum_ is a non-empty set of nodes containing at least one quorum
slice of each of its members.  For instance, suppose `v1` has the
single quorum slice `{v1, v2, v3}`, while each of `v2`, `v3`, and `v4`
has the single quorum slice `{v2, v3, v4}`.  In this case, `{v2, v3,
v4}` is a quorum, because it contains a slice for each member.  On the
other hand `{v1, v2, v3}` is not a quorum, because it does not contain
a quorum slice for `v2` or `v3`.  The smallest quorum including `v1`
in this example is the set of all nodes `{v1, v2, v3, v4}`.

Unlike traditional Byzantine agreement protocols, nodes in SCP only
care about quorums to which they belong themselves (and hence which
contain at least one of their quorum slices).  Intuitively, this is
what protects nodes from Sybil attacks.  In the example above, if `v3`
deviates from the protocol, maliciously inventing 96 Sybils `v5, v6,
..., v100`, the honest nodes' quorums' will all still include one
another, ensuring that `v1`, `v2`, and `v4` continue to agree on
output values.

Every message in the SCP protocol specifies the sender's quorum
slices.  Hence, by collecting messages, a node dynamically learns what
constitutes a quorum and can decide when a particular message has been
sent by a quorum to which it belongs.  (Again, nodes to not care about
quorums to which they do not belong themselves.)

## Input and output

The SCP protocol outputs a series of _values_ each associated with a
consecutively numbered _slot_.  The goal is for all non-faulty nodes
to reach _agreement_ on the output value of each slot.  5 seconds
after SCP outputs the value for one slot, nodes restart the protocol
to select a value for the next slot.  The time elapsed between the
completion of SCP on one slot and its initiation for the next is used
to construct candidate values for the next slot, as well as to
amortize the cost of consensus over an arbitrary-sized batch of
operations.

From SCP's perspective, values are just opaque byte arrays whose
interpretation is left to higher-level applications.  However, SCP
requires a _combining function_ that reduces multiple candidate values
into a single _composite_ value.  When nodes nominate multiple values
for a slot, SCP nodes invoke this combining function to converge on a
single composite value.  By way of example, in an application where
values consist of sets of transactions, the combining function could
take the union of transaction sets.  Alternatively, if values
represent a timestamp and a set of transactions, the combining
function might pair the highest nominated timestamp with the
transaction set that has the highest hash value.

Not every node can influence the composite value for any given slot.
Whether or not a node can affect a given slot depends on how many
other nodes include that node in their quorum slices, as well as how
the current slot number and history perturb a cryptographic hash of
nodes' public keys.

# Protocol

The protocol consists of exchanging digitally-signed messages
specifying nodes' quorum slices.  The format of all messages is
specified using XDR [@!RFC4506].  In addition to quorum slices,
messages compactly convey votes on sets of conceptual statements.  The
core technique of voting with quorum slices is termed _federated
voting_.  We next describe federated voting, then detail protocol
messages in the following subsection.

## Federated voting

Federated voting allows each node to confirm some statement.  Not
every attempt at federated voting may succeed--an attempt to vote on
some statement `a` may get stuck, with the result that nodes can
confirm neither `a` nor its negation `!a`.  However, when a node
succeeds in confirming a statement `a`, federated voting guarantees
two things:

1. No two well-behaved nodes will confirm contradictory statements in
   any configuration and failure scenario in which any protocol can
   guarantee safety for the two nodes (i.e., quorum intersection for
   the two nodes holds despite ill-behaved nodes).

2. If a node that is guaranteed safety by #1 confirms a statement `a`,
   and that node is a member of one or more quorums consisting
   entirely of well-behaved nodes, then eventually every member of
   every such a quorum will also confirm `a`.

Intuitively, these conditions are key to ensuring agreement among
nodes as well as a weak form of liveness (absence of stuck states)
that is compatible with the FLP impossibility result [@?FLP].

As a node `v` collects federated voting messages for a given message
`m`, two thresholds trigger state transitions in `v` depending on the
message.  We define these thresholds as follows:

* _quorum threshold_:  When every member of a quorum to which `v`
  belongs (including `v` itself) has issued message `m`

* _blocking threshold_:  When at least one member of each of `v`'s
  quorum slices (a set that does not necessarily include `v` itself)
  has issued message `m`

Each node `v` can send three types of message with respect to a
statement `a` during federated voting:  _vote_ `a`, _accept_ `a`, and
_vote-or-accept_ `a`.

* _vote_ `a` states that `a` is a valid statement and constitutes a
  promise by `v` not to vote for any contradictory message, such as
  `!a`.

* _accept_ `a` says that nodes may or may not come to agree on `a`,
  but if they don't, then the system has experienced a catastrophic
  set of Byzantine failures to the point that no quorum containing `v`
  consists entirely of correct nodes.  (Nonetheless, accepting `a` is
  not sufficient to act on it, as doing so could violate agreement,
  which is worse than merely getting stuck from a lack of correct
  quorum.)

* _vote-or-accept_ `a` is the disjunction of the above two messages.
  A node implicitly sends such a message if it sends either _vote_ `a`
  or _accept_ `a`.  Where it is inconvenient and unnecessary to
  differentiate between the two cases, a node can explicitly send a
  _vote-or-accept_ message.

(#fig:voting) illustrates the federated voting process.  A node `v`
votes for a valid statement `a` that doesn't contradict past votes `v`
has cast or statements `v` has accepted.  When the _vote_ message
reaches quorum threshold, the node accepts `a`.  In fact, `v` accepts
`a` if the_vote-or-accept_ message has reaches quorum threshold, as
some nodes may accept `a` without first voting for it.  Specifically,
a node that cannot vote for `a` because it has voted for its negation
`!a` still accepts `a` when the message _accept_ `a` reaches blocking
threshold (meaning assertions about `!a` have no hope of reaching
quorum threshold barring catastrophic Byzantine failure).  Finally, if
and when the message _accept_ `a` reaches quorum threshold, `v` has
confirmed `a` and the federated vote has succeeded.  In effect, the
_accept_ messages constitute a second vote on the fact that the
initial vote messages succeeded.

{#fig:voting}
                    "vote-or-accept a"          "accept a"
                         reaches                 reaches
                     quorum threshold        quorum threshold
                    +-----------------+     +-----------------+
                    |                 |     |                 |
                    |                 V     |                 V
                 +-----------+     +-----------+     +-----------+
      a is +---->|  voted a  |     |accepted a |     |confirmed a|
     valid |     +-----------+     +-----------+     +-----------+
           |           |                 ^
    +-----------+      |                 | "accept a" reaches
    |uncommitted|------+-----------------+ blocking threshold
    +-----------+      |
           |           |
           |     +-----------+
           +---->|  voted !a |
                 +-----------+
Figure: Federated voting process

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
However, arbitrary predicates on sets cannot be encoded concisely.
Instead we specify quorum slices as any set of k-of-n members, where
each of the n members can either be an individual node ID, or,
recursively, another k-of-n set.

~~~~~ {.xdr}
// supports things like: A,B,C,(D,E,F),(G,H,(I,J,K,L))
// only allows 2 levels of nesting
struct SCPQuorumSet
{
    uint32 threshold;            // the k in k-of-n
    PublicKey validators<>;
    SCPQuorumSet1 innerSets<>;
};
struct SCPQuorumSet1
{
    uint32 threshold;            // the k in k-of-n
    PublicKey validators<>;
    SCPQuorumSet2 innerSets<>;
};
struct SCPQuorumSet2
{
    uint32 threshold;            // the k in k-of-n
    PublicKey validators<>;
};
~~~~~

As described in (#message-envelopes), every protocol message is paired
with a cryptographic hash of the sender's `SCPQuorumSet` and digitally
signed.  Inner protocol messages described in the next few sections
should be understood to be received in alongside such a quorum slice
specification and digital signature.

## Nomination

For each slot, the SCP protocol begins in a nomination phase whose
goal is to devise one or more candidate output values for the
consensus protocol.  Nodes send nomination messages contain a
monotonically growing set of values in the following format:

~~~~~ {.xdr}
struct SCPNomination
{
    Value votes<>;      // X
    Value accepted<>;   // Y
};
~~~~~

The `votes` and `accepted` sets are disjoint; any value that is
eligible for for both sets is placed only in the `accepted` set.

`votes` consists of candidate values nominated by the sender.  Each
node progresses through a series of nomination _rounds_ in which it
potentially grows the set of values in its own `votes` field by adding
the contents of the `votes` and `accepted` fields of `SCPNomination`
messages received from a growing set of peers.  In round `n` of slot
`i`, each node determines the additional peers whose nominated values
it should vote for as follows:

* Let `Gi(m) = SHA-256(i || output[i-1] || m)`, where `output[i-1]` is
  the consensus output of slot i-1 or the zero-byte value for slot 0.
  (Recall values are encoded as an XDR opaque vector, with a 32-byte
  length followed by bytes zero-padded to a multiple of 4 bytes.)
  Treat the output of `Gi` as a 256-bit binary number in big-endian
  format.

* For each peer `v`, define `weight(v)` as the faction of quorum
  slices containing `v`.

* Define the set of nodes `neighbors(n)` as the set of nodes v for
  which `Gi("N" || n || v) < 2^{256} * weight(v)`.

* Define `priority(n, v)` as `Gi("P" || n || v)`.

For each round `n` until nomination has finished, a node picks the
available peer `v` with the highest value of `priority(n, v)` from
among the nodes in `neighbors(n)`.  It then adds any values in `v`'s
`votes` set to its own `votes` set.  If the node itself has the
highest priority--meaning `v` is actually the node doing the
nomination--then it adds to `votes` a candidate value supplied by the
higher-level application of SCP.  Round `n` lasts for `2+n` seconds,
after which, if nomination has not finished (see below), a node
proceeds to round `n+1`.

If a particular valid value `x` reaches quorum threshold in the
messages sent by peers (meaning that every node in a quorum contains
`x` either in the `votes` or the `accepted` field), then the node at
which this happens moves `x` from its `votes` field to its `accepted`
field and broadcasts a new `SCPNomination` message.  Similarly, if `x`
reaches blocking threshold in a node's peers' `accepted` field
(meaning every one of a node's quorum slices contains at least one
node with `x` in its `accepted` field), then the node adds `x` to its
`accepted` field (removing it from `votes` if applicable).  These two
cases correspond to the two conditions for entering the `accepted`
state in (#fig:voting).

A node finishes the nomination phase whenever any value `x` reaches
quorum threshold in the `accepted` fields.  Following the terminology
of (#federated-voting), this condition corresponds to the node
confirmed some value `x` as nominated.  A node that has finished the
nomination phase stops adding new values to its `votes` set.  However,
the node continues adding new values to `accepted` as appropriate.
Doing so may lead to more values becoming confirmed nominated in the
background.

## Prepare messages

Once the nomination process is complete at a node (meaning at least
one candidate value is confirmed nominated), the node moves on to the
prepare phase of the protocol, which uses federated voting to abort
and commit ballots.  A ballot is a pair, consisting of a counter (`n`)
and candidate value (`x`):

~~~~~ {.xdr}
struct SCPBallot
{
    uint32 counter; // n
    Value value;    // x
};
~~~~~

`counter` beings at 1 for each slot, and is incremented to higher
numbers if the first ballot fails to reach consensus on an output
value.  `value` is initially chosen as the output of the deterministic
combining function applied to all values that have been confirmed
nominated for the slot.  Note, however, that this output may change in
subsequent ballots if more values have been confirmed nominated in the
background.  Moreover, as described below, the set of nominated values
becomes irrelevant to as soon any ballot has made it beyond a certain
point in the protocol before failing.

Ballots are totally ordered with `n` more significant than `x`.
Hence, we write `b1 < b2` to mean that either `b1.counter <
b2.counter` or `b1.counter == b2.counter && b1.value < b2.value`.
(Values are compared lexicographically, as a strings of octets.)

For a given ballot `b`, nodes employ federated voting to chose between
two contradictory outcomes:  _commit_ `b` and _abort_ `b`.  The
protocol requires aborting large numbers of ballots
simultaneously--specifically all ballots less than some target ballot
that contain a different value than the target ballot.  We use a
compact notation to encode such a set of _abort_ statements:

* `prepare(b)` encodes an _abort_ statement for every ballot less than
  `b` containing a different value from `b`.  Using set-builder
  notation, `prepare(b) = { b1 | b1 < b AND b1.value != b.value }`.

* _vote_ `prepare(b)` encodes a set of _vote_ messages for every
  _abort_ statement in `prepare(b)`.

* Similarly, _accept_ `prepare(b)` and _vote-or-accept_ `prepare(b)`
  encode sets of _accept_ and _vote-or-accept_ messages for every
  _abort_ statement in `prepare(b)`.

An important invariant in the protocol is that a node must confirm
`prepare(b)` before sending a _vote_ message for _commit_ `b`.

Nodes send the following concrete message in the prepare phase:

~~~~~ {.xdr}
struct SCPPrepare
{
    SCPBallot ballot;         // b
    SCPBallot *prepared;      // p
    SCPBallot *preparedPrime; // p'
    uint32 nC;                // c.n
    uint32 nH;                // h.n
};
~~~~~

The fields have the following meaning:

* `ballot` - the current ballot, determined as discussed below.  This
  field conveys the set of conceptual messages _vote-or-abort_
  `prepare(ballot)`.

* `prepared` - the highest ballot `b` for which the sender has
  accepted `prepared(b)`.  More specifically, `prepared` contains the
  highest `b` for which one of the following two conditions has been
  met:
    * The sending node is part of a quorum in which, for each node,
      there is some ballot `b1` in the `ballot`, `prepared`, or
      `preparedPrime` field of its latest `SCPPrepare` message such
      that `b1.value == b.value && b1.counter >= b.counter`, or
    * There is a set reaching blocking threshold at the sender such
      that for each node in the set, there is some ballot `b1` in the
      `prepared`, or `preparedPrime` field of its latest `SCPPrepare`
      message such that `b1.value == b.value && b1.counter >=
      b.counter`.
  If no such ballot `b` exists, then `prepared` is `NULL`.

* `preparedPrime` - the highest ballot `b` satisfying the same
  criteria as `prepared` with the additional constraint that
  `preparedPrime.value != prepared.value`.  `preparedPrime` is `NULL`
  if no such ballot exists.

* `nH` - the counter from the highest ballot `<n,x>` for which the
  sender is in a quorum in which each member has sent an `SCPPrepare`
  message with one of the following properties (or `nH = 0` if no such
  ballot exists):
    * The `prepared` field contains `<n',x>` where `n' >= n`, or
    * The `preparedPrime` field contains `<n',x'>` where `n' >= n` and
      `x'` can be any value.

* `nC` - the counter for the lowest ballot the sender is attempting to
  confirm (see below), otherwise 0.

The `value` field `x` of each ballot is selected as follows.  If `nH =
0`, then use the deterministic combination function on all values that
have made it all the way through the nomination protocol to produce a
candidate value.  Otherwise, use the value `x` associated with the
ballot that produced `nH`.

The `counter` field `n` of each ballot is selected as follows.

* Initially, `n = 1`.

* If a node is still sending `SCPPrepare` or `SCPConfirm` messages
  (meaning it has not yet output a value for a particular slot), then
  a node arms a timer whenever `n` reaches quorum threshold of ballots
  (meaning the node is a member of a quorum in which each member is
  sending explicit or implicit `SCPPrepare` messages with ballot
  `counter` >= `n`).

* If the timer fires, a node increments `n` and determines a new value
  depending on the latest state of the nomination protocol and `nH`,
  as discussed above.

* If a `counter` value greater than `n` ever reaches a blocking
  threshold, then a node immediately disables any pending timer and
  increases `n` to the blocking `counter` value, recomputing `value`
  in the usual way.

The value `nC` is maintained based on an internally-maintained "commit
ballot" `c`, where initially `c = cNULL = <0, NULL>` (for some
arbitrary or invalid value `NULL`).

* If either `prepared` or `preparedPrime` has `counter` greater than
  `c`'s and a different `value`, then reset `c = cNULL`.

* If `c = cNULL` and the ballot determining `nH` hasn't been
  aborted by `prepared` or `preparedPrime`, then set `c` to the lowest
  ballot containing the value of that (`nH`) ballot that is between
  `ballot` and the `nH` ballot.

* If some ballot with the same value reaches quorum threshold betwen
  `c` and the `nH` ballot, move to the confirm phase.

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
{{reference.flp.xml}}

{backmatter}


