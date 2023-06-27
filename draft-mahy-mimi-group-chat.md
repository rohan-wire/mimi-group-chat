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
(channels, groups, chats, rooms), in this document, we will refer to group chats
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
further choose among the acceptable policies. Users on other providers can either
accept the policies of the room or not. However we want to make it as easy as
possible for clients from other providers to comply with the room policy primitives
without enumerating specific features or requiring all clients implementations to
present an identical user experience.

# Terminology

## General Terms

{::boilerplate bcp14-tagged}

This document borrows heavily from the semantics of the Multi-User Chat {{MUC}}
extension of the Extensible Messaging and Presence Protocol (XMPP) {{XMPP}}.

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

The terms MLS client, MLS group, Proposal Commit, group_id, epoch, LeafNode,
GroupContext, GroupContextExtensions Proposal, Credential, CredentialType, and
RequiredCapabilities have the same meanings as in the MLS
protocol {{!I-D.ietf-mls-protocol}}.

An MLS KeyPackage (KP) is used to establish initial keying material in a group,
analogous to DoubleRatchet prekeys, except one KP is used for a client per group
but each recipient does not require a separate one.

An MLS GroupInfo (GI) object is the information needed for a client to join an MLS group
using an External Commit. It changes with each MLS epoch.

MLS DS and AS.

# Basic Operations

The basic lifecycle of a room starts with creating a room and setting its policies and
operating style. Then additional users need to join. There are several methods of
joining, but most providers only support a subset of them. Many systems support multiple
devices or client instances per user.

In a multi-device context, we expect if the user Alice has 3 stable devices, then
all three of Alice's clients/devices will be members of the MLS groups for each room
in which she belongs. If Alice deletes an old client, it should be removed
from all the MLS groups she belongs to; and if Alice adds a new client, it
should immediately join all the groups she belongs to. Finally is Alice joins
or is added to a new multi-device room, all here clients are added near
simultaneously.

A client that is a member of the MLS group corresponding to a room is an
"in-room client".
A user that has at least one in-room client is an "occupant" of the room. It can be
an owner, an admin, or a "regular user" in the room.  A user might also have no
relationship with a room or it might be an outcast (a user who is banned from a
room).

Say Alice is in a room. In what ways can Bob end up in the same room as Alice?

Alice could try to add Bob directly. To do that Alice needs to have the
appropriate room permissions, and Bob needs to allow Alice to fetch the
necessary information (the KeyPackages) to each of Bob's clients.
Depending on the provider and Bob's preferences, and Bob's
relationship with Alice, the provider might not allow Alice to get a KeyPackage
without some explicit consent.

Alternatively Bob could try to join immediately. To do that Bob needs to have
permission to fetch the MLS group's GroupInfo and then join. Bob might be allowed,
for example because the room was open, Bob was already
pre-authorized, or Bob used a joining link.

Finally, in some rooms Bob can "knock" and request admittance into the room.

Let's use the example of an members-only administered room where Alice is an admin,
and examine the ways in which Bob could end up as a regular user in the room. In
this context, "the group" refers to the MLS group corresponding to the room.

1) If Alice and Bob already have a consent relationship that Bob's provider is aware of,
Alice can claim an MLS KeyPackage for each of Bob's clients and add them to the group.
Bob also becomes a regular user of the room.

2) If Alice and Bob don't have a consent relationship, Alice can request one by
indicating that user @alice@example.com (Alice) asks user @bob@example.com (Bob)
to consent to communicating in #room35@example.com (a specific room). If Bob
grants consent, Bob includes a KeyPackage for each of his clients. Bob could grant
consent to Alice for her to add him to any room or just the specific room requested.
(Bob can remove his consent at any time.) Alice uses the provided KeyPackages to
add all Bob's clients to the group.

3) Bob discovers the room and tries to join directly with an external join.
Bob might discover a public group
or Bob may have established a new client that needs to join Bob's existing groups.
An external join requires a GroupInfo object and a
`ratchet_tree` extension. If Bob is pre-authorized as a future occupant of the group,
Bob's client can download the GroupInfo from the room's owning provider.

If Bob is not already pre-authorized, Bob can "knock", requesting to be admitted to
the room. An admin or owner can the decide whether or not to add Bob to the
pre-authorization list. The admin or owner can send the GroupInfo object directly
to Bob's client. Bob's client adds itself and any other of Bob's clients.

4) Bob receives a join link. Bob uses the link to join directly with an external join.
The link (and any optional password) acts as authorization to fetch the GroupInfo.
Bob's client adds itself and any other of Bob's clients.


~~~ aasvg
                  .             +------------+
Alice requests   / \     Yes,   |            |
Bob's KP        /   \    KP     |            |
-------------->+ OK? +--------->|   Alice    |
                \   /           |            |
                 \ /            |   sends    |
                  +             |            |
    No, not authz |             |   Commit   |
                  v             |            |
    +----------------.  CONSENT |   with     |
    |   Alice         |   KP    |            |
    |   requests      +-------->|   Add      |
    |   CONSENT       |         |            |
    +----------------'          +------------+

                                +------------+
    +-------------.             |  admin     |
    |  Bob         |     KP     |  sends     |
    |  sends KNOCK +----------->|  Commit w/ |
    |  with KP     |  answering |  Add       |
    +-------------'   admin     +------------+
                ^     adds Bob
                |
  No, not authz |
                |
Bob tries to    +               +------------+
get GI to      / \     Yes,     |            |
join group    /   \    GI       |            |
------------>+ OK? +----------->|            |
              \   /             |            |
               \ /              |  External  |
                +               |            |
                                |            |
Bob has a       .               |   Join     |
join link      / \     Yes,     |            |
to get GI     /   \    GI       |            |
------------>+ OK? +----------->|            |
              \   /             |            |
               \ /              +------------+
                +
                |
                v
           bad link error
~~~



## Room Capabilities

The following room capabilities are included from MUC.

Hidden vs. Public:
: A public room is discoverable/searchable. A hidden room is not.

Members-Only vs. Open:
: An open room can be joined by any non-banned user. A members-only room can
only be joined by a user in the member list. In an enterprise context, it is
also common for any member of a particular domain, group, or workgroup to be
allowed to add themselves.

Moderated vs. Unmoderated:
: An an unmoderated room, any user in the room can send messages to the room.
In a moderated room, only users who have "voice" can send messages to the room.

Non-anonymous vs. Semi-anonymous:
: In non-anonymous rooms, the expectation is that each user presents
a long-term stable identifier which can be correlated across rooms. In
semi-anonymous rooms, it is acceptable to obtain a pseudonymous identifier
which is unique per room, but would still be associated with the user's
provider.

Not a part of MUC.

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

Not included from MUC.

Password-Protected vs. Unsecured:
: A password-protected room is one that a user cannot enter without first
providing the correct password. An unsecured room can be entered without
entering a password.

Persistent vs. Temporary:
: In MUC, a temporary room is destroyed when the last occupant exits whereas
a persistent room is not destroyed when the last occupant exist. MLS has no
notion of a group with no members.  A persistent room could have

Knock-First
how do you get the knock to the appropriate user/users? do you have a separate channel?


## Permissions

Knock

Join links

Logging Policy

History Policy

Bot Policy

Direct Messaging

## Fixed-membership groups, naming convention:

the usernames of the participants in lexical order, concatenated and separated
by the horizontal tab character. hashed with SHA-256

im:mimi=%23%233qAm84t9_PGgXoe3d9brqUy5YHgDHp82UysnmCf6zjo@example.com



# Authorizing MLS primitives

- create an MLS group
- add a client to a group (Commit with Add)
- fetch a KeyPackage
- join a group (External Commit)
- fetch GroupInfo
- remove a client from a group (Commit with Remove)
- leave a group
- send an application message
- update the policy of the group (ex: Commit with GroupContextExtensions)

In general bare proposals are authorized like a Commit from the proposer, except as described below.

## Create an MLS group

A user who would be an admin or owner of the room, can create the corresponding MLS group.

## commit with Add

~~~~~
if the user of the client to be added is an occupant of the room
    OR
if the room is open
    OR
if the room is flexible_membership AND the "adding user" is a room admin or owner
~~~~~

multi-device check required for all of the above

also check if a logger/bot/system device is present in the room

## access to KeyPackages

~~~~~
if the requester represents the same user as the target KeyPackage
    OR
if the target user consented for the requester to access their KeyPackages (for this specific room or for all rooms)
    OR
if the requester requests a small number of outstanding KeyPackages at a time as allowed by the target provider's policy
    OR
the target explicitly provided a KeyPackage to the requester (ex: knocked)
~~~~~

(requester could include the room name/URL in the request so it gets correlated/decremented)
requester might be "partially trusted", e.g. already in a shared group,
friend-of-friend, has target address is contact list, etc.

## join

how to prove to other members your request is legitimate

~~~~~
if the joining user is a an occupant of the room
    OR
if the target room is open
    OR
if the joining user has a link (and optional password) - how to validate this?
link must be signed by an admin or owner, or by the system user
    OR
joining user is a system logger in a logging room
~~~~~

multi-device check required for all of the above

## access to the GroupInfo

almost same as join step above. this is proving to owning provider you are authorized


## commit with Remove

In a flexible-membership room:
~~~~~
user doing the removing is an admin or owner of the room
    OR
Proposal was sent by a member (SelfRemove)
    OR
Proposal was sent by the system user of the owning provider
    OR
Proposal was sent by the system user corresponding to the removed user's provider
~~~~~

In a fixed-membership room:

MLS is removing clients, so:

~~~~~
any occupant can remove any of its other clients
    OR
the system user of the owning provider can remove any but the last client
    OR
the system user of another provider can remove any clients of that provider
(except the last client).

can't remove authorized logger / system user
~~~~~

## leave an MLS group

you can always send a proposal to leave, EXCEPT if
- you are the only admin/owner of a administered group
- you are the last member of the MLS group


## destroy an MLS group

only if you are the last member of the MLS group


## send an application message

~~~~~
if you are a member or moderator
    OR
if you are a visitor of a moderated room AND have voice
~~~~~

## update group policy

can send a GroupContextExtensions proposal

## other

MLS-level policy

~~~~~
PSK - optional
ReInit - need special rules. future MLS versions. may be useful for moving a room
Update - what is the policy? min_interval: -1 = infinity  max_interval: -1 = inifinity
External Proposals - default allowed for uses above
~~~~~

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
