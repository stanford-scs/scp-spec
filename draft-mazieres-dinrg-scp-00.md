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

# The Model

Each participant or node in the SCP protocol has a digital signature
key and is named by the corresponding public key, which we term a
`NodeID`.

# Protocol

XDR [@!RFC4506]

Ed25519 [@!RFC8032]

~~~~~ {.xdr}

typedef unsigned int uint32;
typedef int int32;
typedef unsigned hyper uint64;
typedef hyper int64;

typedef opaque Hash[32];
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



typedef PublicKey NodeID;

typedef opaque Value<>;

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

// supports things like: A,B,C,(D,E,F),(G,H,(I,J,K,L))
// only allows 2 levels of nesting
struct SCPQuorumSet
{
    uint32 threshold;
    PublicKey validators<>;
    SCPQuorumSet innerSets<>;
};

~~~~~


# Security considerations

# Acknowledgments


{backmatter}


