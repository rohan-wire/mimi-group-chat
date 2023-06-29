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
       email: xsf@stpeter.im
    date: 2022-12-01
    seriesinfo:
       "XSF XEP": "0045"

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

# Terminology

## General Terms

{::boilerplate bcp14-tagged}

This document borrows heavily from the semantics of the Multi-User Chat {{MUC}}
extension of the Extensible Messaging and Presence Protocol (XMPP) {{XMPP}}.

The terms in this document and {{?I-D.ralston-mimi-terminology}} have not yet
been aligned.

Room:
: A room, also known as a chat room or group chat, is a virtual space users
figuratively enter in order to participate in text-based conferencing.
When used with MLS it typically has a 1:1 relationship with an MLS group.

User:
: A single human user or automated agent (ex: chat bot) with a distinct identifiable
representation in a room.

Client:
: An instant messaging agent instance associated with a specific user account on a
specific device. For example, the mobile phone instance used by the user
@alice@example.com.

Multi-device support:
: A room which supports users, each of which can have multiple client instances
(ex: a Desktop client and mobile phone).

User-Affiliation:
: A long-lived association reflecting the privilege level of a user in a room.
Possible values are owner, admin, member, none, or outcast. **

Occupant:
: A user with a user-affiliation of owner, admin, or member.

Client-Role:
: A temporary position of a client in a room, distinct from the
User-Affiliation of the corresponding user. Possible values are moderator,
participant, visitor, and none. **


Room ID:
: An identifier which uniquely identifies a room.

User ID:
: An internal identifier which uniquely identifies a user.

Nickname:
: The identifier by which a user is referred inside a room. Depending on the
context it may be a display name, handle, pseudonym, or temporary identifier.
The nickname in one room need not correlate with the nickname for the same user
in a different room.

Client ID:
: An internal identifier which uniquely identifies one client/device instance
of one user account.

## Moderation Terms

Knock:
: To request entry into a room.

Ban:
: To remove a user from a room such that the user is not allowed to re-enter
the room (until and unless the ban has been removed). A banned user has a
user-affiliation of "outcast".

Kick:
: To temporarily remove a participant or visitor from a room. The user is allowed
to re-enter the room at any time.

Voice (noun):
: The privilege to send messages in a room.

Revoke Voice:
: To remove the permission to send messages in a room.

Grant Voice:
: To grant the permission to send messages in a room.

## MLS Terms

The terms MLS client, MLS group, Proposal, Commit, External Commit,
external join, group_id, epoch, LeafNode, KeyPackage, GroupInfo,
GroupContext, GroupContextExtensions Proposal, Distribution Service (DS),
Authentication Server (AS), Credential, CredentialType, and
RequiredCapabilities have the same meanings as in the MLS
protocol {{!I-D.ietf-mls-protocol}}.

An MLS KeyPackage (KP) is used to establish initial keying material in a group,
analogous to DoubleRatchet prekeys, except one KP is used for a client per group
but each recipient does not require a separate one.

An MLS GroupInfo (GI) object is the information needed for a client to
externally join an MLS group using an External Commit. The GroupInfo
changes with each MLS epoch.

MLS group agreement refers to the property of MLS that for every epoch in an
MLS group, every member can verify that they have the same membership and
the same GroupContext. This document uses this property to insure that every
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
an owner, an admin, or a "regular user" in the room.  A user might also have no
relationship with a room or it might be an outcast (a user who is banned from a
room). [TODO: should we add visitor as a user category or should it stay client specific?]

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
indicating that user @alice@example.com (Alice) asks user @bob@example.com (Bob)
to consent to communicating in #room35@example.com (a specific room). (Alice
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

Members-Only vs. Open:
: An open room can be joined by any non-banned user. A members-only room can
  only be joined by a user in the occupant list. In an enterprise context, it is
  also common for users from a particular domain, group, or workgroup to be
  pre-authorized to add themselves to a Members-Only room.

Knock-Enabled vs. Knock-Disabled:
: In a knock-enabled room, non-banned users are allowed to programmatically
  request entry into the room. In a knock-disabled room this functionality
  is disabled.

Fixed-Membership vs. Flexible-Membership:
: Fixed-membership rooms have the list of occupants specified when they are
  created. Other users cannot be added. Occupants cannot leave or be removed.
  The most common case of a fixed-membership room is a 1:1 conversation. Only a
  single fixed-membership room can exist for any unique set of occupants. By
  contrast, in a Flexible-Membership room, authorized users can add or remove users
  to the room.

Multi-device vs. Single-device:
: A multi-device room can have multiple simultaneous clients of the same user
  as participants in the room. A single-device room can have a maximum of one
  client per user in the room at any moment.

Parent-subset vs. Independent:
: An independent room maintains its list of occupants independently of any other
  room. In a parent-subset room, the list occupants of the room must be a strict
  subset of the occupants of the parent room. If a user leaves or is removed
  from the parent room, that user is automatically removed from parent-subset
  room.

Moderated vs. Unmoderated:
: An an unmoderated room, any user in the room can send messages to the room.
  In a moderated room, only users who have "voice" can send messages to the room.

Hidden vs. Public:
: A public room is discoverable/searchable. A hidden room is not.

Non-anonymous vs. Semi-anonymous:
: In non-anonymous rooms, the expectation is that each user presents
  a long-term stable identifier which can be correlated across rooms. In
  semi-anonymous rooms, it is acceptable to obtain a pseudonymous identifier
  which is unique per room, but would still be associated with the user's
  provider.

Password-Protected vs. Unsecured:
: A password-protected room is one that a user cannot enter without first
  providing the correct password or code. An unsecured room can be
  entered without entering a password or code.

Persistent vs. Temporary:
: In MUC, a temporary room is destroyed when the last occupant exits whereas
  a persistent room is not destroyed when the last occupant exist. MLS has no
  notion of a group with no members. A persistent room could consist of
  a sequence of distinct MLS groups. In practice this may be a distinction
  without a difference.

In the next section we describe how these room characteristics influence the
rules used to authorize specific MLS primitives.


# Authorizing MLS primitives

Below is a list of MLS primitives

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

- if the user of the client to be added is an occupant of the room; or
- if the room is open; or
- if the room is flexible_membership *and* the "adding user" is a room admin or owner

If the room is single-device, clients must also check that the resulting MLS group
will only contain a single client per user. This check needs to look at the
combination of all proposals included in the Commit.

also check if a logger/bot/system device is present in the room

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
token such as PrivacyPass [].

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
- if the joining user proves possession of a valid joining link (and the related
  password or code, if the link required one). See Note 2 below.

Note 2: More text is needed to descrive how to validate this. Some information
in the link must be signed by an admin or owner, or by the system user. [TODO]

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
- if the joining user proves possession of a valid joining link (and the related
  password or code, if the link required one). See Note 2 below.

Note 2: More text is needed to descrive how to validate this. Some information
in the link must be signed by an admin or owner, or by the system user. [TODO]

## Receive a Welcome message

A client that receives a Welcome message determines the corresponding room
policies from the GroupContext and independently verifies:

- if the client's policies are compatible with the policy of the room; and
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
- if the room is flexible-membership, **AND** all the following:
  - the removing user is an admin or owner of the room, and
  - an admin cannot remove an owner or system user, and
  - an owner cannot remove a system user, and
  - for any client being removed, all their clients are being removed at the same time.

In addition no Remove proposal can remove the last client from an MLS group. Instead
the last client or the owning provider should delete the group.

## Leave an MLS group

A client can always send a proposal (a Remove proposal or SelfRemove proposal)
to leave, EXCEPT if
- the leaving client is the only admin/owner of a administered group; or
- the leaving client is the last member of the MLS group.

## Destroy an MLS group

A client can only destroy an MLS group if the client is the last member
of the MLS group.

In rare circumstances (ex: all users in the room were deleted), the owning provider
can safely destroy a room and its corresponding MLS group.

## Send an application message

A client can send an application message under the following conditions:

- the client is an occupant of an unmoderated room; or
- the client is a "regular user" or moderator of a moderated room; or
- the client is a visitor of the room  AND has been granted "voice" in the room.

## Update the room / group policy

An admin or owner can send a GroupContextExtensions proposal in the MLS group
which changes the policy of the room provided that:

- an admin cannot remove an owner from the pre-authorized list; and
- the new policy cannot result in no occupants with in-room clients; and
- the new policy cannot result in no admins with in-room clients; and
- a fixed-membership room cannot be changed to a flexible-membership room.

[TODO: check this more rigorously]

[TODO: if clients advertise acceptable policies, a policy change Commit must
be still compatible with all client policies or also remove clients incompatible
with the new policy.]

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
  - by whom, and
  - with whom;
- if Direct Messaging is allow inside the context of a room;
- to what extent meta-data privacy is supported

The policy could also include a machine readable description of the anti-spam
and anti-abuse policies.

## Room join links

[TODO]

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

And as fixed-membership room URL in the example.com domain it becomes:

~~~
im:mimi=%23%23xIiZs-mJA6gFSO67f0qYBMun3twIrBU7lXD2y3xbYHI@example.com
~~~

# A Room Policy format

This document proposes a room policy document format.

room_policy

Members-Only vs. Open:
open: true/false (default = false)

Knock-Enabled vs. Knock-Disabled:
knock-allowed: true/false (default = true)
(not supported if open = true)
(not supported if fixed-membership = true or if parent-dependent if true)

Fixed-Membership vs. Flexible-Membership:
fixed-membership: true/false (default = false)
(not supported if open = true)

Multi-device vs. Single-device:
multi-device: true/false (default = true)

Parent-subset vs. Independent:
parent-dependent: true/false (default = false)
(not supported if open = true)
   if true, need room ID of parent room

**Alternatively**
Membership-Style:
- open
- administered (default)
- fixed-membership
- parent-dependent


Moderated vs. Unmoderated:
moderated: true/false (default = false)

Hidden vs. Public:
discoverable: true => public (default)
discoverable: false => hidden

Non-anonymous vs. Semi-anonymous:
non_anonymous: true/false (default = true)

Password-Protected vs. Unsecured:
password_protected: true/false (default = false)

Persistent vs. Temporary:
persistent: true/false (default = true)

Other features

logging_policy: forbidden / optional / required
logging_status: disabled

delivery_notification_policy: forbidden / optional / required
delivery_notification_status: enabled

read_receipt_policy: forbidden / optional / required
read_receipt_status: enabled

history_sharing:

links:


bots:
   which ones are enabled


room_roles


pre-authorization
  system
  owner
  admin
  regular-user
    domain
    workgroup
    group
    user
  visitor
  bot


# Example Use

~~~~~
GET /room/{provider}/{roomId}

{
  "policy": {
      "discoverable": true,
      "open": false,
      "membership": "knock-first",
      "moderated": false
      "logging": false,
      "fixed-membership": true,
      "multi-device": true,
       "welcome-directly": true
  },
  "preauth-owner": [],
  "preauth-admin": [
      { "team": "sales@example.com"}
  ],
  "preauth-member": [
      { "domain": "provider1.example"},
      { "domain": "provider2.example"}
  ],
  "preauth-visitor": [],
  "owning-system": [],
  "system-auth": [],


  "user-affiliations": {
       "owner": ["alice@example.com"],
       "admin": [],
       "member": ["bob@example.com"],
       "outcast": []
   }

}
~~~~~




# Extension Description



## JSON Schema

[TODO]


## MLS extension

We would propose an MLS GroupContext extension called
`room_policy`, which is represented as an opaque vector of bytes--
specifically the compact JSON representation of the policy document.


# Specific examples

## One-on-one chat

- fixed-membership = true
- multi-device = true
- owner alice
- admin bob

## administrated room

## open room

## semi-open room

~~~~~
"preauthorized": [
    "domain": ["example.net"],
    "workgroup": ["engineering@example.com",
     "sales@example.com", "company@provider.example"],
    "group": [],
    "user": [],
    "system": []
]
~~~~~

## moderated room




## fixed-membership room with logging

- fixed-membership = true
- muti-device = false
- logging = true
- alice, bob, logger

~~~~~
"system": ["financial_transactions_chat_logger@example.com"]
~~~~~

# IANA Considerations

This document proposes registration of a new media type


# Security Considerations

Capture consent and authorization in several forms

- Owner or administrator of room authorizes a user to join a room
- Consent of the user authorizes messages from the group to the user
- User authorization in a room authorizes a client to join the corresponding MLS group

pre-authorized for room membership vs. room membership vs. MLS group membership

pre-authorized means that any identifiers for specific user,
user groups, workgroups, or domains are allowed to enter
(likewise pre-authorized for admin or owner...)

room membership means the user has explicitly joined or was explicitly granted
permission to join.

MLS group membership means that the client is included as a LeafNode in the MLS group
and has the information necessary to encrypt to and decrypt from the group.



--- back

# Scraps to keep in the Appendix for now

Create, Join, Send Messages, Remove, Destroy, Change Policies.
These operations need to be mapped on MLS primitives.

Since some MLS primitives don't map directly, we need policies that respect privacy
for some sensitive MLS operations like claiming KeyPackages or fetching the GroupInfo.

We also refer to Users as the humans or agents involved in a chat and Clients.

Balance the authorization goals of the room with the authorization goals of the end user.

client is allowed

For example allowing a user to be added to a group without prior approval


## Methods of joining

- joiner asks to join room (knocks)
- joiner uses a room link, possibly with a password
- another user sends a connection request for a 1:1 conversation
- an existing member of a room invites the new user into the room immediately, and
sends MLS Welcome
- an existing member of a room adds the new user to a list of pre-authorized members.
The clients of the new user then add themselves to the corresponding MLS group.
- an existing member of a room suggests the new user join the room and the
corresponding MLS group
- joiner joins an open room or as a client whose user is already pre-authorized
in the room


## Client-Role and User-Affiliation Values

# Acknowledgments
{:numbered="false"}

Thanks to Peter Saint-Andre for the clear terminology in XEP-0045 {{MUC}}, so much of which was
borrowed here.

# TODOs
{:numbered="false"}

- do we ever want to not add clients directly and just add them to a pre-authorization list?
