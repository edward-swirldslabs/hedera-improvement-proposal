---
hip: <HIP number (this is determined by the HIP editor)>
title: Event Serialization With Round Number, EventDescriptors, and Multiple Other Parents.
author: Edward Wertz <edward@swirldslabs.com>
working-group: Swirlds Labs Platform Hashgraph Team
type: Standards Track
category: Core
needs-council-approval: Yes
status: Draft
created: 2023-10-25
discussions-to: https://github.com/hashgraph/hedera-improvement-proposal/discussions/827
updated: <comma separated list of dates>
requires: <HIP number(s)>
replaces: <HIP number(s)>
superseded-by: <HIP number(s)>
---

## Abstract

This HIP proposes several changes to the event serialization format.

1. Add the round during which an event was created to the event's meta-data.
2. Represent links to event parents as EventDescriptors, 4-tuples of (Event Hash, Creator NodeId, Generation, Round
   Number).
3. Allow for multiple other parents.
4. Reduce an event's unhashed meta-data to just the event's signature.

## Motivation

### Add Address Book Round Number

When we allow dynamic address books such that nodes can be added or removed without restart, we need a way to indicate
which address book was active when an event was created. A new address book becomes active on a round boundary. Writing
the current round number into an event's meta-data upon creation will provide a way to determine which
address book to use when validating the event.

This round number will also be used to determine when an event has become ancient and is no longer able to come to
consensus if it has not reached consensus yet.

### Represent Links to Event Parents as EventDescriptors

An EventDescriptor is a 4-tuple:

1. Event Hash - The hash of the event.
2. Creator NodeId - The NodeId of the node that created the event.
3. Generation - The generation of the event. (1 + the max of the event's parent's generations)
4. Round Number - The round during which the event was created.

The EventDescriptor is the minimal amount of information needed to identify an event by its hash, which node created
it, and be able to determine when an event has become ancient. It is used in place of the event itself when the
content of the event (it's transactions) are not needed. This encapsulation of meta-data will allow us to further
improve gossip, validation, and consensus algorithms.

### Allow for Multiple Other Parents

An event's ability to come to consensus is based on whether witness events can strongly see the
event through parent links. Allowing for multiple other parents will improve the ability of events to be
strongly seen by witnessing events. This is expected to reduce the number of rounds needed for an event to come to
consensus.

### Just The Signature In Unhashed Meta-Data

All event data that is used in consensus must be hashed and signed to validate that the event has not been tampered
with. After the above changes have been added to the collection of hashed data, the only remaining piece of data
that is unhashed is the signature itself, which cannot be hashed.

## Rationale

This proposal comes from the Swirlds Labs Platform Hashgraph Team which maintains the algorithms and implementation
used for gossiping events and determining consensus. We have bundled these serialization changes together to
minimize the number of times we change the serialization format.

## User stories

Jane is a node operator and pays a per byte data rate for her internet usage. Improved gossip algorithms will reduce
data duplication during node communication and save Jane money.

John is a retail vendor with customer payments over the Hedera network. The time an event takes to come to
consensus is part of the payment processing experience of his customers. A shorter consensus time translates to improved
customer experience.

Jill is retiring her node from the network. Jack is starting a new node in the network. Both want their nodes to
transition into (in)activity outside of network maintenance windows. The ability to dynamically change the active
nodes participating in consensus while keeping the network secure and performant requires the ability to identify
which address book an event is related to when it is created.

## Specification

The following specification describes the bytes on the wire for an event and its contained objects. The data
structures are described in order of their composite nesting, leaves of the tree first. Serialization formats which
have not changed may be omitted.

### NodeId

```

+------------------------------------------------------------------------+
| NodeId Object                                                          |
+------------+---------+---------------------------------+---------------+
| # of bytes | Type    | Description                     | Typical Value |
+------------+---------+---------------------------------+---------------+
| 4 bytes    | Integer | Serialization version of object | 0x00000001    |
+------------+---------+---------------------------------+---------------+
| 8 bytes    | Long    | Node id value                   |               |
+------------+---------+---------------------------------+---------------+
```

### Hash Object

```
+------------------------------------------------------------------------+
| Hash Object                                                            |
+------------+---------+---------------------------------+---------------+
| # of bytes | Type    | Description                     | Typical Value |
+------------+---------+---------------------------------+---------------+
| 4 bytes    | Integer | Serializaiton version of object | 0x00000001    |
+------------+---------+---------------------------------+---------------+
| 4 bytes    | Integer | Hash type                       | 0x58ff811b    |
+------------+---------+---------------------------------+---------------+
| 4 bytes    | Integer | Length of hash                  | 0x00000030    |
+------------+---------+---------------------------------+---------------+
| N bytes    | byte[]  | Hash value (N = length of hash) |               |
+------------+---------+---------------------------------+---------------+
```

### EventDescriptor

```
+------------------------------------------------------------------------+
| EventDescriptor Object                                                 |
+------------+---------+---------------------------------+---------------+
| # of bytes | Type    | Description                     | Typical Value |
+------------+---------+---------------------------------+---------------+
| 4 bytes    | Integer | Serialization version of object | 0x00000003    |
+------------+---------+---------------------------------+---------------+
| H bytes    | Hash    | The hash of the event           |               |
+------------+---------+---------------------------------+---------------+
| 12 bytes   | NodeId  | The node that created the event |               |
+------------+---------+---------------------------------+---------------+
| 8 bytes    | Long    | The generation of the event     |               |
+------------+---------+---------------------------------+---------------+
| 8 bytes    | Long    | The roster round of the event   |               |
+------------+---------+---------------------------------+---------------+
```

### BaseEventHashedData

```
+--------------------------------------------------------------------------------------------------+
| BaseEventHashedData Object                                                                       |
+------------+-----------------------+---------------------------------------------+---------------+
| # of bytes | Type                  | Description                                 | Typical Value |
+------------+-----------------------+---------------------------------------------+---------------+
| 4 bytes    | Integer               | Serialization version of object             | 0x00000004    |
+------------+-----------------------+---------------------------------------------+---------------+
| V bytes    | SoftwareVersion       | The software version that created the event |               |
+------------+-----------------------+---------------------------------------------+---------------+
| 12 bytes   | NodeId                | The node that created the event             |               |
+------------+-----------------------+---------------------------------------------+---------------+
| S bytes    | EventDescriptor       | The descriptor of the event's self parent   |               |
+------------+-----------------------+---------------------------------------------+---------------+
| L bytes    | EventDescriptor[]     | A list of descriptors for other parents     |               |
+------------+-----------------------+---------------------------------------------+---------------+
| 8 bytes    | Long                  | The round the event was created             |               |
+------------+-----------------------+---------------------------------------------+---------------+
| I bytes    | Instant               | The time the event was created              |               |
+------------+-----------------------+---------------------------------------------+---------------+
| T bytes    | Transaction[]         | The transactions within the event           |               |
+------------+-----------------------+---------------------------------------------+---------------+
```

### BaseEvent

```
+--------------------------------------------------------------------------------------------------+
| BaseEvent Object                                                                                 |
+------------+-----------------------+---------------------------------------------+---------------+
| # of bytes | Type                  | Description                                 | Typical Value |
+------------+-----------------------+---------------------------------------------+---------------+
| 4 bytes    | Integer               | Serialization version of object             | 0x00000004    |
+------------+-----------------------+---------------------------------------------+---------------+
| H bytes    | BaseEventHashedData   | The hashed data of the event                |               |
+------------+-----------------------+---------------------------------------------+---------------+
| 4 bytes    | Integer               | The length of the signature = S             |               |
+------------+-----------------------+---------------------------------------------+---------------+
| S bytes    | Byte[]                | The signature of the node signing the event |               |
+------------+-----------------------+---------------------------------------------+---------------+
```

## Backwards Compatibility

This event serialization change is included in v0.45 of the implementation of the platform. The implementation of
v0.45 will support deserialization of the event format used in v0.44 and will re-serialize the events in the older
serialization scheme for the purpose of checking the signature on the events. All new events created by v0.45 will
be in the new serialization scheme. After v0.45 is delivered, the older serialization scheme may not be supported
in future releases.

## Security Implications

N/A

## How to Teach This

The documentation of the serialization scheme change is here in this HIP and as code.

## Reference Implementation

The reference implementation is located in this PR: https://github.com/hashgraph/hedera-services/pull/9344

## Rejected Ideas

No rejected ideas, yet.

## Open Issues

No open issues, currently.

## References

A collections of URLs used as references through the HIP.

* https://github.com/hashgraph/hedera-services/pull/9344

## Copyright/license

This document is licensed under the Apache License, Version 2.0 -- see [LICENSE](../LICENSE)
or (https://www.apache.org/licenses/LICENSE-2.0)
