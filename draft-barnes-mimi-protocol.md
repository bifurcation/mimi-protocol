---
title: "More Instant Messaging Interoperability: The Protocol"
abbrev: "MIMI Protocol"
category: info

docname: draft-barnes-mimi-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "bifurcation/mimi-protocol"
  latest: "https://bifurcation.github.io/mimi-protocol/draft-barnes-mimi-protocol.html"

author:
 -
    fullname: Richard L. Barnes
    organization: Cisco
    email: rlb@ipv.sx
 -
    fullname: Matthew Hodgson
    organization: The Matrix.org Foundation C.I.C.
    email: matthew@matrix.org
 -
    fullname: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -
    fullname: Rohan Mahy
    organization: Wire
    email: rohan.mahy@wire.com
 -
    fullname: Travis Ralston
    organization: The Matrix.org Foundation C.I.C.
    email: travisr@matrix.org
 -
    fullname: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com


normative:

informative:


--- abstract

This document specifies the More Instant Messaging Interoperability (MIMI)
transport protocol, which allows users of different messaging providers to
interoperate in group chats (rooms), including to send and receive messages,
share room policy, and add participants to and remove participants from rooms.
MIMI describes messages between providers, leaving most aspects of the
provider-internal client-server communication up to the provider.

--- middle

# Introduction

The goal of the More Instant Messaging Interoperability (MIMI) Working Group is
to enable multiple providers of end-to-end encrypted instant messaging to
interoperate. As described in the MIMI architecture, group chats and direct
messages are described in terms of "rooms". Each MIMI protocol room is hosted at
a single provider (the "hub" provider"), but allows users from different
providers to become participants in the room. The hub provider maintains the
policy and participation for the room, authorizes messages, and is responsible
for message ordering, the participation list, and policy of its rooms. Each
provider also stores initial keying material and consent for its users (who may
be offline).

This document describes the communication between and among different providers
necessary to send and receive encrypted messages, share room policy, and add
participants to and remove participants from rooms. In support of these
functions, the protocol also has primitives to fetch initial keying material,
fetch the current state of the underlying end-to-end encryption protocol for the
room, and request, grant, and reject consent to communicate with users on other
providers.

Messages sent inside each room are end-to-end encrypted using the Messaging
Layer Security (MLS) protocol {{!RFC9420}}, and each room is associated with an
MLS group. MLS also ensures that clients in a room agree on the room policy and
participation. One of the core principles of this protocol is that at all times,
the clients of the end-to-end encryption protocol (the MLS clients who are in an
MLS group) all need to correspond to users who are in the associated room.

## Known Gaps

In this version of the document, we have tried to capture enough concrete
functionality to enable basic application functionality, while defining enough
of a protocol framework to indicate how other necessary functionality.  The
following functions are likely to be needed by the complete protocol, but are
not covered here:

Authorization policy:
: In this document, we assume that all participants in a room have equal
capability.  Actual messaging systems have authorization policies for which
clients can take which actions in a room.

Advanced join/leave flows:
: In this document, all adds / removes / joins / leaves are initiated from
within the group, since this aligns well with MLS.  Messaging application
support a variety of other flows, some of which this protocol will need to
support.

Consent:
: In this document, we assume that any required consent has already been
obtained, e.g., a user consenting to be added to a room by another user.  The
full protocol will need some mechanisms for establishing this consent.  Some
initial notes are in {{consent}}

Identifiers:
: Certain entities in the MIMI system need to be identified in the protocol.  In
{{identifiers}}, we define a notional syntax for identifiers, but a more
concrete one should be defined.

Abuse reporting:
: There is no mechanism in this document for reporting abusive behavior to a
messaging provider.

Identifier resolution:
: In some cases, the identifier used to initiate communications with a user
might be different from the identifier that should be used internally.  For
example, a user-visible handle might need to be mapped to a durable internal
identifier.  This document provides no mechanism for such resolution.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Most MIMI-related terms are inherited from {{!I-D.barnes-mimi-arch}}. Readers
of this document should be familiar with the Terminology section of the MLS protocol {{!RFC9420}}.

Throughout this document, the examples use the TLS Presentation Language
{{!RFC8446}} and the semantics of HTTP {{!RFC7231}} respectively as
placeholders for a binary encoding mechanism and transport semantics.

# Protocol Overview

As shown in {{fig-layers}}, the overall MIMI protocol is built out of three layers:

1. An application layer that enables messaging functionality
2. A security layer that provides end-to-end security guarantees:
    * Confidentiality for messages
    * Authentication of actors making changes to rooms
    * Agreement on room state across the clients involved in a room
3. A transport layer that provides secure delivery of protocol objects between
   servers.

The MIMI transport is based on HTTPS over mutually-authenticated TLS.  MIMI uses
MLS {{!RFC9420}} for end-to-end security, using the MLS ApplicationSecuritySync
proposal type to syncronize room state across the clients involved in a room.

~~~ aasvg
+------------------------+
|       Application      |
|       +----------------+
|       |  E2E Security  |
+-------+----------------+
|        Transport       |
+------------------------+
~~~
{: #fig-layers title="MIMI protocol layering" }

The remainder of this section walks through a basic scenario that illustrates
how a room works in the MIMI protocol.  The scenario involves the following
actors:

* Service providers `a.example`, `b.example`, and `c.example` represented by
  servers `ServerA`, `ServerB`, and `ServerC` respectively
* Users Alice (`alice@a.example`), Bob (`bob@b.example`) and Cathy
  (`cathy@c.example) of the respective service providers
* Clients `ClientA1`, `ClientA2`, `ClientB1`, etc. belonging to these users
* A room `clubhouse@a.example` where the three users interact

As noted in {{!I-D.mimi-arch}}, the MIMI protocol only defines interactions
between service providers' servers.  Interactions between between clients and
servers within a service provider domain are shown here for completeness, but
surrounded by `[[ double brackets ]]`.

## Alice Creates a Room

The first step in the lifetime of a MIMI room is its creation on the hub server.
This operation is local to the service provider, and does not entail any MIMI
protocol operations.  However, it must establish the initial state of the room,
which is then the basis for protocol operations related to the room.

Here, we assume that Alice uses ClientA1 to create a room with the following
properties:

* Identifier: `clubhouse@a.example`
* Participants: `[alice@a.example]`

ClientA1 also creates an MLS group with group ID `clubhouse@a.example` and
ensures via provider-local operations that Alice's other clients are members of
this MLS group.

## Alice adds Bob to the Room

Adding Bob to the room entails operations at two levels.  First, Bob's user
identity must be added to the room's participant list.  Second, Bob's clients
must be added to the room's MLS group.

The process of adding Bob to the room thus begins by Alice fetching key material
for Bob's clients.  Alice then updates the room by sending an MLS Commit over
the following proposals:

* An ApplicationSync proposal updating the room state by adding Bob to the
  participant list
* Add proposals for Bob's clients

The MIMI protocol interactions are between Alice's server ServerA and Bob's
server ServerB.  ServerB stores KeyPackages on behalf of Bob's devices.  ServerA
performs the key material fetch on Alice's behalf, and delivers the resulting
KeyPackages to Alice's clients.  Both ServerA and ServerB remember the sources
of the KeyPackages they handle, so that they can route a Welcome mesasage for
those KeyPackages to the proper recipients -- ServerA to ServerB, and ServerB to
Bob's clients.

[[ NOTE: In the full protocol, it will be necessary to have consent and access
control on these operations.  We have elided that step here in the interest of
simplicity.  Some initial thoughts are in {{consent}} ]]

~~~ ascii-art
ClientA1       ServerA         ServerB         ClientB*
  |               |               |               |
  |               |               |     Store KPs |
  |               |               |<~~~~~~~~~~~~~~|
  |               |               |<~~~~~~~~~~~~~~|
  | Request KPs   |               |               |
  |~~~~~~~~~~~~~~>| /keyMaterial  |               |
  |               |-------------->|               |
  |               |        200 OK |               |
  |           KPs |<--------------|               |
  |<~~~~~~~~~~~~~~|               |               |
  |               |               |               |

ClientB*->ServerB: [[ Store KeyPackages ]]
ClientA1->ServerA: [[ request KPs for bob@b.example ]]
ServerA->ServerB: POST /keyMaterial KeyMaterialRequest
ServerB: Verify that Alice is authorized to fetch KeyPackages
ServerB: Mark returned KPs as reserved for Alice’s use
ServerB->ServerA: 200 OK KeyMaterialResponse
ServerA: Remember that these KPs go to b.example
ServerA->ClientA1: [[ KPs ]]
~~~
{: #fig-ab-kp-fetch title="Alice Fetches KeyPackages for Bob's Clients" }

~~~ ascii-art
ClientA1       ServerA         ServerB         ClientB*
  |               |               |               |
  | Commit, etc.  |               |               |
  |~~~~~~~~~~~~~~>| /notify       |               |
  |               |-------------->| Welcome, Tree |
  |               |               |~~~~~~~~~~~~~~>|
  |               |               |~~~~~~~~~~~~~~>|
  |               |        200 OK |               |
  |      Accepted |<--------------|               |
  |<~~~~~~~~~~~~~~|               |               |
  |               |               |               |

ClientA1: Prepare Commit over AppSync(+bob@b.example), Add*
ClientA1->ServerA: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerA: Verify that AppSync, Adds are allowed by policy
ServerA: Identifies Welcome domains based on KP hash in Welcome
ServerA->ServerB: POST /notify/clubhouse@a.example Intro{ Welcome, RatchetTree? }
ServerB: Recognizes that Welcome is adding Bob to room clubhouse@a.example
ServerB->ClientB*: [[ Welcome, RatchetTree? ]]
~~~
{: #fig-ab-add title="Alice Adds Bob to the Room and Bob's Clients to the MLS Group" }


## Bob adds Cathy to the Room

The process of adding Bob was a bit abbreviated because Alice is a user of the
hub service provider.  When Bob adds Cathy, we see the full process, involving
the same two steps (KP fetch followed by add), but this time indirected via the
hub server ServerA.  Also, now that there are users on ServerB involved in the
room, the hub ServerA will have to distribute the Commit adding Cathy and
Cathy's clients to ServerB as well as forwarding the Welcome to ServerC.

~~~ ascii-art
ClientB1       ServerB         ServerA         ServerC         ClientC*
  |               |               |               |               |
  |               |               |               |     Store KPs |
  |               |               |               |<~~~~~~~~~~~~~~|
  |               |               |               |<~~~~~~~~~~~~~~|
  | Request KPs   |               |               |               |
  |~~~~~~~~~~~~~~>| /keyMaterial  | /keyMaterial  |               |
  |               |-------------->|-------------->|               |
  |               |        200 OK |        200 OK |               |
  |           KPs |<--------------|<--------------|               |
  |<~~~~~~~~~~~~~~|               |               |               |
  |               |               |               |               |

ClientC*->ServerC: [[ Store KeyPackages ]]
ClientB1->ServerB: [[ request KPs for bob@b.example ]]
ServerB->ServerA: POST /keyMaterial KeyMaterialRequest
ServerA->ServerC: POST /keyMaterial KeyMaterialRequest
ServerB: Verify that Bob is authorized to fetch KeyPackages
ServerB: Mark returned KPs as reserved for Bob’s use
ServerC->ServerA: 200 OK KeyMaterialResponse
ServerA: Remember that these KPs go to b.example
ServerA->ServerB: 200 OK KeyMaterialResponse
ServerB->ClientB1: [[ KPs ]]
~~~
{: #fig-bc-kp-fetch title="Bob Fetches KeyPackages for Cathy's Clients" }

~~~ ascii-art
ClientB1       ServerB         ServerA         ServerC         ClientC*  ClientB*  ClientA*
  |               |               |               |               |         |         |
  | Commit, etc.  |               |               |               |         |         |
  |~~~~~~~~~~~~~~>| /update       |               |               |         |         |
  |               |-------------->|               |               |         |         |
  |               |        200 OK |               |               |         |         |
  |               |<--------------|               |               |         |         |
  |      Accepted |               | /notify       |               |         |         |
  |<~~~~~~~~~~~~~~|               |-------------->| Welcome, Tree |         |         |
  |               |               |               |~~~~~~~~~~~~~~>|         |         |
  |               |               |               |~~~~~~~~~~~~~~>|         |         |
  |               |       /notify |               |               |         |         |
  |               |<--------------|               |               |         |         |
  |               | Commit        |               |               |         |         |
  |               |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|         |
  |               |               | Commit        |               |         |         |
  |               |               |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|
  |               |               |               |               |         |         |

ClientB1: Prepare Commit over AppSync(+cathy@c.example), Add*
ClientB1->ServerB: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerB->ServerA: POST /update/clubhouse@a.example CommitBundle
ServerA: Verify that Adds are allowed by policy
ServerA->ServerB: 200 OK
ServerA->ServerC: POST /notify/clubhouse@a.example Intro{ Welcome, RatchetTree? }
ServerC: Recognizes that Welcome is adding Cathy to clubhouse@a.example
ServerC->ClientC*: [[ Welcome, RatchetTree? ]]
ServerA->ServerB: POST /notify/clubhouse@a.example Commit
ServerB->ClientB*: [[ Commit ]]
ServerA->ClientA*: [[ Commit ]]
~~~
{: #fig-bc-add title="Bob Adds Cathy to the Room and Cathy's Clients to the MLS Group" }

## Cathy Sends a Message

Now that Alice, Bob, and Cathy are all in the room, Cathy wants to say hello to
everyone.  Cathy's client encapsulates the message in an MLS PrivateMessage and
sends it to ServerC, who forwards it to the hub ServerA on Cathy's behalf.
Assuming Cathy is allowed to speak in the room, ServerA will forward Cathy's
message to the other servers involved in the room, who distribute it to their
clients.

~~~ ascii-art
ClientC1       ServerC         ServerA         ServerB         ClientB*  ClientC*  ClientA*
  |               |               |               |               |         |         |
  | Message       |               |               |               |         |         |
  |~~~~~~~~~~~~~~>| /message      |               |               |         |         |
  |               |-------------->|               |               |         |         |
  |               |        200 OK |               |               |         |         |
  |               |<--------------|               |               |         |         |
  |      Accepted |               | /notify       |               |         |         |
  |<~~~~~~~~~~~~~~|               |-------------->| Message       |         |         |
  |               |               |               |~~~~~~~~~~~~~~>|         |         |
  |               |       /notify |               |               |         |         |
  |               |<--------------|               |               |         |         |
  |               | Message       |               |               |         |         |
  |               |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|         |
  |               |               | Message       |               |         |         |
  |               |               |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|
  |               |               |               |               |         |         |

ClientC1->ServerC: [[ MLSMessage(PrivateMessage) ]]
ServerC->ServerA: POST /message/clubhouse@a.example MLSMessage(PrivateMessage)
ServerA: Verifies that message is allowed
ServerA->ServerC: POST /notify/clubhouse@a.example Message{ MLSMessage(PrivateMessage) }
ServerA->ServerB: POST /notify/clubhouse@a.example Message{ MLSMessage(PrivateMessage) }
ServerA->ClientA*: [[ MLSMessage(PrivateMessage) ]]
ServerB->ClientB*: [[ MLSMessage(PrivateMessage) ]]
ServerC->ClientC*: [[ MLSMessage(PrivateMessage) ]]
~~~
{: #fig-c-msg title="Cathy Sends a Message to the Room" }

## Bob Leaves the Room

A user removing another user follows the same flow as adding the user.  They
user performing the removal creates an MLS commit covering Remove proposals for
all of the removed user's devices, and an AppSync proposal updating the room
state to remove the removed user from the room's participant list.

Leaving is slightly more complicated because the leaving user cannot remove all
of their devices from the MLS group.  Instead, the leave happens in three steps:

1. The leaving client constructs MLS Remove proposals for all of the user's
   devices (including the leaving client), and an AppSync proposal that removes
   its user from the participant list.
2. The leaving client sends this proposals to the hub.  The hub caches the proposals.
3. The next time a client attempts to commit, the hub requires the client to
   include the cached proposals.

The hub thus guarantees the leaving client that they will be removed as soon as
possible.

~~~
ClientB1       ServerB         ServerA         ServerC         ClientC1
  |               |               |               |               |
  | Proposals     |               |               |               |
  |~~~~~~~~~~~~~~>| /update       |               |               |
  |               |-------------->|               |               |
  |               |        200 OK |               |               |
  |               |<--------------|               |               |
  |      Accepted |               |               |               |
  |<~~~~~~~~~~~~~~|               |               |               |
  |               |               |               |        Commit |
  |               |               |               |<~~~~~~~~~~~~~~|
  |               |               |       /update |               |
  |               |               |<~~~~~~~~~~~~~~|               |
  |               |               | 401 Proposals |               |
  |               |               |~~~~~~~~~~~~~~>|               |
  |               |               |               | Reject(Props) |
  |               |               |               |~~~~~~~~~~~~~~>|
  |               |               |               | Commit(Props) |
  |               |               |               |<~~~~~~~~~~~~~~|
  |               |               |       /update |               |
  |               |               |<~~~~~~~~~~~~~~|               |
  |               |               | 200 OK        |               |
  |               |               |-------------->|               |
  |               |               |               | Accepted      |
  |               |               |               |~~~~~~~~~~~~~~>|
  |               |       /notify | /notify       |               |
  |               |<--------------|-------------->|               |
  |               |               |               |               |

ClientB1: Prepare Remove*, AppSync(-bob@b.example)
ClientB1->ServerB: [[ Remove*, AppSync ]]
ServerB->ServerA: POST /update/clubhouse@a.example Remove*, AppSync
ServerA: Verify that Removes, AppSync are allowed by policy; cache
ServerA->ServerB: 200 OK
ClientC1->ServerC: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerC->ServerA: POST /update/clubhouse@a.example CommitBundle
ServerA: Check whether Commit includes queued proposals; reject
ServerA->ServerC: 401 Unauthorized; Remove*, AppSync
ServerC->ClientC1: Remove*, AppSync
ClientC1: Prepare Commit over Remove*, AppSync, in addition to any others
ClientC1->ServerC: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerC->ServerA: POST /update/clubhouse@a.example CommitBundle
ServerA: Check whether Commit includes queued proposals; accept
ServerA->ServerC: 200 OK
ServerA->ServerB: POST /notify/clubhouse@a.example Commit
ServerA->ServerC: POST /notify/clubhouse@a.example Commit
~~~
{: #fig-b-leave title="Bob Leaves the Room" }

# Identifiers

Four types of entity need to be identified in the MIMI protocol:

1. Messaging providers
2. Users
3. Rooms
4. MLS groups (in the MLS group ID field)

Messaging providers are identified by domain name.  The remaining identifier
types are domain-scoped, where the domain represents the messaging provider
hosting a user or room.  MLS groups use the same identifier as the room to which
they are attached.

There are several options for the precise syntax for these identifiers, as
discussed in {{?I-D.mahy-mimi-identity}}.  For purposes of this document, we use
the following notional syntax:

* Messaging providers: `a.example`
* Users: `alice@a.example`
* Rooms: `clubhouse@a.example`
* MLS groups: `clubhouse@a.example`

# Transport layer

MIMI servers communicate using HTTPS.  The HTTP request MUST identify the
source and target providers for the request, in the following way:

* The target provider is indicated using a Host header {{!RFC9110}}.  If the
  provider is using a non-standard port, then the port component of the Host
  header is ignored.
* The source provider is indicated using a From header {{!RFC9110}}.  The
  `mailbox` production in the From header MUST use the `addr-spec` variant, and
  the `local-part` of the address MUST contain the fixed string `mimi`.  Thus,
  the content of the From header will be `mimi@a.example`, where `a.example` is
  the domain name of the source provider.

The TLS connection underlying the HTTPS connection MUST be mutually
authenticated.  The certificates presented in the TLS handshake MUST
authenticate the source and target provider domains, according to {{!RFC6125}}.

The bodies of HTTP requests and responses are defined by the individual
endpoints defined in {{application-layer}}.

# E2E Security Layer

Every MIMI room has an MLS group associated to it, which provides end-to-end
security guarantees.  The clients participating in the room manage the MLS-level
membership by sending Commit messages covering Add and Remove proposals.

Every message sent within a room is authenticated and confidentiality-protected
by virtue of being encapsulated in an MLS PrivateMessage object.

MIMI uses the MLS application state synchronization mechanism
({{mls-application-state-synchronization}}) to ensure that the clients involved
in a MIMI room agree on the state of the room.  Each MIMI message that changes
the state of the room is encapsulated in an AppSync proposal and transmitted
inside an MLS PublicMessage object.

The PublicMessage encapsulation provides sender authentication, including the
ability for actors outside the group (e.g., servers involved in the room) to
originate AppSync proposals.  Encoding room state changes in MLS proposals
ensures that a client will not process a commit that confirms a state change
before processing the state change itself.

# Application Layer

Servers in MIMI provide a few functions that enable messaging applications.
All servers act as publication points for key material used to add their users
to rooms. The hub server for a room tracks the state of the room, and controls
how the room's state evolves, e.g., by ensuring that changes are compliant with
the room's policy. Non-hub servers facilitate interactions between their clients
and the hub server.

In this section, we desribe the state that servers keep and the HTTP endpoints
they expose to enable these functions.

## Server State

Every MIMI server is a publication point for users' key material, via the
`keyMaterial` endpoint discussed in {{fetch-key-material}}.  To support this
endpoint, the server stores a set of KeyPackages, where each KeyPackage belongs
to a specific user and device.

The hub server for the room stores the state of the room, comprising:

* A list of user identifiers for users who are members of the room

[[ NOTE: This state is clearly not sufficient.  We need a more full description
of the room. ]]

When a client requests key material via the hub, the hub records the
KeyPackageRef values for the returned KeyPackages, and the identity of the
provider from which they were received.  This information is then used to route
Welcome message to the proper provider.

## Room State Changes

The state of the room can be changed by adding or removing users.  These changes
are described with a simple JSON object of the following form:

~~~
{
    "add": ["diana@d.example", "eric@e.example"],
    "remove": ["bob@b.example"],
}
~~~
{: #fig-room-state-change title="Changing the state of the room" }

[[ NOTE: This syntax is clearly not sufficient.  To go along with the more full
description of room state mentioned above, we need a more full taxonomy of the
ways that room state can change. ]]

To put these changes into effect, a client or server encodes them in an AppSync
proposal, signs the proposal as a PublicMessage, and submits them to the
`update` enpoint on the hub.

## Directory

Like the ACME protocol {{?RFC8555}}, the MIMI protocol uses a directory document
to convey the HTTPS URLs used to reach certain endpoints (as opposed to hard
coding the endpoints).

The directory URL is discovered using the `mimi-protocol-directory` well-known
URI. The response is a JSON document with URIs for each type of endpoint.

~~~
GET /.well-known/mimi-protocol-directory
~~~

~~~
{
  "keyMaterial":
     "https://mimi.example.com/v1/keyMaterial/{targetUser}",
  "update": "https://mimi.example.com/v1/update{roomId}",
  "notify": "https://mimi.example.com/v1/notify/{roomId}",
  "submitMessage":
     "https://mimi.example.com/v1/submitMessage/{roomId}",
  "groupInfo": "https://mimi.example.com/v1/groupInfo/{roomId}",
  "requestConsent":
     "https://mimi.example.com/v1/requestConsent/{targetUser}",
  "updateConsent":
     "https://mimi.example.com/v1/updateConsent/{requesterUser}"
}
~~~

## Fetch Key Material

This action attempts to claim initial keying material for all the clients
of a single user at a specific provider. The keying material may not be
reused unless identified as "last resort" keying material.

The target user's URI is listed in the request path. KeyPackages
requested using this primitive MUST be sent via the hub provider of whatever
room they will be used in. (If this is not the case, the hub provider will be
unable to forward a Welcome message to the target provider).

~~~
POST /keyMaterial/{targetUser}
~~~

The path includes the target user. The request body includes the protocol
(currently just MLS 1.0), and the requesting user. The request SHOULD include
the room ID for which the KeyPackage is intended, as the target may have only
granted consent for a specific room.

For MLS, the request includes a non-empty list of acceptable MLS ciphersuites,
and an MLS `RequiredCapabilities` object (which contains credential types,
non-default proposal types, and extensions) required by the requesting provider
(these lists can be an empty). The `lastResortAllowed` field SHOULD be false.

The request body has the following form.

~~~ tls
enum {
    reserved(0),
    mls10(1),
    (255)
} Protocol;

struct {
    opaque uri<V>;
} IdentifierUri;

struct {
    Protocol protocol;
    IdentifierUri requestingUser;
    IdentifierUri targetUsers<V>;
    IdentifierUri roomId;
    select (protocol) {
        case mls10:
            CipherSuite acceptableCiphersuites<V>;
            RequiredCapabilities requiredCapabilities;
            bool lastResortAllowed;
    };
} KeyMaterialRequest;
~~~

The response contains a user status code that indicates keying material was
returned for all the user's clients (`success`), that keying material was
returned for some of their clients (`partialSuccess`), or a specific user code
indicating failure. If the user code is success or partialSuccess, each client
is enumerated in the response. Then for each client with a *client* success
code, the structure includes initial keying material (a KeyPackage for MLS 1.0).
If the client's code is `nothingCompatible`, the client's capabilities are
optionally included (The client's capabilities could be omitted for privacy
reasons.)

If the *user* code is `noCompatibleMaterial`, the provider MAY populate the
`clients` list. For any other user code, the provider MUST NOT populate the
`clients` list.

Keying material provided from one response MUST NOT be provided in any other
response unless the `lastResortAllowed` field was set in the requests.
The target provider MUST NOT provide expired keying material (ex: an MLS
KeyPackage containing a LeafNode with a `notAfter` time past the current date
and time).

~~~ tls
enum {
    success(0);
    partialSuccess(1);
    incompatibleProtocol(2);
    noCompatibleMaterial(3);
    userUnknown(4);
    noConsent(5);
    noConsentForThisRoom(6);
    userDeleted(7);
    (255)
} KeyMaterialUserCode;

enum {
    success(0);
    keyMaterialExhausted(1);
    onlyLastResort(2);
    nothingCompatible(3);
    (255)
} KeyMaterialClientCode;


struct {
    KeyMaterialClientCode clientStatus;
    IdentifierUri clientUri;
    select (protocol) {
        case mls10:
            select (clientStatus) {
                case success:
                    KeyPackage keyPackage;
                case nothingCompatible:
                    optional<Capabilities> clientCapabilities;
            };
    };
} ClientKeyMaterial;

struct {
    Protocol protocol;
    KeyMaterialUserCode userStatus;
    IdentifierUri userUri;
    ClientKeyMaterial clients<V>;
} KeyMaterialResponse;
~~~

At minimum, as each MLS KeyPackage is returned to a requesting provider (on
behalf of a requesting IM client), the target provider needs to associate its
`KeyPackageRef` with the target client and the hub provider needs to associate
its `KeyPackageRef` with the target provider. This insures that Welcome messages
can be correctly routed to the target provider and client. These associations
can be deleted after a Welcome message is forwarded or after the KeyPackage
`leaf_node.lifetime.not_after` time has passed.


## Update Room State

Adds, removes, and policy changes to the room are all forms of updating the room
state. They are accomplished using the update transaction which is used for
updating the room base policy, participation list, or its underlying MLS group.

~~~
POST /update/{roomId}
~~~

In MLS 1.0, any change to the room base policy document is always expressed
using a `GroupContextExtensions` proposal. Likewise, any change to the
participant list is always communicated via an `AppSync` proposal type. The
participant list change needed to add a user MUST happen either before or
simultaneously with the corresponding MLS operation.

Removing an active user from a participant list or banning an active participant
SHOULD happen simultaneously with any MLS changes made to the commit removing
the participant.

A hub provider which observes that an active user has been removed or banned,
but still has clients MUST prevent any of those clients from sending or
receiving any additional application messages; MUST prevent any of those clients
from sending Commit messages; and MUST prevent it from sending any proposals
except for `Remove` and `SelfRemove` proposals.

~~~ tls
enum {
  reserved(0),
  system(1),
  owner(2),
  admin(3),
  regular_user(4),
  visitor(5),
  banned(6),
  (255)
} Role;

struct {
  IdentifierUri user;
  Role roles<V>;
} UserRoles;

struct {
    UserRoles addUsers<V>;
    UserRoles updateUsers<V>;
    IdentifierUri removeUsers<V>;
} AppSync;
~~~

Each user can be added with one or more roles. This list of roles can be
updated. Note that removing a user does not ban the user. To ban a user,
update their role to `banned`.

The update request body is described below:

~~~ tls
struct {
  select (room.protocol) {
    case mls10:
      PublicMessage proposalOrCommit;
      select (proposalOrCommit.content.content_type) {
        case commit:
          optional<Welcome> welcome;
          GroupInfo groupInfo;   /* without embedded ratchet_tree */
          RatchetTreeOption ratchetTreeOption;
        case proposal:
          PublicMessage moreProposals<V>; /* a list of additional proposals */
      };
  };
};
~~~

In this case Alice creates a Commit containing a AppSync proposal adding
Bob@b.example, and Add proposals for all Bob's MLS clients.  Alice includes the
Welcome message which will be sent for Bob, a GroupInfo object for the hub
provider, and complete `ratchet_tree` extension.

~~~ tls
enum {
  reserved(0),
  full(1),
  compressed(2),
  delta(3),
  patch(4),
  byReference(5)
  (255)
} RatchetTreeRepresentation;

struct {
  RatchetTreeRepresentation representation;
  select (representation) {
    case full:
      Node ratchet_tree<V>;
  };
} RatchetTreeOption;
~~~

The response body is described below:

~~~ tls
enum {
  success(0),
  wrongEpoch(1),
  notAllowed(2),
  hubUnresponsive(3),
  invalidProposal(4),
  (255)
} UpdateResponseCode;

struct {
    UpdateResponseCode responseCode;
    string errorDescription;
} UpdateRoomResponse
~~~

## Submit a Message

~~~
POST /submitMessage/{roomId}
~~~

If the protocol is MLS 1.0, the request body is an MLSMessage with. a WireFormat
of PrivateMessage (an application message).

The response merely indicates if the message was accepted by the hub provider.

**ISSUE:** Do we want to offer a distinction between regular application
messages and ephemeral applications messages (for example "is typing"
notifications), which do not need to be queued at the target provider.

## Notify Servers of Room Events

If the hub provider accepts an application or handshake message (proposal or
commit) message, it forward that message to all other providers with active
participants in the room and all local clients which are active participants.
This is described as fanning the message out. An MLS Welcome message is sent to
the providers and local users associated with the `KeyPackageRef` values to
which the bulk of the Welcome (the `encrypted_group_info`) was encrypted.

The hub provider also fans out any messages which originate from itself (ex: MLS
External Proposals).

The hub can include multiple concatenated `FanoutMessage` objects relevant to
the same room.

~~~
POST /notify/{roomId}
~~~

~~~ tls
struct {
  MLSMessage message;
  /* the hub acceptance time (in milliseconds from the UNIX epoch) */
  uint64 timestamp;
  select (message.wire_format) {
    case welcome:
      RatchetTreeOption ratchetTreeOption;
  };
} FanoutMessage;
~~~

If the protocol is MLS 1.0, the request body is one or more of MLSMessage with
wire_format one of PrivateMessage (application message), PublicMessage (Commit
or Proposal), or Welcome. In the case of a Welcome message, a
`RatchetTreeOption` is also included.

**NOTE:** Correctly fanning out Welcome messages relies on the hub and target
providers storing the `KeyPackageRef` of claimed KeyPackages.

# Use of MLS DS/AS

MLS defines an abstract Delivery Server (DS) and Authentication Server (AS).
This abstract functionality is split across all the MLS clients and the
providers, but an important function is provided specifically by the hub
provider. The hub is responsible for the ordering of messages, fanning messages
out to the appropriate providers, and maintaining the GroupInfo so authorized
clients can add themselves to groups using an external commit.

The MLS profile for MIMI requires a concept of a user identifier and a mechanism
to associate credentials in MLS clients to MIMI users.

Clients are free to fetch pseudonymous user identifiers from their local
provider, and use these in rooms. These pseudonymns need only identify the
client's provider to other providers and clients. The actual user identity could
be shared among the active participants of the room, by using a credential with
selective disclosures, or by signing a binding with both the pseudonymous
signature key and the "real" identity signature key and presenting the binding
to other members.

# MLS Application State Synchronization

[[ This section should be moved to its own document in the MLS working group ]]

One of the primary security benefits of MLS is that the MLS key schedule
confirms that the group agrees on certain metadata, such as the membership of
the group. Members that disagree on the relevant metadata will arrive at
different keys and be unable to communicate. Applications based on MLS can
integrate their state into this metadata in order to confirm that the member of
an MLS group agree on application state as well as MLS metadata.

Here, we define two extensions to MLS to facilitate this application design:

1. A GroupContext extension `application_state` that confirms agreement on
   application state.
2. A new proposal type AppSync that allows MLS group members to propose changes
   to the agreed application state.

The `application_state` extension allows the application to inject a state
object into the MLS key schedule.  Changes to this state can be made out of
band, or using the AppSync proposal.  Using the AppSync proposal ensures that
members of the MLS group have received the relevant state changes before they
are reflected in the group's `application_state`.

The content of the `application_state` extension and the `AppSync` proposal are
structured as follows:

~~~ tls
struct {
    uint32 application_id;
    opaque application_state<V>;
} ApplicationState;
~~~
{: #fig-app-state title="The application_state extension" }

~~~ tls
struct {
    uint32 application_id;
    opaque application_state_update<V>;
} AppSync;
~~~
{: #fig-app-sync title="The AppSync proposal type" }

The `application_id` determines the structure and interpretation of the
`application_state` and `application_state_update` fields.  The assumed
structure is that the application holds some state, which is either expressed
directly or committed to in the `application_state` field.  AppSync proposals
contain deltas / diffs on this state, which the client uses to update the
representation of the state in `application_state`.

A client receiving an AppSync proposal applies it in the following way:

* Identify a GroupContext extension with the same `application_id` as the
  AppSync
* Verify that the current application state matches the value of the
  `application_state` field in the extension
* Apply the `application_state_update` to the current application state
* Replace the `application_state` in that extension with a representation of the
  updated application state.

Note that the `application_state` extension is updated directly by AppSync
proposals; a GroupContextExtensions proposal is not necessary.  A proposal list
that contains both an AppSync proposal and a GroupContextExtensions proposal
that changes the value of the affected `application_state` extension is invalid.

A proposal list in a Commit MAY contain more than one AppSync proposal.  The
proposals are applied in the order that they are sent in the Commit, after any
GroupContextExtensions proposals.

[[ TODO: IANA registry for application_id; register extension and proposal types
]]

# Consent

Most instant messaging systems have some notion of how a user consents to be
added to a room, and how they manipulate this consent.

In the connection-oriented model, once two users are connected, either user can
add the other to any number of rooms. In other systems (often with many large
and/or public rooms), a user needs to consent individually to be added to a
room.

The MIMI consent mechanism supports both models and allows them to coexist. It
allows a user to request consent, grant consent, revoke consent, and cancel a
request for consent. Each of these consent operations can indicate a specific
room, or indicate any room.

A connection grant or revoke does not need to specify a room if a connection
request did, or vice versa. A connection grant or revoke does not even need to
follow a connection request.

For example, Alice could ask for consent to add Bob to a specific room. Bob
could send a connection grant for Alice to add him to any room, or a connection
revoke preventing Alice from adding him to any room. Similarly, Alice might have
sent a connection request to add Bob for any room (as a connection request),
which Bob ignored or did not see. Later, Bob wants to join a specific room
administered by Alice. Bob sends a connection grant for the specific room for
Alice and sends a Knock request to Alice asking to be added. Finally, Cathy
could send a connection grant for Bob (even if Bob did not initiate a connection
request to Cathy), and Alice could recognize Cathy on the system and send a
connection revoke for her preemptively.

**NOTE:** Many providers use additional factors to apply default consent within
their service such as a user belonging to a specific workgroup or employer,
participating in a related room (ex: WhatsApp "communities"), or presence of a
user in the other user's contact list. MIMI does not need to provide a way to
replicate or describe these supplemental mechanisms, since they are strongly
linked to specific provider policies.

Consent requests have sensitive privacy implications. The sender of a consent
request should receive an acknowledgement that the request was received by the
*provider* of the target user. For privacy reasons, the requestor should not
know if the target user received or viewed the request. The original requestor
will obviously find out if the target grants consent, but a consent
revocation/rejection is typically not communicated to the revoked/rejected user
(again for privacy reasons).

Consent operations are only sent directly between the acting provider (sending
the request, grant, revoke, or cancel) and the target provider (the object of
the consent). In other words, the two providers must have a direct peering
relationship.

In our example, Alice requests consent from Bob for any room. Later, Bob sends a
grants consent to Alice to add him to any room. At the same time as sending the
consent request, Alice grants consent to Bob to add her to any room.

# Security Considerations

TODO Security

[[ Servers authenticate to each other with mTLS ]]

[[ E2E security via MLS ]]

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
