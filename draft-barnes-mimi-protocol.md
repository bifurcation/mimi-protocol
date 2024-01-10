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
fetch the current MLS GroupInfo, and request, grant, and reject consent to
communicate with users on other providers.

Messages sent inside each room are end-to-end encrypted using the Messaging
Layer Security (MLS) protocol {{!RFC9420}}, and each room is associated with an
MLS group. MLS also ensures that clients in a room agree on the room policy and
participation.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Most MIMI-related terms are inherited from {{!I-D.barnes-mimi-arch}}. Readers
of this document should be familiar with the Terminology section of the MLS protocol {{!RFC9420}}.

Throughout this document, the examples use the TLS Presentation Language
{{!RFC8446}} and the semantics of HTTP {{!RFC7231}} respectively as
placeholders for a binary encoding mechanism and transport semantics.

# Protocol Overview

[[ Layering: State control; security; transport ]]

## Dramatis Personae
## Alice Creates a Room
## Alice adds Bob to the Room
## Bob adds Charlie to the Room
## Cathy sends a Message

# Transport layer

* Server for user identified by domain half of identifier
* HTTPS over mutually-authenticated TLS
* Transport framing needs to identify intended src/dst domains
* TLS certificates need to authenticate src/dst domains

# E2E Security Layer

* All actions within a room are E2E-authenticated
* Each room has an associated MLS group
* Messages are protected as PrivateMessage
* Room changes are protected as PublicMessage(Proposal)
* ... incl. origination by servers

# Application Layer

## Room State

[[ Only room state right now is a list of participant IDs, MLS group ID ]]
[[ Hub and followers track KP hashes for Welcome routing ]]

## Application Endpoints

[[ keyMaterial / notify / update / message ]]
[[ Discovery from domain via .well-known ]]

### Fetching Key Material for a User
### Updating Room State
### Notifying Followers of Room Events
### Sending Messages

# Consent Sketch

[[ Consent is important, so we include a preliminary sketch here ]]



# HEDGEDOC CONTENT BELOW THIS LINE

# Operational Model

[[ RLB: This might be better to pull up into the architecture. ]]

* Goal of MIMI protocol is to allow control of room state
* State stored on hub for room
    * Confirmed state
    * List of outstanding proposals
    * Proposed state = Confirmed + proposals

* Clients take action "in immediate mode" ~ Proposals + Commit
* Some actions cannot be committed immediately
    * Self-remove
    * Any action intiated by a server
* If an action cannot be committed immediately, the hub enqueues it and assures it is committed as soon as possible

* Some actions happen outside a room: Consent, claim key material, identifier resolution

Functional/layering model of MIMI protocol:

* Transport ~ authenticated, request/response connection between servers
* Identifier resolution request / response
* Consent request / response
* Room state change request / response
    * Proposal* + \[CommitBundle]
    * Proposal = Add / Update / Remove / MIMI
* Message submission request / response

```
                   +---------------+-------------+
                   | MIMI Proposal | Content Fmt |
+--------+---------+---------------+-------------+
| ID res | Consent | State Change  | Submission  |
+--------+---------+---------------+-------------+
|                    Transport                   |
+------------------------------------------------+
```
```
POST /updateRoom/dxzxxDuvMOTsUD4D_YdSyQ

PublicMessage{
  /* MLS Commit - inside is a list of Proposals */
  Commit[
    ParticipantListPatch{
      addUsers [
        [Charlie, [normal]],
        [Doug, [normal,admin]]
      ]
      updateUsers [],
      removeUsers [Bob]
    },
    Add(Charlie1),
    Add(Doug1),
    Add(Doug2),
    Remove(Bob1),
    Remove(Bob2),
    Remove(Bob3)
  ],
  /* MLS Welcome encrypted for Charlie1, Doug1, Doug2 */
  Welcome
  /* MLS GroupInfo if the Commit is accepted (without the ratchet_tree)
  GroupInfo,
  /*
  RatchetTree
}
```

Removing yourself
```
POST /updateRoom/dxzxxDuvMOTsUD4D_YdSyQ

/* concatenated MLS Proposals (each inside a PublicMessage) */
ParticipantListPatch{
  removeUsers [Alice]
},
Remove(Alice1),
SelfRemove(Alice2),
Remove(Alice3)

```

Charlie1 commits Alice's pending proposals. This is accepted even though Charlie wouldn't have permission to remove Alice himself.
```
POST /updateRoom/dxzxxDuvMOTsUD4D_YdSyQ

PublicMessage{
  /* MLS Commit - inside is a list of Proposals */
  Commit[
    ParticipantListPatch{
      removeUsers [Alice]
    },
    Remove(Alice1),
    SelfRemove(Alice2),
    Remove(Alice3)
  ],
  /* No MLS Welcome  */
  /* MLS GroupInfo if the Commit is accepted (without the ratchet_tree) */
  GroupInfo,
  /* ratchet_tree or some other representation */
  RatchetTree
}
```

# ABCs of the Protocol

The MIMI transport protocol is designed to allow users of instant messaging
on different providers to participate together in group chats using
a common end-to-end encryption protocol and content format. It does not specify the transport protocol between clients and their providers, but it does
constrain them to faithfully implement the end-to-end encryption protocol and the MIMI content format.

We explain the protocol by walking through various scenarios, starting with the most commonly used. First we will show how Alice adds Bob to an existing room. For this case we assume that Room1 exists and Alice is room administrator (she can add and remove users and change roles), Alice already has a stable identifier for the user Bob, and that Bob has somehow indicated consent for Alice to add him to this room (or rooms in general).

Throughout this section, we will detail the individual operations from the
perspective of the user Alice. All actions by Alice are taken through Alice's
client and her server's proprietary API. While some parts of the individual
messages originate from Alice's clients, the MIMI protocol is only concerned
with the communication between Alice's, Bob's and Cathy's servers. So whenever
users are referenced throughout this section, it is implied that they are
communicating with their respective servers, which then communicate with
one another.

## Dramatis Personae

* Alice, Bob: Users
* Alice1, Alice2, etc. are Alice's devices; likewise for Bob and other users
* ProviderA is Alice's provider, ProviderB is Bob's provider, etc.
* Room1 is a room in which the users are interacting

## Alice creates a room

We assume that someone on ProviderA (possibly Alice) has created Room1, that ProviderA is the *hub provider* for the room, and that the room policy means that if you have the "admin" role in the room you can add and remove users to the participant list and change the roles of participants.

Creating a room is done between a client and its local provider and is
out of scope of MIMI. However, we assume that Room1 has the following
policies.

- the room is for members only;
- the room is not moderated;
- multiple MLS clients are allowed for each user;
- the administrator and owner roles are allowed to add and remove users, and to promote a user to be an administrator.

We also assume that Alice is the owner, and therefore an administrator,
of the room. Alice has owner, administrator, and regular user roles in the room.

## Alice adds Bob

### Alice gets initial keying material for Bob

In end-to-end encryption protocols, there is typiclly some type of initial ephemeral keying material which clients generate. Clients often upload this keying information in care of their provider. This keying information is almost always designed to be used only once. In MLS this initial keying information is called
a KeyPackage. We use the terminology "claim" to exclusively reserve keying information from the target user's provider.

In MLS, in order for Alice to add Bob, her client needs to somehow obtain a KeyPackage unique for each group, for each of Bob's clients. In MLS, two rooms with the same client have separate keying information for that client, so even if Bob is already in another group with Alice, her client needs Bob's KeyPackages to add his clients to a new group.


To create a conversation with Bob, Alice would like to get KeyPackages for all
of Bob's clients (for example for clients on his laptop, his phone, and his tablet). Alice's client requests Bob's KeyPackages from her local provider (not specified by MIMI), then her provider sends a request to Bob's provider asking for KeyPackages for Bob's clients. That request may include the
room ID in which Alice intends to use the KeyPackages, in case Bob's consent
only applies to a single room.

```
Bob* -> ProviderB: KeyPackage
Alice1 -> ProviderA: Get key material for Bob@B in Room1
ProviderA -> ProviderB: KeyMaterialRequest(Bob@B in Room1)
ProviderB -> ProviderA: KeyMaterialResponse(Bob1, Bob2, Bob3)
ProviderA -> Alice1: KeyPackages for Bob1, Bob2, Bob3
```

Like all operations that are not authorized based on room policy, claiming a key material is subject to complying with Bob's consent policy (often as delegated to Bob's provider). Consent is discussed more in [[consent section]]. For the purpose of this simple example, we assume that Alice already has appropriate consent.

### Alice adds Bob to the participant list and MLS group

In MIMI, the room policy consists of the base policy (which can refer to roles,
but does not refer to specific participants), the participant list, and the
cryptographic state of the end-to-end protocol (in MLS, the corresponding MLS
group state). The participant list also contains the roles associated with each
participant.

The base policy is expected to be modified infrequently and therefore it can
only be updated in its entirety. The participant list, however is expected to
change frequently in busy rooms. Each change to the participant list contains
a list of users who are added (with their roles), a list of users who are removed, and a list of
users whose roles have changed.

In MLS, the base policy document is communicated in an MLS GroupContext
extension. Any change to the base policy document (which is likely to be rare)
is made by committing an MLS GroupContextExtensions proposal.

Changes to the participant list are maintained by patching the participation
list. In MLS this accomplished by committing a `ParticipantListPatch` proposal
(defined in this document).

Unless the base policy document defines otherwise, an MLS client who makes a
change to the participant list is expected to simultaneously effect the
corresponding change to the MLS group state (to the extent allowed by the
protocol). Except for active participants committing valid pending remove
proposals, a client making a change to the participant list should not expect
any other client to take any immediate action based on updating the participant
list alone.

Making any of these changes, the client sends either a set of proposals,
or sends a Commit bundle, which consists of a Commit, a Welcome if there were
any new clients added, a GroupInfo, and any changes to the ratchet tree. This is
described in more detail in the syntax section.

If Alice wants to add Bob, her client generates an MLS commit
with a `ParticipantListPatch` proposal adding Bob as a participant and `Add`
proposals for each of Bob's clients; and an MLS Welcome message including Bob's clients. Between Alice's client and Alice's provider they also must insure Alice's provider (which is the hub for this room) has the GroupInfo and ratchet tree for the room if the commit is accepted. If the hub provider accepts this "commit bundle", it will forward (fan out) the commit to the current members of the MLS group, and forward a welcome message to any new MLS clients just added to the group.

Note that there are other ways for Bob to eventually participate in the room, but the method just dicsussed is the most straightforward and is preferred. Alice could commit a `ParticipantListPatch`
just adding Bob as an inactive participant, and may need to if she cannot obtain
KeyPackages for Bob's clients (for example, if she is waiting for consent from
Bob), but Alice should not expect another participant, Bob, or some provider to
react to Bob being added, unless some specific policy arrangement is specified
in the base policy.

```
Alice1 -> ProviderA: CommitBundle(PLP, Add*)
ProviderA: Validate commit sequencing; PLP authz by policy; Add authz by PL
ProviderA -> Alice1: Commit accepted
```

### Alice and Bob exchange messages

Any of Alice's or Bob's clients can try to send a instant message to the room. For an MLS room, Alice's or Bob's provider just sends the appropriate MLS application message to the hub provider. The hub provider verifies that the sender is an active participant of the room (and therefore a member of the corresponding MLS group), has permission to send messages in the room, and is valid according to some other MLS-specific checks.

If the hub accepts the message, it fans the message out to any other providers (and to its local clients).

```
Alice1 -> ProviderA: Submit message
ProviderA: Validate message (e.g., Alice authz to send)
ProviderA -> Alice*: Message notification
ProviderA -> ProviderB: Message notification
ProviderB -> Bob*: Message notification
```

## Alice makes Bob a room administrator

Digress to talk about room policy here...

Since Alice is the owner of the room and the room policy is that administrators have to Alice commits a `ParticipantListPatch` proposal updating Bob's roles in the room to `[regular_user, admin]`.

If the hub provider accepts the commit, it fans it out so all the active participants receive it.

```
Alice1 -> ProviderA: CommitBundle(PLP)
ProviderA: Validate commit sequencing; PLP authz by policy
ProviderA -> Alice1: Commit accepted
```
## Bob adds Cathy and Doug

Since Bob is now an admin, Bob wants to Cathy and Doug. Bob starts by fetching their KeyPackages (assuming Bob already has consent from Cathy and Doug). Then Bob commits a `ParticipantListPatch` proposal adding Cathy as both a regular user and administrator, and Doug as a regular user; and Add proposals for all of Cathy's and Doug's KeyPackages.

If the hub provider accepts the commit, it fans it out so all the active participants receive it, and also forwards the welcome message to Cathy's and Doug's providers.

Once any of the other clients in group receive the commit, or the newcomers receive the welcome, that client can send a message which will be received by the newcomers (assuming it is acceptable to the hub provider).

```
Bob1 -> ProviderB: Get key material for Cathy@C and Doug@D in Room1
ProviderA -> ProviderA (Hub): KeyMaterialRequest([Cathy], C, for Room1)
ProviderA -> ProviderA (Hub): KeyMaterialRequest([Doug], D, for Room1)
ProviderA -> ProviderC: KeyMaterialRequest(Cathy@C in Room1)
ProviderC -> ProviderB: KeyMaterialResponse(Cathy1, Cathy2)
ProviderB -> ProviderD: KeyMaterialRequest(Doug@C in Room1)
ProviderD -> ProviderB: KeyMaterialResponse(Doug1)
ProviderB -> Bob1: Key material(Cathy1, Cathy2, Doug1)
Bob1 -> ProviderB: Commit(PLP, Add(C1, C2, D1)) + Welcome
ProviderB -> ProviderA: CommitBundle(Commit(PLP, Add(C1, C2, D1)),Welcome, GroupInfo, ratchet_tree)
ProviderA: Validate commit sequencing; PLP authz by policy; Add authz by PL
ProviderA -> ProviderB: Commit accepted
ProviderB -> Bob: Commit accepted
ProviderA -> Alice*: Commit
ProviderA -> ProviderB: Commit
ProviderA -> ProviderC: Commit
ProviderA -> ProviderD: Commit
ProviderB -> Bob's other clients: Commit
ProviderC -> Cathy*: Commit
ProviderD -> Doug*: Commit
```

## Bob leaves the room

Removing oneself from a room deserves a special mention. In MLS a client cannot send a commit which removes itself from the group. Instead the leaver needs to send proposals that will be committed by a remaining member of the group.

Bob sends a single `ParticipantListPatch` proposal
removing himself, a `SelfRemove` proposal for his initiating client, and `Remove` proposals for any of his other clients. An active participant then sends a commit referencing all those proposals together in a timely fashion.

If the commit is accepted by the hub provider, Bob's clients all receive the commit message, so they are aware that they have been removed. This is safe to do because they no longer have the information nececssary to calculate the new decryption key for the group, as it was encrypted only to the remaining members of the group.

```
Bob2 -> ProviderB: CommitBundle(Rem(B1, B3)); ProposalBundle(Rem(B2), PLP)
ProviderB -> ProviderA: CommitBundle(Rem(B1, B2, B3), PLP)
ProviderA: Validate commit sequencing; PLP authz by policy; Rem authz by PL
ProviderA: Enqueue Rem(B2), PLP
ProviderA: [[ fanout Commit ]]
```

## Cathy removes Doug

For Cathy to remove Doug, she commits a `ParticipantListPatch` proposal removing Doug, and `Remove` proposals for each of Doug's clients. Assuming the commit is accepted by the hub, both Doug's clients and the remaining members of the group recieve the commit, but Doug's clients can no longer decrypt messages sent for the group.

```
(similar Commit flow)
```

## Cathy gets a new client

When a user is already a participant of a room, but a specific client is not yet a member of the corresponding MLS group, the client can join by generating an external commit and sending it and the rest of the commit bundle to the hub provider for the room. To construct an external commit, the client needs to first obtain the current GroupInfo object and the `ratchet_tree` for the group. Being an existing participant in the room is a simple way to authorize getting a GroupInfo object.

```
Cathy3 -> ProviderC: Request to join [proof of identity]
ProviderC -> ProviderA: Request to join [proof of idenitty]
ProviderA -> ProviderC: GroupInfo
ProviderC -> Cathy3: GroupInfo
Cathy3 -> ProviderC: CommitBundle()
ProviderC -> ProviderA: CommitBundle()
ProviderA: Validate commit sequencing; external commit authz by PL
ProviderA -> ProviderC: Commit accepted
ProviderC -> Cathy3: Commit accepted
ProviderA -> ProviderB, ProviderC, ProviderD: Commit notification
ProviderA -> Alice*: Commit notification
ProviderB -> Bob*: Commit notification
ProviderD -> Doug*: Commit notification
```

## Consent

Most instant messaging systems have some notion of how a user consents to be added to a room, and how they manipulate this consent.

In the connection-oriented model, once two users are connected, either user can add the other to any number of rooms. In other systems (often with many large and/or public rooms), a user needs to consent individually to be added to a room.

The MIMI consent mechanism supports both models and allows them to coexist. It allows a user to request consent, grant consent, revoke consent, and cancel a request for consent. Each of these consent operations can indicate a specific room, or indicate any room.

A connection grant or revoke does not need to specify a room if a connection request did, or vice versa. A connection grant or revoke does not even need to follow a connection request.

For example, Alice could ask for consent to add Bob to a specific room. Bob could send a connection grant for Alice to add him to any room, or a connection revoke preventing Alice from adding him to any room. Similarly, Alice might have sent a connection request to add Bob for any room (as a connection request), which Bob ignored or did not see. Later, Bob wants to join a specific room administered by Alice. Bob sends a connection grant for the specific room for Alice and sends a Knock request to Alice asking to be added. Finally, Cathy could send a connection grant for Bob (even if Bob did not initiate a connection request to Cathy), and Alice could recognize Cathy on the system and send a connection revoke for her preemptively.

**NOTE:** Many providers use additional factors to apply default consent within their service such as a user belonging to a specific workgroup or employer, participating in a related room (ex: WhatsApp "communities"), or presence of a user in the other user's contact list. MIMI does not need to provide a way to replicate or describe these supplemental mechanisms, since they are strongly linked to specific provider policies.

Consent requests have sensitive privacy implications. The sender of a consent request should receive an acknowledgement that the request was received by the *provider* of the target user. For privacy reasons, the requestor should not know if the target user received or viewed the request. The original requestor will obviously find out if the target grants consent, but a consent revocation/rejection is typically not communicated to the revoked/rejected user (again for privacy reasons).

Consent operations are only sent directly between the acting provider (sending the request, grant, revoke, or cancel) and the target provider (the object of the consent). In other words, the two providers must have a direct peering relationship.

In our example, Alice requests consent from Bob for any room. Later, Bob sends a grants consent to Alice to add him to any room. At the same time as sending the consent request, Alice grants consent to Bob to add her to any room.

**TODO:** decide if we like any of the text below better:

The consent primitive needs to include the following:
- the specific operation (a request for consent, a grant of consent, or a rejection/revocation of consent);
- the user ID of the user requesting consent;
- the user ID of the target user; and
- optionally, the room ID for which the consent was requested.


If the consent primitive does not specify a room, it implies consent for any room. This is a common model for systems that use connection requests. Once a user accepts a connection request, either party is consenting to add the other to any number of rooms (connection requests involve granting two unidirectional consents). In other systems, consent for Alice to add Bob to a soccer fans room does not imply that Alice has permission to add Bob to a timeshare presentations room. Both models are common, so MIMI should be able to support both models.

Note that a user could reject consent for all rooms from a user even if there was a consent request for a specific room, or even no consent request.

## Other ways to join

[[ RWM: TBC ]]

### Joining with an join URL / code

### Joining an open room

### Knocks

### Invites

## Reporting abuse/spam

[[ RWM: TBC ]]

## Alice sends an attachment which Cathy downloads

[[ RWM: TBC ]]

## Getting the internal identitifier for a user

[[ RWM: TBC ]]


## Moderation

out of scope

# Definition of Primitives

## MIMI URL directory

Like the ACME protocol {{?RFC8555}}, the MIMI protocol uses a directory document to convey the HTTPS URLs used to reach certain endpoints (as opposed to hard coding the endpoints).

The directory URL is discovered using the `mimi-protocol-directory` well-known URI.

~~~
GET /.well-known/mimi-protocol-directory
~~~

~~~
enum {
  get(1),
  post(2),
  (255)
} HttpMethod;

enum {
  requestConsent(1),
  updateConsent(2),
  claimKeyMaterial(3),
  updateRoom(4),
  provisionalMessage(5),
  fanoutMessage(6),
  claimGroupInfo(7),
  downloadFile(8),
  reportAbuse(9),
  identifierQuery(10),
  (255)
} MimiPrimitive;

struct {
  MimiPrimitive primitive;
  HttpMethod method;
  HttpsUri uri;
} MimiDirectoryEntry;

struct {
  MimiDirectoryEntry directory<V>;
} MimiDirectory;
~~~

## Obtain consent

Alice can request consent for Bob in the context of a room, or in general. Bob can grant or refuse consent to Alice in the context of a room or in general.

~~~
POST /requestConsent/{targetDomain}
POST /updateConsent/{requesterDomain}

enum {
  cancel(0),
  request(1),
  grant(2),
  revoke(3),
  (255)
} ConsentOperation;

struct {
  ConsentOperation consentOperation;
  IdentifierUri requesterUri;
  IdentifierUri targetUri;
  optional<RoomId> roomId;
  select(consentOperation) {
      case grant:
          KeyPackage clientKeyPackages<V>;
  }
} ConsentEntry;
~~~


## Claim keys

This action attempts to claim initial keying material for all the clients
of a list of users at a single provider. The keying material may not be
reused unless identified as "last resort" keying material.

The target provider's domain name is listed in the request path. KeyPackages
requested using this primitive MUST be sent via the hub provider of whatever
room they will be used in. (If this is not the case, the hub provider will be
unable to forward a Welcome message to the target provider).

~~~
POST /claimKeymaterial/{domain}
~~~

The request body includes the protocol (currently just MLS 1.0), the requesting
user, and one or more target users hosted at the target provider domain. The
request SHOULD include the room ID for which the KeyPackage is intended, as the
target may have only granted consent for a specific room.

For MLS, the request
includes a non-empty list of acceptable MLS ciphersuites, and an MLS
`RequiredCapabilities` object (which contains credential types, non-default
proposal types, and extensions) required by the requesting provider (these lists
can be an empty). The `lastResortAllowed` field SHOULD be false.

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
    select(protocol) {
        case mls10:
            CipherSuite acceptableCiphersuites<V>;
            RequiredCapabilities requiredCapabilities;
            bool lastResortAllowed;
    };
} KeyMaterialRequest;
~~~

A successful response contains a list of users.
For each user in the original request, there is a user status code that
indicates keying material was returned for all the user's clients (`success`),
that keying material was returned for some of their clients (`partialSuccess`),
or a specific user code indicating failure. If the user code is success or
partialSuccess, each client is enumerated in the response. Then for each client
with a *client* success code, the structure includes initial keying material (a
KeyPackage for MLS 1.0). If the client's code is `nothingCompatible`, the
client's capabilities are optionally included (The client's capabilities could
be omitted for privacy reasons.)

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
    select(protocol) {
        case mls10:
            select(clientStatus) {
                case success:
                    KeyPackage keyPackage;
                case nothingCompatible:
                    optional<Capabilities> clientCapabilities;
            }
    };
} ClientKeyMaterial;

struct {
    KeyMaterialUserCode userStatus;
    IdentifiedUri userUri;
    ClientKeyMaterial clients<V>;
} UserKeyMaterial;

struct {
    Protocol protocol;
    UserKeyMaterial users<V>;
} KeyMaterialResponse;
~~~

At minimum, as each MLS KeyPackage is returned to a requesting provider (on
behalf of a requesting IM client), the target provider needs to associate its
`KeyPackageRef` with the target client and the hub provider needs to associate
its `KeyPackageRef` with the target provider. This insures that Welcome
messages can be correctly routed to the target provider and client. These
associations can be deleted after a Welcome message is forwarded or after the
KeyPackage `leaf_node.lifetime.not_after` time has passed.


**ISSUE**: Do we want key material requests to be combined for multiple users?

## Change room policy/participation

Adds, removes, and policy changes to the room are all forms of updating the
room state. They are accomplished using the updateRoom transaction which is
used for updating the room base policy, participation list, or its underlying MLS group.

~~~
POST /updateRoom/{roomId}
~~~

In MLS 1.0, any change to the room base policy document is always expressed
using a `GroupContextExtensions` proposal. Likewise, any change to the
participant list is always communicated via a `ParticipantListPatch`
proposal type. The participant list change needed to add a user MUST happen either before or simultaneously with the corresponding MLS operation.

Removing an active user from a participant list or banning an active participant SHOULD happen simultaneously with any MLS changes made to the commit removing the participant.


[[ RWM: TO REMOVE: "The provider is in an impossible situation handling users removed from the participation list but who are still active participants. It should forward the commit removing Bob as a participant to Bob's clients so that Bob is aware that it has been removed, but it should not forward the commit to Bob's clients because those clients would then be able to form the epoch secret for the new epoch." ]]

A hub provider which observes that an active user has been removed or banned, but still has clients MUST prevent any of those clients from sending or receiving any additional application messages; MUST prevent any of those clients from sending Commit messages; and MUST prevent it from sending any proposals except for `Remove` and `SelfRemove` proposals.


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
} ParticipantListPatch;
~~~

Each user can be added with one or more roles. This list of roles can be
updated. Note that removing a user does not ban the user. To ban a user,
update their role to `banned`.

The updateRoom request body is described below:

~~~ tls
select(room.protocol) {
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
~~~

In this case Alice creates a Commit containing a ParticipantListPatch
proposal adding Bob@b.example, and Add proposals for all Bob's MLS clients.
Alice includes the Welcome message which will be sent for Bob, a
GroupInfo object for the hub provider, and complete `ratchet_tree` extension.

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

## Send provisional message

~~~
POST /provisionalMessage/{roomId}
~~~

If the protocol is MLS 1.0, the request body is an MLSMessage with. a WireFormat of PrivateMessage (an application message).

The response merely indicates if the message was accepted by the hub
provider.

**ISSUE:** Do we want to offer a distinction between regular application messages and ephemeral applications messages (for example "is typing" notifications), which do not need to be queued at the target provider.

## Fan out messages

If the hub provider accepts an application or handshake message (proposal or commit) message, it forward that message to all other providers with active participants in the room and all local clients which are active participants. This is described as fanning the message out. An MLS Welcome message is sent to the providers and local users associated with the `KeyPackageRef` values to which the bulk of the Welcome (the `encrypted_group_info`) was encrypted.

The hub provider also fans out any messages which originate from itself (ex: MLS External Proposals).

The hub can include multiple concatenated `FanoutMessage` objects relevant to the same room.

~~~
POST /fanout/{roomId}
~~~

~~~ tls
struct {
  MLSMessage message;
  /* the hub acceptance time (in milliseconds from the UNIX epoch) */
  uint64 timestamp;
} FanoutMessage;
~~~

If the protocol is MLS 1.0, the request body is one or more of MLSMessage
with wire_format one of PrivateMessage (application message), PublicMessage
(Commit or Proposal), or Welcome.

**NOTE:** Correctly fanning out Welcome messages relies on the hub and target providers storing the `KeyPackageRef` of claimed KeyPackages.

## Claim end-to-end crypto group key information

For Bob's new client to join the MLS group and therefore fully participate
in the room with Alice, Bob needs to fetch the MLS GroupInfo (or analogous).

~~~
POST /claimGroupInfo/{roomId}
~~~

In the case of MLS 1.0, Bob provides a credential proving his client's
real or pseudonymous identity (for permission to join the group).

~~~ tls

struct {
  select (protocol) {
    case mls10:
      SignaturePublicKey requestingSignatureKey;
      Credential requestingCredential;
  };
} GroupInfoRequest;

~~~

The response body contains the GroupInfo and a way to get the ratchet_tree.

~~~ tls
struct {
  GroupInfoCode status;
  select (protocol) {
    case mls10:
      GroupInfo groupInfo;   /* without embedded ratchet_tree */
      RatchetTreeOption ratchetTreeOption;
  };
} GroupInfoResponse;
~~~

**ISSUE**: What security properties are needed to protect a
GroupInfo object in the MIMI context are still under discussion. It is
possible that the requester only needs to prove possession of their private
key. The GroupInfo in another context might be sufficiently sensitive that
it should be encrypted from the end client to the hub provider (unreadable
by the local provider).


## Download files

[[ RWM: TBC ]]

## Report abuse

Abuse reports are only sent to the hub provider.

~~~
POST /reportAbuse/{roomId}
~~~

~~~ tls

struct {
  /* an application message (wire_format = private_message and */
  /* content_type = application) and the key and nonce to decrypt */
  /* just this single application message */
  MLSMessage
  bytes key;
  bytes nonce;
} AbusiveMessage;

struct {
  IdentifierURI reportingUser;
  IdentifierURI allegedAbuserURI;
  opaque group_id<V>;
  uint64 epoch;
  uint32 allegedAbuserLeafIndex;
  AbusiveMessage messages<V>;
  AbuseType reasonCode;
  string note;
  AbusiveMessage messages<V>;
} AbuseReport;
~~~

The response code only indicates if the abuse report was accepted, not if any specific automated or human action was taken.


## Find internal address

~~~
GET /identifierQuery/{domain}
~~~

The request body is described as:

~~~ tls
enum {
  reserved(0),
  handle(1),
  nick(2),
  email(3),
  phone(4),
  partialName(5),
  wholeProfile(6),
  oidcStdClaim(7),
  vcardField(8),
  (255)
} SearchIdentifierType;

struct {
  SearchIdentifierType searchType;
  string searchValue;
  select(type) {
    case oidcStdClaim:
      string claimName;
    case vcardField:
      string fieldName;
  };
} IdentifierRequest;
~~~

The response body is described as:

~~~ tls
enum {
  success(0),
  notFound(1),
  ambiguous(2),
  forbidden(3),
  unsupportedField(4),
  (255)
} IdentifierQueryCode;

enum {
  reserved(0),
  oidcStdClaim(7),
  vcardField(8),
  (255)
} FieldSource;

struct {
  FieldSource fieldSource;
  string fieldName;
  opaque fieldValue<V>;
} ProfileField;

struct {
  IdentifierUri stableUri;
  ProfileField fields<V>;
} UserProfile;

struct {
  IdentifierQueryCode responseCode;
  IdentifierUri uri<V>;
  UserProfile foundProfiles<V>;
} IdentifierResponse;
~~~

**TODO**: The format of specific identifiers is discussed in
{{?I-D.mahy-mimi-identity}}. Any specific conventions which are needed
should be merged into this document.

# Use of MLS DS/AS

MLS defines an abstract Delivery Server (DS) and Authentication Server (AS). This abstract functionality is split across all the MLS clients and the providers, but an important function is provided specifically by the hub provider. The hub is responsible for the ordering of messages, fanning messages out to the appropriate providers, and maintaining the GroupInfo so authorized clients can add themselves to groups using an external commit.

The MLS profile for MIMI requires a concept of a user identifier and a mechanism to associate credentials in MLS clients to MIMI users.

Clients are free to fetch pseudonymous user identifiers from their local provider, and use these in rooms. These pseudonymns need only identify the client's provider to other providers and clients. The actual user identity could be shared among the active participants of the room, by using a credential with selective disclosures, or by signing a binding with both the pseudonymous signature key and the "real" identity signature key and presenting the binding to other members.






# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
