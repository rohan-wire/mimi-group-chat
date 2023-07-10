---
title: Group Chat Framework for More Instant Messaging Interoperability (MIMI)
abbrev: MIMI Group Chat
docname: draft-mahy-mimi-group-chat-latest
ipr: trust200902
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"area: art
workgroup: MIMI
area: art
category: info
keyword:
 - group chat
 - muc
 - multiuser chat
 - multi-user chat
 - mls
 - xep-45

stand_alone: yes
pi: [toc, sortrefs, symrefs]

venue:
  group: MIMI
  type: Working Group
  mail: mimi@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/mimi/
  github: rohan-wire/mimi-groupchat/
#  latest: https://github.com/rohan-wire/mimi-groupchat/latest

author:
 -  ins: R. Mahy
    name: Rohan Mahy
    organization: Wire
    email: rohan.mahy@wire.com

normative:

informative:
  XMPP: RFC6120
  MUC:
    target: http://xmpp.org/extensions/xep-0045.html
    title: Multi-User Chat
    author:
     - name: Peter Saint-Andre
       mail: xsf@stpeter.im
    date: 2022-12-01
    seriesinfo:
       "XSF XEP": "0045"
  DoubleRatchet:
    target: https://signal.org/docs/specifications/doubleratchet
    title: The Double Ratchet Algorithm
    author:
     - name: Trevor Perrin
       organization: Signal
     - name: Moxie Marlinspike
       organization: Signal
    date: 2016-11-20
  I-D.mahy-mls-room-policy-ext:
    target: https://datatracker.ietf.org/doc/html/draft-mahy-mls-room-policy-ext-00
    title: A Messaging Layer Security (MLS) extension for More Instant Messaging Interoperability (MIMI) room policies
    author:
     - name: Rohan Mahy
       organization: Wire
    date: 2023-07-11


--- abstract

This document describes a group instant messaging ("group chat") semantic framework
for the More Instant Messaging Interoperability (MIMI) Working Group. It describes
several properties and policy options which can be combined to model a wide
range of chat and multimedia conference types. It also describes how to build
these options as an overlay to Messaging Layer Security (MLS) groups and to
authorize MLS primitives.

--- middle

# Introduction

Group instant messaging ("group chat") is available on dozens of messaging
platforms. It can be used in a proliferation of modes, from a three-person
adhoc chat group, to an open discussion group, to a fully-moderated
auditorium-style group, and in many other configurations.  While they go by many names,
(channels, groups, chats, rooms), in this document we will refer to group chats
as rooms.

Making rooms interoperable across existing clients is challenging, as rooms and clients
can support different policies and capabilities across vendors and providers.
Our goal is to balance the policy and authorization goals of the room with the
policy and authorization goals of the end user, so we can support a broad range
of vendors and providers. We need to map these functions onto the primitives of
the underlying Messaging Layer Security (MLS) protocol {{!I-D.ietf-mls-protocol}}
used for end-to-end encryption.

We assume that each room is owned by one provider at a time. The owning provider
controls the range of acceptable policies. The user responsible for the room can
further choose among the acceptable policies. Users (regardless if on other providers) can either
accept the policies of the room or not. However we want to make it as easy as
possible for clients from other providers to comply with the room policy primitives
without enumerating specific features or requiring all clients implementations to
present an identical user experience.

This work is largely complimentary to the work of specifying an MLS Distribution
Service for MIMI in {{?I-D.robert-mimi-delivery-service}}.

# Terminology

## General Terms

{::boilerplate bcp14-tagged}

This document borrows heavily from the semantics of the Multi-User Chat {{MUC}}
extension of the Extensible Messaging and Presence Protocol (XMPP) {{XMPP}}.
The terms used here differ from MUC in two important ways.
First, MUC has separate notions of an "affiliation" (which effectively applies to a user),
and a "role" (which effectively applies to a client). This document describes
roles that are assigned to users, but permissions associated with those roles
apply to the user or to a client of such a user, depending on the context. For
example, while a user might have an admin role authorizing her to add a new user
to a group chat, only one of her clients which is a member of the associated
MLS group is actually capable of adding the new user's clients to the group.
Second, MLS uses the term *member* to refer to clients that have the keying material
for the group in a particular epoch. To avoid confusion, the MUC affiliation
"member" corresponds to the user role "regular-user" in this document, and "visitor"
becomes a role associated with a user.

The terms in this document and {{?I-D.ralston-mimi-terminology}} have not yet
been aligned.

**Room**:
: A room, also known as a chat room or group chat, is a virtual space users
figuratively enter in order to participate in text-based conferencing.
When used with MLS it typically has a 1:1 relationship with an MLS group.

**User**:
: A single human user or automated agent (ex: chat bot) with a distinct identifiable
representation in a room.

**Client**:
: An instant messaging agent instance associated with a specific user account on a
specific device. For example, the mobile phone instance used by the user
@alice@example.com.

**Multi-device support**:
: A room which supports users, each of which can have multiple client instances
(ex: a Desktop client and mobile phone).

**Role**:
: A long-lived position reflecting the privilege level of a *user* in a room.
Possible values are owner, admin, regular-user, visitor, none, or outcast.
A role closely maps to the MUC concept of affiliation, not the MUC concept of role.

**Occupant**:
: A user that has at least one client in the corresponding MLS group,
*and* a role of owner, admin, regular-user, or visitor.

**Room ID**:
: An identifier which uniquely identifies a room.

**User ID**:
: An internal identifier which uniquely identifies a user.

**Nickname**:
: The identifier by which a user is referred inside a room. Depending on the
context it may be a display name, handle, pseudonym, or temporary identifier.
The nickname in one room need not correlate with the nickname for the same user
in a different room.

**Client ID**:
: An internal identifier which uniquely identifies one client/device instance
of one user account.

**Persistent vs. Temporary rooms**:
: In MUC, a temporary room is destroyed when the last occupant exits whereas
  a persistent room is not destroyed when the last occupant exist. As MLS has no
  notion of a group with no members, a persistent room could consist of
  a sequence of distinct MLS groups, zero or one of which would exist at a time.

## Moderation Terms

**Knock**:
: To request entry into a room.

**Ban**:
: To remove a user from a room such that the user is not allowed to re-enter
the room (until and unless the ban has been removed). A banned user has a
role of "outcast". Typically this action is only used in an open or semi-open
room. It is distinct than merely removing a user from an administered group.

**Kick**:
: To temporarily remove a participant or visitor from a room. The user is allowed
to re-enter the room at any time.

**Voice** *(noun)*:
: The privilege to send messages in a room.

**Revoke Voice**:
: To remove the permission to send messages in a room.

**Grant Voice**:
: To grant the permission to send messages in a room.

**Moderator**:
: A client whose user has a role of admin or owner in the room. A moderator
can ban users, and (when allowed in the room) can grant and revoke voice, kick
individual clients, and grant access (ex: in response to a knock).

## User Roles

**System-user**:
: A virtual user of the one of the providers with specific abilities, for example
  to remove users with deleted accounts from all groups associated with that
  user.

**Room Owner**:
: The user who created the room or a user who has been designated by the room
  creator or owner as someone with owner status (if allowed); an owner is allowed
  to change the room configuration and destroy persistent rooms, in addition to all admin
  status. A room owner has the role of "owner".

**Room Administrator**:
: A user empowered by the room owner to perform administrative functions such
as adding, removing, or banning users; however, a room administrator is not
allowed to change the room configuration or to destroy a persistent room.
A room administrator has the role "admin".

**Regular-user**:
: A user that is either preauthorized to join the room or that has at least
one client that is a member of the corresponding MLS group; and that has no
other more specific role in the room. It has the role of "regular-user"

**Visitor**:
: In a moderated room, a user who does not automatically have voice when added to
  a room. (in contrast to a regular-user). A visitor has a role of "visitor".

**Outcast**:
: A user who has been banned from a room. An outcast has a role of "outcast".

## MLS Terms

The terms MLS client, MLS group, (MLS group) member, Proposal, bare proposal,
Commit, External Commit, external join, group_id, epoch,
LeafNode, KeyPackage, GroupInfo,
GroupContext, GroupContextExtensions Proposal, Distribution Service (DS),
Authentication Service (AS), Credential, CredentialType, and
required_capabilities have the same meanings as in the MLS
protocol {{!I-D.ietf-mls-protocol}}.

An MLS **KeyPackage** (KP) is used to establish initial keying material in a group,
analogous to DoubleRatchet {{DoubleRatchet}} prekeys, except one KP is used for
a client per group but each recipient does not require a separate one.

An MLS **GroupInfo** (GI) object is the information needed for a client to
externally join an MLS group using an External Commit. The GroupInfo
changes with each MLS epoch.

MLS **group membership** means that the client is included as a LeafNode in
the MLS group and has the information necessary to encrypt to and decrypt
from the group.

**MLS group agreement** refers to the property of MLS that for every epoch in an
MLS group, every member can verify that they have the same membership and
the same GroupContext. By using the `room_policy` MLS extension
{{I-D.mahy-mls-room-policy-ext}}, this document uses this property to insure that every
MLS member has the room policy and can independently authorize any MLS action
taken within the group.

# Joining Rooms

The basic lifecycle of a room starts with creating a room and setting its policies and
operating style. Then additional users need to join it, leave it, send messages in it,
and eventually delete it. There are several methods of
joining, but most systems only implement a user interface for a subset of them.
In any case, most room policies focus on what **users** are allowed to do, while MLS
primitives focus on creating MLS groups and adding or removing MLS **clients** from them.

Many messaging systems support multiple devices or client instances per user.
In a multi-device context, we expect if the user Alice has 3 stable devices, then
all three of Alice's clients/devices will be members of the MLS groups for each room
in which she belongs. If Alice deletes an old client, it should be removed
from all the MLS groups she belongs to; and if Alice adds a new client, it
should immediately join all the groups she belongs to. Finally if Alice joins
or is added to a new multi-device room, all here clients are added near
simultaneously.

For the purposes of mapping clients to users, we adopt some specific terms.
A client that is a member of the MLS group corresponding to a room is an
"in-room client".
A user that has at least one in-room client is an "occupant" of the room. It can be
an owner, an admin, a visitor, or a "regular user" in the room.  A user might also have no
relationship with a room or it might be an outcast (a user who is banned from a
room).

Say Alice is already in a room. In what ways can Bob end up in the same room as Alice?

Alice could try to add Bob directly. To do that Alice needs to have the
appropriate room permissions, and Bob needs to allow Alice to fetch the
necessary information (the KeyPackages) to each of Bob's clients.
Depending on the provider and Bob's preferences, and Bob's
relationship with Alice, the provider might not allow Alice to get a KeyPackage
without some explicit consent.

Alternatively Bob could try to join immediately. To do that Bob needs to have
permission to fetch the MLS group's GroupInfo and then enter the group via and
external join. Bob might be allowed
to fetch the GroupInfo, for example because the room was open, because Bob was already
pre-authorized, or because Bob used a joining link.

Finally, in some rooms Bob can "knock" and request admittance into the room.

Walking through the same scenarios (See {{fig-join-methods}}) in slightly more detail,
let's use the example of an members-only administered room where Alice is an admin,
and examine the ways in which Bob could end up as a regular user in the room. In
this context, "the group" refers to the MLS group corresponding to the room.

1) If Alice and Bob already have a consent relationship that Bob's provider is aware of,
Alice can claim an MLS KeyPackage for each of Bob's clients and add them to the group.
Bob also becomes a regular user of the room.

2) If Alice and Bob don't have a consent relationship, Alice can request one by
indicating that user `@alice@example.com` (Alice) asks user `@bob@example.com` (Bob)
to consent to communicating in `#room35@example.com` (a specific room). (Alice
could also immediately add Bob to the room's pre-authorization list, which would
allow Bob to join as in step 3. However Alice might only grant pre-authorization
contingent on Bob's consent for her to contact Bob directly.)

If Bob grants consent,
Bob includes a KeyPackage for each of his clients. Bob could grant
consent to Alice for her to add him to any room or just the specific room requested.
(Bob can remove his consent at any time.) Alice uses the provided KeyPackages to
add all Bob's clients to the group. If the KeyPackages expire before Alice adds
Bob, she now has consent from Bob and can fetch fresh KeyPackages to add each of
Bob's clients as in step 1.

3) Bob discovers the room and tries to join directly with an external join.
Bob might discover a public and open group
or Bob may have established a new client that needs to join Bob's existing groups.
An external join requires a GroupInfo object and a
`ratchet_tree` extension. If Bob is pre-authorized as a future occupant of the group,
Bob's client can download the GroupInfo from the room's owning provider, send an
External Commit and immediately add the rest of his clients.

If Bob is not already pre-authorized, Bob can "knock", requesting to be admitted to
the room, including KeyPackages for each of his clients. This also creates a
consent for any admin of the room to re-fetch Bob's KeyPackages. An admin or owner
can then decide to use the (provided or newly fetches KeyPackages) to directly
add all of Bob's clients.

4) Bob receives a join link. Bob uses the link to join directly with an external join.
The link (and any optional password) acts as authorization to fetch the GroupInfo.
Bob's client adds itself via External Commit and immediately adds any of Bob's
other clients.


~~~ aasvg
                  .             +------------+
Alice requests   / \     Yes,   |            |
Bob's KP        /   \    KP     |            |
-------------->+ OK? +--------->|            |
                \   /           |            |
                 \ /            |   Alice    |
   No, Bob        +             |   sends    |
   does not authz |             |   Commit   |
                  v             |   with     |
    +----------------.  CONSENT |   Add      |
    |   Alice         |   KP    |            |
    |   requests      +-------->|            |
    |   CONSENT       |         |            |
    +----------------'          +------------+

                                +------------+
    +-------------.             |  admin     |
    |  Bob         |     KP     |  sends     |
    |  sends KNOCK +----------->|  Commit w/ |
    |  with KP     |  answering |  Add       |
    +-------------'   admin     +------------+
                ^     adds Bob
 No, room       |
 does not authz |
                |
Bob tries to    +               +------------+
get GI to      / \     Yes,     |            |
join group    /   \    GI       |            |
------------>+ OK? +----------->|            |
              \   /             |            |
               \ /              |  External  |
                +               |  Join      |
                                |  then add  |
Bob has a       .               |  remaining |
join link      / \     Yes,     |  clients   |
to get GI     /   \    GI       |            |
------------>+ OK? +----------->|            |
              \   /             |            |
               \ /              +------------+
                +
                |
                v
     bad link or password error
~~~
{: #fig-join-methods title="Different ways to join a room using MLS clients"}


# Room Capabilities

In earlier sections we described several different types of rooms.
These are room policies composed from sets of room capabilities.
Each capability has two options. Most group chat use cases can
be composed from these specific capabilities.

**Membership-Style**:
: The overall approach of membership authorization in a room, which could be
  open, members-only (administrated), fixed-membership, or parent-dependent.

**Open room**:
: An open room can be joined by any non-banned user.

**Members-Only room**:
: A members-only room can only be joined by a user in the occupant list, or
  who is pre-authorized. Authorized users can add or remove users
  to the room. In an enterprise context, it is
  also common for users from a particular domain, group, or workgroup to be
  pre-authorized to add themselves to a Members-Only room.

**Fixed-Membership room**:
: Fixed-membership rooms have the list of occupants specified when they are
  created. Other users cannot be added. Occupants cannot leave or be removed,
  however a user can remove all its clients from the associated MLS group.
  The most common case of a fixed-membership room is a 1:1 conversation.
  This room membership style is used to implement Direct Message (DM) and Group DM
  features. Only a single fixed-membership room can exist for any unique set
  of occupants.

**Parent-dependent room**:
: In a parent-dependent room, the list occupants of the room must be a strict
  subset of the occupants of the parent room. If a user leaves or is removed
  from the parent room, that user is automatically removed from any
  parent-dependent rooms of that parent.

**Multi-device vs. Single-device:**
: A multi-device room can have multiple simultaneous clients of the same user
  as participants in the room. A single-device room can have a maximum of one
  client per user in the room at any moment.

**Knock-Enabled vs. Knock-Disabled**:
: In a knock-enabled room, non-banned users are allowed to programmatically
  request entry into the room. In a knock-disabled room this functionality
  is disabled.

**Moderated vs. Unmoderated**:
: An an unmoderated room, any occupant can send messages
  to the room. In a moderated room, only occupants who have "voice" can send messages
  to the room.

In the next section we describe how these room characteristics influence the
rules used to authorize specific MLS primitives.

# Authorizing MLS primitives

Below is a list of MLS primitives:

- create an MLS group
- add a client to a group (Commit with Add)
- fetch a KeyPackage
- join a group (External Commit)
- fetch GroupInfo
- remove a client from a group (Commit with Remove)
- leave a group
- send an application message
- update the policy of the group (ex: Commit with GroupContextExtensions)

In general bare proposals are authorized like a Commit from the proposer,
except as described below.

Most of these primitives are validated by the clients in an MLS group. For
example, the clients in an MLS group each individually verify that a Commit
sent by Alice to add Bob is valid, because Alice is an admin in the
corresponding room.

Some primitives are verified primarily by the responsible provider in MIMI.
This includes, for example, creating a room, validating the first Commit in an MLS group,
fetching KeyPackages, or fetching a GroupInfo.

## Create an MLS group

MLS groups are created locally, but for the purposes of this discussion
we consider the initial Commit message starting epoch 1 (associated with
an already created room) to cause the group to exist on the owning provider
and to be accessible to others.

A user who is an admin or owner of the room, can "create" the corresponding MLS group.
In a persistent group, a regular user can also "create" the corresponding MLS group.

## Send a Commit with Add proposal

In any of these conditions, a client is authorized to send Add proposals in a
Commit:

- if the user of the client to be added is already an occupant of the room; or
- if the room is open; or
- if the room is parent-dependent *and* the client is a member of the parent group; or
- if the room is flexible-membership *and* the "adding user" is a room admin or owner

If the room is single-device, clients must also check that the resulting MLS group
will only contain a single client per user. This check needs to look at the
combination of all proposals included in the Commit.

Whenever a logging device, bot, or system-user is added to an MLS group,
each receiver should verify this is both consistent with the overall room
policy, and acceptable to the user of the client.

## Fetch KeyPackages

Most KeyPackages are designed to be used in a single group. In this case they are
"claimed" and consumed. In MIMI the provider is responsible for authorization checks
for fetching KeyPackages, enforcing rate limiting, etc. The client who created the
KeyPackages (the target client) is responsible for verifying that its KeyPackages
are not used more than once or used in an unintended group.

The requester SHOULD include the destination room or group id context in which the
target KeyPackage will be used (to add the target client) when requesting
KeyPackages. This allows the provider to correlate a claimed KeyPackage with an
Add proposal to prevent DoS and spam. Privacy concerns may dictate that the requester
prefers to omit this information or use a pseudonymous context or rate limited
token such as PrivacyPass {{?I-D.ietf-privacypass-architecture}}.

In any of these conditions, the provider of the target client
is authorized to provide a client KeyPackage:

- if the requester represents the same user as the target KeyPackage; or
- if the target user consented for the requester to access their KeyPackages
  (for this specific room or for all rooms); or
- if the requester requests a small number of outstanding KeyPackages (ex: 3) at a time
  as allowed by the target provider's policy. See Note 1 below.

This check does not apply if the target explicitly provided a KeyPackage to the
requester, for example in a knock or grant of consent.

Note 1: Partially trusting the requester may be more appropriate if the requester
and the target already have an existing relationship, for example, if they are in a
shared group, they have a friend-of-friend relation, or the target address is
already in the requesters contact list or vice versa.

## Send an External Commit (or External Proposal) to add itself.

In any of these conditions, a client with a valid GroupInfo and `ratchet_tree`
is authorized to send an External Commit or External Proposal to Add its own
client to a group:

- if the joining user is already an occupant of the room; or
- if the joining user is already pre-authorized to enter the room; or
- if the target room is open; or
- if the target room is parent-dependent *and* the joining client is a member of
  the parent MLS group; or
- if the joining user proves possession of a valid joining link (and the related
  password or code, if the link required one). See {{room-join-links}} and
  {{room-policy-format-syntax}} for more information about join links.

If the room is single-device, clients must also check that the resulting MLS group
will only contain a single client per user. This check needs to look at the
combination of all proposals included in the Commit.

## Fetch the GroupInfo

This primitive is almost same as join step above. This is proving to the room's
**owning provider** that the requested client is authorized to get the GroupInfo

A provider is authorized to return a valid GroupInfo and `ratchet_tree` for
the target group, in any of these conditions:

- if the joining user is already an occupant of the room; or
- if the joining user is already pre-authorized to enter the room; or
- if the target room is open; or
- if the target room is parent-dependent *and* the requesting client is a member of
  the parent MLS group; or
- if the joining user proves possession of a valid joining link (and the related
  password or code, if the link required one). See {{room-join-links}} and
  {{room-policy-format-syntax}} for more information about join links.

## Receive a Welcome message

A client that receives a Welcome message determines the corresponding room
policies from the GroupContext and independently verifies:

- if the client's policies are compatible with the policy of the room;
- if the user and clients already in the group are acceptable to receiving client; and
- if the client's user consented (directly or indirectly) to being added to the room.

If the policies are incompatible or there is no consent, the client SHOULD
immediately leave the group and silently discard any application messages
received. The client could also report the sender of the Welcome message
for anti-spam/anti-abuse purposes.

## Send a Commit with Remove proposal

The principles of which users can join a room were discussed extensively. The
principles of which clients can leave a room are:

- a user or the user's provider can remove some or all of its clients;
- the owning provider can remove a disruptive client or (all the clients of a) user;
- in flexible membership rooms, an admin or owner can remove (all the clients of)
  another user that has the same or lower access in the room.

To Send a Commit containing a Remove proposal, or to verify such a Commit,
the following conditions apply:

- if the Remove (or SelfRemove) Proposal was sent by an MLS member; or
- if the Remove Proposal was sent by the system user of the owning provider; or
- if the Remove Proposal was sent by the system user corresponding to the
  removed user's provider; or
- if the room is not fixed-membership, **AND** all the following:
  - the removing user is an admin or owner of the room, and
  - an admin cannot remove an owner or system user, and
  - an owner cannot remove a system user, and
  - for any client being removed, all their clients are being removed at the same time.

In addition no Remove proposal can remove the last client from an MLS group. Instead
the last client or the owning provider should delete the group.

## Leave an MLS group

A client can always send a proposal (a Remove proposal or SelfRemove proposal)
to leave, EXCEPT if
- the leaving client is the only admin/owner of a members-only group; or
- the leaving client is the last member of the MLS group.

## Destroy an MLS group

A client can only destroy an MLS group if the client is the last member
of the MLS group.

In rare circumstances (ex: all users in the room were deleted), the owning provider
can safely destroy a room and its corresponding MLS group.

## Send an application message

A client can send an application message under the following conditions:

- the client's user is an occupant of an unmoderated room; or
- the client's user is a "regular user" or moderator of a moderated room; or
- the client is a visitor of the room  AND has been granted "voice" in the room.

## Update the room / group policy

An admin or owner can send a GroupContextExtensions proposal in the MLS group
which changes the policy of the room provided that:

- an admin cannot remove an owner from the pre-authorized list; and
- the new policy cannot result in no occupants with in-room clients; and
- the new policy cannot result in no admins with in-room clients; and
- a fixed-membership room cannot be changed to a flexible-membership room.

[TODO: check this more rigorously]

If clients advertise acceptable policies, a policy change Commit must
be still compatible with all client policies, *or* remove clients incompatible
with the new policy.

## Send other MLS proposal types

We have wandered into the policies described in
Section 6 of {{?I-D.ietf-mls-architecture}}.

- PSK - optional
- ReInit - need special rules. future MLS versions. may be useful for
  moving a room to a new provider.
- Update - what is the policy? minimum and maximum intervals
- External Proposals - default allowed for uses above

# Other Room Policies

The owning provider of a room may also offer additional features such as

- if links to join a room are allowed, and if so:
  - if a password or code is required,
  - their validity period (ex: one week),
  - if they are single or multiple use, and
  - if they are only valid for a particular domain, workgroup, or user;
- if logging is allowed, for example as required in many countries for
  financial transactions and healthcare discussion;
- if chat bots are allowed as users;
- if chat history can be shared with new joiners, and if so:
  - if shared automatically or manually,
  - for how long,
- if Direct Messaging is allow inside the context of a room;
- to what extent meta-data privacy is supported

Various messaging providers offer the option to change the name of a room, add
a subject or topic to a room, rename nicknames, change status, change the display
name of a user, and share presence. These features are not discussed in this
document as some of them are out of scope of MIMI (ex: presence) and they do not
interact strongly with the authorization of MLS primitives.

The policy could also include a machine readable description of the anti-spam
and anti-abuse policies. This document does *not* include such a description,
since these policies are the same for every room, and a policy would have to
be included for every provider with participating users in a room.

**Hidden vs. Public**:
: A public room is discoverable/searchable. A hidden room is not.

**Non-anonymous vs. Semi-anonymous**:
: In non-anonymous rooms, the expectation is that each user presents
  a long-term stable identifier which can be correlated across rooms. In
  semi-anonymous rooms, it is acceptable to obtain a pseudonymous identifier
  which is unique per room, but would still be associated with the user's
  provider.

**Password-Protected vs. Unsecured**:
: A password-protected room is one that a user cannot enter without first
  providing the correct password or code. An unsecured room can be
  entered without entering a password or code.

## Fixed-membership groups, naming convention

This document proposes a room naming convention for fixed-membership groups.

- Take the user IDs of the participants as MIMI URIs,
- sort in lexical order,
- concatenate and separate with the horizontal tab character (U+0009),
- hash with SHA-256,
- base64url encode with no padding
- prepend with two octothorpe characters '#' (U+0023)
- format as a MIMI URI using the provider of the first initiator of the room

For example, consider a fixed-membership group first created by Cathy at provider
`example.com` with users with the following handles and providers:

| Handle | Provider |
|-------:+---------:|
| @cathy | example.com |
| @alice | providerA.example |
| @betty | providerB.example |
| @bobby | providerB.example |
| @willy | providerA.example |

Represented as MIMI URIs and lexically sorted:

~~~
im:mimi=%40alice@providerA.example
im:mimi=%40betty@providerB.example
im:mimi=%40bobby@providerB.example
im:mimi=%40cathy@example.com
im:mimi=%40willy@providerA.example
~~~

Concatenated with tab characters, hashed with SHA-256 (shown in hex):

~~~
c48899b3e98903a80548eebb7f4a9804cba7dedc08ac153b9570f6cb7c5b6072
~~~

Then base64url encoded with no padding:

~~~
xIiZs-mJA6gFSO67f0qYBMun3twIrBU7lXD2y3xbYHI
~~~

As a room name this becomes:

~~~
##xIiZs-mJA6gFSO67f0qYBMun3twIrBU7lXD2y3xbYHI
~~~

And as a fixed-membership room URL in the example.com domain it becomes:

~~~
im:mimi=##xIiZs-mJA6gFSO67f0qYBMun3twIrBU7lXD2y3xbYHI@example.com
~~~

## Room join links

Room join links



~~~
im:mimi=#d_Nv1ZCPWArKtN0vhC_Wqw?join;code=k5KUJgAZuDesTsMVxRP
  @example.com
~~~



~~~ tls-presentation
struct {
  Uri room;
  opaque passcode<V>;
  uint32 expiration; /* seconds since UNIX epoch */
  bool multiple_uses;
  Uri authorized_user;
} CreateJoiningLinkRequest;
~~~


# Room Policy format syntax

This document proposes a room policy document format using the TLS
Presentation Language format {{!RFC8446}}. The complete format is
in {{formal-syntax}}.

This section will explain the fields in the RoomPolicy struct in
chunks of related policy. An elipsis indicates that a chunk has been omitted which
was or will be explained in another part of this section.

## Membership-related policy

~~~ tls-presentation
enum {
  false(0),
  true(1)
} bool;

enum {
  reserved(0)
  open(1),
  members-only(2),
  fixed-membership(3),
  parent-dependent(4),
  (255)
} MembershipStyle;

struct {
  MembershipStyle membership_style;
  Uri parent_room_uri<V>;
  bool multi_device;
  bool knock_allowed;
  bool moderated;
  bool persistent_room;
  bool password_protected;
  ...
} RoomPolicy;
~~~

The `membership style` can be one of the following values as discussed in
{{room-capabilities}}:

- open
- members-only (default)
- fixed-membership
- parent-dependent

If the `membership_style` is parent-dependent the parent_room_uri MUST be set
with the room ID of the parent. Otherwise the field is zero-length.

If multi_device is true (the default), the MLS group may contain multiple
clients per user. If false only a single client can be an MLS member at
one time.

If `knock_allowed` is true, a user can send a knock requesting access to the
target room. If false, a user cannot. This option can only be enabled
if the `membership_style` is members-only. The default is false.

If moderated is true, the room supports granting and revoking voice. The default
is false.

If `persistent_room` is false, the room will be automatically deleted when the
corresponding MLS group is destroyed (when there are no clients in the group).
If `persistent_room` is true, the room policy will remain and a client whose user
has appropriate authorization can create a new MLS group for the same room.
(There is not a 1:1 correlation of MLS group to room ID in a persistent room.)

If `password_protected` is true, the room requires a passcode or passphrase when
a client of a new user requests access to the GroupInfo used to join a group.
The default is false.

## Pre-authorized users

~~~ tls-presentation
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
  Role target_role;
  /* preauth_domain consists of ASCII letters, digits, and hyphens */
  opaque preauth_domain<V>;
  /* the remaining fields are in the form of a URI */
  opaque preauth_workgroup<V>;
  opaque preauth_group<V>;
  opaque preauth_user<V>;
} PreAuthPerRoleList;

struct {
  ...
  PreAuthPerRoleList pre_auth_list<V>;
  ...
} RoomPolicy;
~~~

In `members-only` rooms, it is convenient to pre-authorize specific users--or
users from specific domains, workgroups/teams, and groups--to specific roles.
The workgroup, group, and user are expressed as a Uri. The domain is expressed in
US-ASCII letters, digits, and hyphens only. If the domain is internationalized,
the Internationalized Domain Names {{!RFC5890}} conversion MUST be done before
filling in this value.

Note that the system role is used to authorize external proposals for operations
for *other* users. For example, the system role can be used to authorize a
provider to remove clients from groups when the corresponding user account is
deleted.

## Delivery and Read notifications, Pseudonyms

~~~ tls-presentation
enum {
  optional(0),
  required(1),
  forbidden(2)
} Optionality;

struct {
  ...
  Optionality delivery_notifications;
  Optionality read_receipts;
  bool pseudonymous_ids;
  ...
} RoomPolicy;
~~~

The `delivery_notifications` value can be set to "forbidden",
"optional", or "required". If the value is set to "optional", the client uses its local
configuration to determine if it should send delivery notifications in the group.

The `read_receipts` value can be set to "forbidden",
"optional", or "required". If the value is set to "optional", the client uses its local
configuration to determine if it should send read receipts in the group.

The format for delivery notifications and read receipts is described in
Section 5.12 of {{?I-D.ietf-mimi-content}}.

If `pseudonymous_ids` is true, clients in the MLS group are free to use pseudonymous
identifiers in their MLS credentials. Otherwise the policy of the room is that
"real" long-term identifiers are required in MLS credentials in the room's
corresponding MLS group.

## Link, Logging, History, and Bot policies

~~~ tls-presentation
struct {
  bool on_request;
  Uri join_link;
  bool multiuser;
  uint32 expiration;
  Uri link_requests;
} LinkPolicy;

struct {
  Optionality logging;
  Uri logging_clients<V>;
  Uri machine_readable_policy;
  Uri human_readable_policy;
} LoggingPolicy;

struct {
  Optionality history_sharing;
  Role who_can_share<V>;
  bool automatically_share;
  uint32 max_time_period;
} HistoryPolicy;

struct {
  opaque name<V>;
  opaque description<V>;
  Uri homepage;
  Role bot_role;
  bool can_read;
  bool can_write;
  bool can_target_message_in_group;
  bool per_user_content;
} Bot;

struct {
  ...
  bool discoverable;
  LinkPolicy link_policy;
  LoggingPolicy logging_policy;
  HistoryPolicy history_sharing;
  Bot allowed_bots<V>;
  ...
} RoomPolicy;
~~~

### Link policies

If `discoverable` is true, the room is searchable. Presumably this means the
the only way to join the room in a client user interface is to be added by
an administrator or to use a joining link.

Inside the LinkPolicy are several fields that describe the behavior of links.

If the `on_request` field is true, no joining link will be provided in the
room policy; the client will need to fetch a joining link out-of-band or
generate a valid one for itself. If present, the URI in `link_requests`
can be used by the client to request an invite code.
The value of `join_link` is empty and the other fields are ignored.

If the `on_request` field is false, the `join_link` field will contain a
joining link. If the link will work for multiple users, `multiuser` is true.
The expiration field represents the time, in seconds after the start of the
UNIX epoch (1-January-1970) when the link will expire. The `link_requests` field
can be empty.

### Logging policies

Inside the LoggingPolicy, the `logging` field can be forbidden, optional, or
required. If `logging` is forbidden then the other fields are empty. If
`logging` is required, the list of `logging_clients` needs to contain at least
one logging URI. Each provider should have no more than one logging client at
a time in a room.
The `machine_readable_policy` and `human_readable_policy` fields optionally
contain pointers to the owning provider's machine readable and human readable
logging policies, respectively.
If `logging` is optional and there is at least one `logging_client` then logging
is active for the room.

### Chat history policies

Inside the HistoryPolicy, if `history_sharing` is forbidden, this means that
clients (including bots) are expected to not to share chat history with new
joiners, in which case `who_can_share` is empty, `automatically_share` is false,
and `max_time_period` is zero.

Otherwise `who_can_share` is a list of roles that
are authorized to share history (for example, only admins and owners can share).
The values of none and outcast cannot be used in `who_can_share`. If
`automatically_share` is true, clients can share history with new joiners without
user initiation. The history that is shared is limited to `max_time_period` seconds
worth of history.

### Chat bot policies

Inside the RoomPolicy there is a list of `allowed_bots`. Each of which has
several fields. The `name`, `description`, and `homepage` are merely descriptive.
The `bot_role` indicates if the chat bot would be treated as a `system-user`, `owner`,
`admin`, `regular_user`, or `visitor`.

The `can_read` and `can_write` fields indicate if the chat bot is allowed to
read messages or send messages in the MLS group, respectively.
If `can_target_message_in_group` is true it indicates that the chat bot can
send an MLS targeted message (see Section 2.2 of {{?I-D.ietf-mls-extensions}})
or use a different conversation or out-of-band
channel to send a message to specific individual users in the room. If
`per_user_content` is true, the chat bot is allowed to send messages with distinct
content to each member. (For example a poker bot could deal a different hand
to each user in a chat).

Users could set policies to reject or leave groups with bots rights that are
inconsistent with the user's privacy goals.

## Extensibility of the policy format

~~~ tls-presentation
enum {
  null(0),
  boolean(1),
  number(2),
  string(3),
  jsonObject(4)
} ExtType;

struct {
  opaque name<V>;
  ExtType type;
  opaque value<V>;
} PolicyExtension;

struct {
  ...
  PolicyExtension policy_extensions<V>;
} RoomPolicy;
~~~

Finally, The extensibility mechanism allows for future addition of new room
policies.


# Formal Syntax

Below is the complete TLS Presentation

~~~ tls-presentation
enum {
  false(0),
  true(1)
} bool;

struct {
  /* a valid Uniform Resource Identifier (URI) */
  opaque uri<V>;
} Uri;

enum {
  optional(0),
  required(1),
  forbidden(2)
} Optionality;

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
  Role target_role;
  /* preauth_domain consists of ASCII letters, digits, and hyphens */
  opaque preauth_domain<V>;
  /* the remaining fields are in the form of a URI */
  opaque preauth_workgroup<V>;
  opaque preauth_group<V>;
  opaque preauth_user<V>;
} PreAuthPerRoleList;

enum {
  reserved(0)
  open(1),
  members-only(2),
  fixed-membership(3),
  parent-dependent(4),
  (255)
} MembershipStyle;

struct {
  Optionality logging;
  bool enabled;
  Uri logging_clients<V>;
  Uri machine_readable_policy;
  Uri human_readable_policy;
} LoggingPolicy;

struct {
  bool on_request;
  Uri join_link;
  bool multiuser;
  uint32 expiration;
  Uri link_requests;
} LinkPolicy;

struct {
  opaque name<V>;
  opaque description<V>;
  Uri homepage;
  Role bot_role;
  bool can_read;
  bool can_write;
  bool can_target_message_in_group;
  bool per_user_content;
} Bot;

struct {
  Optionality history_sharing;
  Role who_can_share<V>;
  bool automatically_share;
  uint32 max_time_period;
} HistoryPolicy;

enum {
  null(0),
  boolean(1),
  number(2),
  string(3),
  jsonObject(4)
} ExtType;

struct {
  opaque name<V>;
  ExtType type;
  opaque value<V>;
} PolicyExtension;

struct {
  MembershipStyle membership_style;
  bool multi_device;
  bool knock_allowed;
  bool moderated;
  bool password_protected;
  PreAuthPerRoleList pre_auth_list<V>;
  Uri parent_room_uri;
  bool persistent_room;
  Optionality delivery_notifications;
  Optionality read_receipts;
  bool semi_anonymous_ids;
  bool discoverable;
  LinkPolicy link_policy;
  LoggingPolicy logging_policy;
  HistoryPolicy history_sharing;
  Bot allowed_bots<V>;
  PolicyExtension policy_extensions<V>;
} RoomPolicy;

RoomPolicy room_policy;
~~~


## MLS extension

{{I-D.mahy-mls-room-policy-ext}} defines a new `room_policy` GroupContext
extension to MLS, that holds the room policy document format described in
this document.

# Specific examples

## One-on-one chat

A 1:1 chat is a fixed-membership room with exactly two specific pre-authorized
users: "@alice@providerA.example" (owner) and "@bobby@providerB.example"
(regular-user). For fixed-membership rooms all users in the room must be
explicitly pre-authorized. Each user can each have multiple clients. Also
included are system users from the providers who are authorized to remove
their own users if their accounts have been deleted.

Even if both Alice and Bob "leave the room" and delete the corresponding MLS group,
if either initiates a DM to the other at a later time, the room itself is
persistent and can be associated with a different (new) MLS group.

~~~ tls-presentation
room_policy.membership-style = fixed;
room_policy.persistent_room = true;

room_policy.pre_auth_list[0].target_role = system;
room_policy.pre_auth_list[0].preauth_user[0] =
   "im:mimi=providerA.example";
room_policy.pre_auth_list[0].preauth_user[1] =
   "im:mimi=providerB.example";

room_policy.pre_auth_list[1].target_role = owner;
room_policy.pre_auth_list[1].preauth_user[0] =
   "im:mimi=%40alice@providerA.example";

room_policy.pre_auth_list[2].target_role = regular_user;
room_policy.pre_auth_list[2].preauth_user[0] =
   "im:mimi=%40bobby@providerB.example";
~~~

## Administrated room

In this simple administrated room, Alice is the owner.
If Alice adds another user, the user does not need to be added to the
pre-authorized list while one of the user's clients is a member of the
MLS group, but if the user requires a role other than default one, Alice
will need to add the new user to the pre-authorization list of the new role.

~~~ tls-presentation
room_policy.membership-style = members-only;
room_policy.pre_auth_list[0].target_role = owner;
room_policy.pre_auth_list[0].preauth_user[0] =
   "im:mimi=%40alice@providerA.example";
~~~

## Open room

In this open room, Alice is an admin and can kick or ban
other occupants of the room.

~~~ tls-presentation
room_policy.membership-style = open;
room_policy.pre_auth_list[1].target_role = admin;
room_policy.pre_auth_list[1].preauth_user[0] =
   "im:mimi=%40alice@providerA.example"
~~~

## Semi-open room

In this room, anyone from the "engineering" workgroup of
`example.com` can automatically join as an admin. Anyone
from the `example.net` domain, the "sales" workgroup
of `example.com`, or the "companyX" workgroup of `provider.example`
can join as a regular-user. Other users could be added manually
by an admin, via the join link below, or via a knock.

~~~ tls-presentation
room_policy.membership-style = members-only;
room_policy.persistent_room = true;
room_policy.knock_allowed = true;

room_policy.pre_auth_list[0].target_role = admin;
room_policy.pre_auth_list[0].preauth_workgroup[0] =
   "im:mimi=#engineering@example.com"

room_policy.pre_auth_list[1].target_role = regular_user
room_policy.pre_auth_list[1].preauth_domain[0] =
   "im:mimi=example.net";
room_policy.pre_auth_list[1].preauth_workgroup[0] =
   "im:mimi=#sales@example.com";
room_policy.pre_auth_list[1].preauth_workgroup[1] =
   "im:mimi=#companyX@provider.example";

room_policy.link_policy.on_request = false;
room_policy.link_policy.join_link =
   "im:mimi=#d_Nv1ZCPWArKtN0vhC_Wqw?join;" +
   "code=k5KUJgAZuDesTsMVxRP@example.com";
room_policy.link_policy.multiuser = true;
room_policy.link_policy.expiration =
  1699207200;  /* 2023-11-05 18:00:00 UTC */
room_policy.link_policy.link_requests =
  "https://im.example.com/link-me?room=" +
  "#d_Nv1ZCPWArKtN0vhC_Wqw@example.com";
~~~

## Moderated room

In this moderated room, Alice is an owner and Adam is a admin.
When Bob and Cathy join the group, neither of them can send
messages to the group until Alice or Adam grants them "voice",
most likely using a machine-readable message.

~~~ tls-presentation
room_policy.membership-style = members-only;
room_policy.moderated = true;

room_policy.pre_auth_list[0].target_role = owner;
room_policy.pre_auth_list[0].preauth_user[0] =
   "im:mimi=%40alice@providerA.example";

room_policy.pre_auth_list[1].target_role = admin;
room_policy.pre_auth_list[1].preauth_user[0] =
   "im:mimi=%40adam@providerA.example";
~~~

## Room with mandatory logging

Alice messages her doctor Cathy. Cathy's provider requires logging
of all patient interactions. The logging client shows up in the
"roster"/user-list that Alice sees. Alice's client learns about
the logging even before the first message with Cathy. Because the
room is administered instead of fixed-membership, either
user (since both have at least admin permissions) could add another
user to the room. For example, Alice might include her partner, child,
or parent. Cathy might add a nurse, lab technician, assistant,
intern, or specialist.

~~~ tls-presentation
room_policy.membership-style = members-only;
room_policy.pre_auth_list[0].target_role = owner;
room_policy.pre_auth_list[0].preauth_user[0] =
   "im:mimi=%40alice@providerA.example";

room_policy.pre_auth_list[1].target_role = admin;
room_policy.pre_auth_list[1].preauth_user[0] =
   "im:mimi=%40cathy@medicalProviderC.example";

room_policy.logging_policy.logging = required;
room_policy.logging_policy.enabled = true;

room_policy.logging_policy.logging_clients[0] =
  "im:mimi=%40logging/#Y9agWMglTlaP8F251hmgVQ" +
  "medicalProviderC.example";

room_policy.logging_policy.machine_readable_policy =
  "https://medicalProviderC.example/P3P/im_policy.xml";
room_policy.logging_policy.human_readable_policy =
  "https://medicalProviderC.example/legal/IM+logging.html";
~~~

If instead of being a patient, Alice was an auditor, whose
provider required logging as well, each provider could have its
own authorized logger.

~~~ tls-presentation
room_policy.logging_policy.logging_clients[1] =
  "im:mimi=%40auditlogging/#Y9agWMglTlaP8F251hmgVQ" +
  "providerA.example";
~~~

# IANA Considerations

This document registers the "application/mimi-room-policy" media type.
The registration template follows:


Type name:
: application

Subtype name:
: mimi-room-policy

Required parameters:
: none

Optional parameters:
: version
    version:
    : The MIMI room policy protocol version expressed as a string
    `<major>.<minor>`.  If omitted the version is "1.0", which
    corresponds to the version published in RFC XXXX.

Encoding considerations:
: MIMI room policy documents are represented using the TLS
  presentation language {{!RFC8446}}. Therefore this media type needs to be
  treated as binary data.

Security considerations:
: MIMI room policy documents can contain sensitive information and are
  designed to be transmitted either inside encrypted and authenticated
  Messaging Layer Security (MLS) {{!I-D.ietf-mls-protocol}} messages,
  or over a protocol protected using a secure transport such as HTTP
  {{?RFC9112}} {{?RFC9113}} {{?RFC9114}} over TLS {{!RFC8446}}. The
  security considerations in this document (RFC XXXX) also apply.

Interoperability considerations:
: N/A

Published specification:
: RFC XXXX

Applications that use this media type:
: MLS-based instant messaging applications

Fragment identifier considerations:
: N/A

Additional information:

  * Deprecated alias names for this type: N/A
  * Magic number(s): N/A
  * File extension(s): N/A
  * Macintosh file type code(s): N/A

Person & email address to contact for further information:
: IETF MIMI Working Group <mimi@ietf.org>

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: IETF MIMI Working Group

Change controller:
: IESG

Provisional registration? (standards tree only):
: No

# Security Considerations

While this section is not started, the entire focus of this document is on
authorization.

The goal is to consider and balance consent, authorization, and privacy
requirements from:

- providers
- owners and administrators of rooms
- other users


--- back


# Acknowledgments
{:numbered="false"}

Thanks to Peter Saint-Andre for the clear terminology in XEP-0045 {{MUC}}, much of which was
borrowed here.

# TODOs
{:numbered="false"}

- Check the authorization rules more thoroughly in the "Update the room /
  group policy" Section.
