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

~~~~~ {.xdr}
// supports things like: A,B,C,(D,E,F),(G,H,(I,J,K,L))
// only allows 2 levels of nesting
struct SCPQuorumSet
{
    uint32 threshold;
    PublicKey validators<>;
    SCPQuorumSet innerSets<>;
};
~~~~~

## Messages

~~~~~ {.xdr}
struct SCPBallot
{
    uint32 counter; // n
    Value value;    // x
};

enum SCPStatementType
{
    SCP_ST_PREPARE = 0,
    SCP_ST_CONFIRM = 1,
    SCP_ST_EXTERNALIZE = 2,
    SCP_ST_NOMINATE = 3
};

struct SCPNomination
{
    Hash quorumSetHash; // D
    Value votes<>;      // X
    Value accepted<>;   // Y
};

struct SCPStatement
{
    NodeID nodeID;    // v
    uint64 slotIndex; // i

    union switch (SCPStatementType type)
    {
    case SCP_ST_PREPARE:
        struct
        {
            Hash quorumSetHash;       // D
            SCPBallot ballot;         // b
            SCPBallot* prepared;      // p
            SCPBallot* preparedPrime; // p'
            uint32 nC;                // c.n
            uint32 nH;                // h.n
        } prepare;
    case SCP_ST_CONFIRM:
        struct
        {
            SCPBallot ballot;   // b
            uint32 nPrepared;   // p.n
            uint32 nCommit;     // c.n
            uint32 nH;          // h.n
            Hash quorumSetHash; // D
        } confirm;
    case SCP_ST_EXTERNALIZE:
        struct
        {
            SCPBallot commit;         // c
            uint32 nH;                // h.n
            Hash commitQuorumSetHash; // D used before EXTERNALIZE
        } externalize;
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

# Acknowledgments

{{reference.scp.xml}}

{backmatter}


