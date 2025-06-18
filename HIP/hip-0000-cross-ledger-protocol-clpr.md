---
hip: <HIP number (assigned by the HIP editor), usually the PR number>
title: Cross-Ledger Protocol (CLPR)
author: Edward Wertz <@edward-swirldslabs>
working-group: Leemon Baird <leemon@hashgraph.com>, Richard Bair <@rbair32>, Jasper Potts <@jasperpotts>, Joseph Sinclair <@jsync-swirlds>, Rohit Sinha <@rsinha>, Kiran Pachhai <@kpachhai>, Torfin Olsen <torfinn@hashgraph.com>, Edward Wertz <@edward-swirldslabs> 
requested-by: Leemon Baird <leemon@hashgraph.com>
type: "Standards Track"
category: "Core"
needs-council-approval: "Yes"
status: "Idea"
created: 2026-06-18
discussions-to:
  <A URL pointing to the official discussion thread. Ex: https://github.com/hashgraph/hedera-improvement-proposal/discussions/000>
updated: <Latest date HIP was updated, in YYYY-MM-DD format.>
requires: 1200
replaces: N/A
superseded-by: N/A
---

## Abstract

This HIP introduces the `Cross-Ledger PRotocol` or `CLPR`, pronounced `clipper`
like the type of ship. `CLPR` is an interledger communication protocol that
uses a connector abstraction to manage `CLPR Application Message` queues 
and payments for message handling between two CLPR enabled endpoints. The 
use of the connector abstraction allows for a clean separation of concerns 
between the `CLPR Connector Protocol`, which performs the secure message 
passing through state proofs, and the `CLPR Application Protocol` which 
enables the  development of interledger applications. CLPR is an extensible 
protocol that eliminates the need for intermediate relays, bridges, or oracles 
between two `CLPR Endpoints` that have had one or more connectors registered 
between them. If two `CLPR Endpoints` are individually asynchronous 
byzantine fault-tolerant (`ABFT`) and they handle the messages in order of 
their sending through connectors, then the whole interledger interaction is 
`ABFT`. CLPR preserves the weaker security profile between the two CLPR 
Endpoints.

In this HIP we specify the CLPR extensions needed to support the 
registration of connectors and message passing between two Hiero ledgers. 
With the implementation of this HIP, every Hiero Ledger can be configured 
to act as a CLPR Endpoint, register connectors to other CLPR Endpoints, 
and support the deployment of `CLPR Applications`.

## Motivation

Anyone can stand up an isolated Hiero ledger, but currently the only way to
facilitate communication between two different Hiero ledgers is to use an
EVM based interledger communication protocol and use an additional relay or 
bridge to monitor ledger state and reflect the interledger communication. This 
may be sufficient for EVM and smart contract based communication, but it does 
not support interledger communication for HTS tokens, HCS messages, or Hiero 
native NFTs. Furthermore, the dependency on relays and third party bridges 
which weakens the trust model of the interledger interaction. None of the 
existing solutions play well with ABFT security constraints or Hiero's
signature scheme for signing block root hashes.

There is a clear need for a Hiero compatible interledger communication
protocol to support Hiero to Hiero asset interactions that are ABFT secure.  

The `Cross-Ledger PRotocol` (`CLPR`) proposed here could also form the basis 
of the communication layer between shards in a sharded Hiero ledger.

## Rationale

This `CLPR` design originates with Dr. Leemon Baird and has been fleshed out
and refined through Hashgraph's architecture team.

Core to the design of CLPR is the connector abstraction which creates a
clean separation of concerns between delivery and payment for interledger
messages in the `CLPR Connector Protocol` and the semantics of handling the
interledger messages in the `CLPR Application Protocol`.

### CLPR Connector Protocol

A `Connector` is a collection of data structures within a `CLPR Endpoint`
that specify the connector's local state and outgoing message queue. Each 
connector has its own public/private ECDSA key pair. When a connector is 
registered with an endpoint, the registration message is signed by the 
connector's private key to establish authenticity and authority to create
the connector.  Part of the signed registration data is the public key to
verify the private key's signature. Two connectors in two different
CLPR Endpoints match if they share the same public key. When two 
connectors are successfully paired between two CLPR Endpoints, a 
bidirectional `Channel` is formed for communicating connector queue state 
changes between the CLPR Endpoints. Before a channel can be formed, each of 
the CLPR Endpoints must have the matching connector public key registered 
along with the data format and trust paradigm used by the remote CLPR 
endpoint. At least one of the CLPR Endpoints will need the valid IP 
Addresses of the other CLPR Endpoint. Two CLPR Endpoints may have many 
connectors between them, but a single connector may only connect two CLPR 
Endpoints. 

In addition to the configuration of the remote CLPR Endpoint, the signed
connector registration data contains submission criteria and payment 
configuration for enqueuing CLPR Application messages destined to the 
remote CLPR Endpoint. The payment configuration also indicates how the 
handling of received CLPR Application messages are paid for on the local 
CLPR Endpoint. 

To support the `ABFT` property, each CLPR Endpoint must ensure that the CLPR 
Application messages are handled in the order they were sent and that 
replies are returned in the order in which the original messages are sent.    

Through bundling the mechanisms of communication, trust, and payments into
the `CLPR Connector Protocol`, the interledger application logic can remain
free of those concerns.

### CLPR Application Protocol

The `CLPR Application Protocol` is an extensible collection of `CLPR 
Application Messages` that enable the communication and  coordination 
between deployed CLPR Applications. Each message must be handled in the 
order received, and a reply message must be sent in response to each message 
in the same order. Each CLPR Application Message is use case specific, 
containing parameters for that use case, and has its own dedicated handler. 

As the set of supported application use cases are expanded, creating new 
types of CLPR Application Messages, the message handlers on CLPR Endpoints 
will need to be updated.

To keep the specification and implementation of this HIP from becoming too 
complex, the initial application use case supported is enabling remote smart 
contract calls. Future HIPS will support the transfer of ERC20 tokens, 
ERC721 NFTs, and Hiero native assets and will not have to modify the core 
mechanics of the `CLPR Connector Protocol` introduced in this HIP.

### Alternatives to `CLPR`

All the existing interledger communication protocols are dependent on the 
existence of relays, bridges, or oracles which observe and translate between 
two different ledgers.  This introduces additional potential points of 
failure in the trust and communication models between ledgers. None of the 
existing interledger communication protocols have strong support for `ABFT` 
semantics. None of the existing interledger communication protocols support 
Hiero's Threshold Signature Scheme (hinTS based TSS) for generating network 
signatures or SHA 384 for Merkle Proofs.  None of the existing interledger 
soltions are architected in a pattern similar to the `Connector` abstraction 
which provides a clean separation between application logic and the logistics 
of transmission, trust, and payment.  

Several interledger solutions use Light Clients which reflect state proofs 
between ledgers to facilitate interledger communication.  Light clients are 
still reliant on relays to observe and reflect the observed state from one 
ledger to another.  The receiving ledger must store the state proofs within 
its state for later use in verification of the message payload.  `CLPR` can 
be viewed as _lighter than light clients_ because the state proofs of one 
ledger are not added to the state of another ledger.  Light clients also do 
not have provisions in their protocol for specifying who pays for the cost 
of validating the state proofs and handling the messages on the receiving 
network.    

## Personas and User Stories

Personas
* _Connector Financier_: Someone willing to finance interledger interactions 
* _dApp Developer_: A developer of interledger applications.   
* _End User_: A person who uses intereldger applications. 

User Stories
* As a Connector Financier, I want to setup connectors between ledgers and 
  make financial arrangements with dApp developers to support the execution 
  of their interledger applications.
* As a dApp developer, I want to create an interledger dApp which will 
  employ specific connectors between specific ledgers. 
* As a user of dApps, from my account on one ledger, I want to invoke a 
  specific function or capability on another ledger.  

## Specification

`CLPR` will be incrementally extended per type of `CLPR Endpoint` and 
application use case. What follows here is a specification for: 
1. The abstract description of the `CLPR Connector Protocol`
2. The abstract description of the `CLPR Application Protocol`
3. The `Remote Contract Call` application use case. 
4. Hiero specific extensions and implementation 
   1. Hiero extension of the `CLPR Connector Protocol`
   2. Hiero implementation of the `CLPR Endpoint`
   3. Hiero implementation of the `Remote Contract Call` use case

### Abstract `CLPR Connector Protocol`

Each type of `CLPR Endpoint` has its own data format for representing 
connector state and trust paradigm for generating state proofs.  If a CLPR 
Endpoint does not understand the state proof mechanism of a remote CLPR 
Endpoint or the data format for the connector state being shared, they 
cannot form a channel of communication between the two connectors.

The `Abstract CLPR Connector Protocol` specifies the minimal content needed 
to register new connectors and share state proofs of connector message queue 
state between endpoints. Each CLPR Endpoint type may require additional data 
upon connector registration or provide additional data in its channel state
proofs beyond the core abstraction. CLPR Endpoints of different types can
only communicate with each other if their deployed software versions
supports communication between those two endpoint types.

#### Abstract Connector Registration

At minimum, the following connector metadata will be included in the 
connector registration transaction and signed by the connector private key:
1. `connector_key` : The connector's public key for pairing
2. `local_clpr_endpoint_type` : indicates the type of local clpr endpoint
3. `local_clpr_endpoint_config` : local endpoint configuration
4. `remote_clpr_endpoint_type` : indicatest the type of remote clpr endpoint
5. `remote_clpr_endpoint_config` : remote endpoint configuration for that type
    1. `clpr_endpoint_ips` : An optional list of CLPR Endpoint IP addresses
    2. `proof_config` : The initial config for validating state proofs

The signature on the hash of the bytes of the connector registration message 
must be verifiable with the public key provided in the `connector_key`. 

The `local_clpr_endpoint_type` indicates the content and data format for the
`local_clpr_endpoint_config`. This content is specific to the local CLPR 
endpoint type and may include parameters for approving the local submission 
of CLPR Application messages to the connector, charging fees for submitted 
messages, and how to pay for the local handling of remotely submitted 
CLPR Application messages.

The `remote_clpr_endpoint_type` indicates the CLPR Endpoint type of the 
remote ledger which entails the data format of the content received and the 
type of state proof used to validate the content. The 
`remote_clpr_endpoint_config` indicates the optional ip addresses to connect 
to when reaching out to the remote endpoint and the initial configuration for 
validating state proofs. If the list of ip addresses is empty, the local 
ledger will only receive connections from the remote ledger. At least one of 
the CLPR Endpoints must be configured with the IP Addresses of the other 
endpoint otherwise neither endpoint will attempt to communicate to the other. 

#### Abstract CLPR Channel Protocol

When one CLPR Endpoint (local) reaches out to a remote CLPR Endpoint, there
is at least 1 registered connector on the local endpoint that has provided
one or more IP addresses to connect to. The process of creating channels
between matching connectors and exchanging queued messages for each
connector is as follows:

1. The initiating endpoint sends the latest state proof validation
   configuration it has for the receiving endpoint. 
   1. In response, the receiving endpoint provides an update of the state 
      proof validation configuration to the latest configuration
2. The receiving endpoint sends the latest state proof validation 
   configuration it has for the initiating endpoint
   1. In response, the receiving endpoint provides an update of the state 
      proof validation configuration to the latest configuration
3. The initiating endpoint starts a series of connector syncs
4. The initiating endpoint sends a message indicating it is done
5. The receiving endpoint starts a series of connector syncs
6. The receiving endpoint sends a message indicating it is done
7. The network connection is closed.

The messages for conveying and updating the state proof validation 
configuration are specific to the `CLPR Endpoint Type` and defined in that 
type's extension of the `CLPR Connector Protocol`.    

The protocol for a `Connector Sync` has the following pattern:

1. The initiating endpoint sends a `proof of queue state` message 
   for its local connector queue state.
   1. If the receiving endpoint does not have a matching connector, a reject
      message is sent in response and nothing further happens in the sync 
      for this specific connector.  
2. Otherwise, the receiving endpoint sends a `proof of queue state` message 
   for its local connector queue state.
3. The initiating endpoint sends a `throttle specification` which determines
   how many queued messages it is willing to receive in this sync.
4. The receiving endpoint responds with its own `throttle specification`.
5. Each endpoint sends a sequence of `proof of message sequence` messages that
   communicate the next messages in the queue while respecting the throttle 
   specifications. 
6. Each endpoint sends a `finished with this sync` message to indicate they are
   ready to proceed to the next connector sync.

#### Abstract Connector Sync Channel Messages

##### Abstgract Proof Of Queue State

A `Proof Of Queue State` Channel Message has two parts.

1. The content of the local connector's queue state.
2. The state proof validating the hash of the bytes of the above content. 

The content of the endpoint's connector queue state is in the data format
determined by the endpoint type and consists of at least the following:

1. `connector_key` : The connector's public key identifier
2. `connector_alias` : The endpoint's optional short name for the connector
3. `in_received` : The latest received incoming message's sequence number
4. `in_running_hash` : The running hash of received messages after processing
   the `in_rec` incoming message.
5. `out_received` : The highest observation of the remote connector's 
   `in_received` value.
6. `out_next_seq_num` : The next unused sequence number for outgoing
   messages.
   
The optional `connector_alias` is assumed to be unique within the context of the
remote endpoint and if specified, it is used in place of the `connector_key` 
in the current connector sync for incoming content. If the 
alias is blank, the `connector_key` in full is required. The purpose of the 
alias is to save space both in communication during the syncs and in 
key-value storage within an endpoint's internal state.

The `in_received` value indicates the highest sequence number of received
messages that have been durably enqueued locally for handling. This value
does not indicate the message has been handled fully, only that it is
guaranteed to be handled in order of receipt. 

The `in_running_hash` is the running hash produced by hashing the `in_received` 
message concatenated with the previous `in_running_hash` value generated 
form the `in_received - 1` message. The first received message has a 
sequence number of `1` and the initial `in_running_hash` value with no 
messages received is an empty sequence of bytes.

The outgoing messages in the interval `(out_rec, out_next_seq_num)` are
stored in the local endpoint's state so they can be provided during
connector syncs. Once an endpoint receives a proof that the remote
connector's state such that `remote.in_received > out_rec`, the local 
connector's queue state is updated such that `out_received := remote.
in_received` and the outgoing messages with sequence number between the old 
`out_received` and the new `out_received` (inclusive) are purged from the local 
endpoint's state.

Each time a new message is enqueued to be sent to the remote connector's
endpoint, it is given the next available sequence number indicated by
`out_next_seq_num`, and `out_next_seq_num` is incremented.

##### Abstract Proof of Message Sequence

The `Proof of Message Sequence` has 3 parts:

1. A list of outgoing CLPR Application messages with sequence numbers
   `(m+1 .. n-1)` where  `m+1 <= n-1`
2. The CLPR Endpoint's state data for storing the `n_th` outgoing message of 
   the connector.
3. The state proof validating the hash of the bytes of the CLPR Endpoint's 
   state entry for the `n_th` outgoing message.

The state data for the `n_th` outgoing message has the at minimum the 
following content:

1. `connector_alias` : The short name for the connector, or the full key.
2. `seq_num` : The sequence number for the outgoing message (`n`).
3. `running_hash` : The running hash after processing the following message.
4. `message` : the next outgoing message in the sequence.

When sending this channel message, it is assumed that the receiving endpoint
has previously received at least the `m_th` outgoing message and that its
running hash from that point forward is accurate. This assumption is
founded on the previously received queue state from the remote connector.

The remote endpoint processes this channel message in the following way:

1. The remote connector's queue state has `in_received >= m`
2. The remote connector's queue state's `in_running_hash` is used to advance
   the running hash through the messages `((in_received+1)..n)`
3. The resulting running hash must match the `running_hash` in the channel
   message's endpoint state data for the `n_th` message.
4. Once the received messages are durably enqueued to be handled, the remote
   connector state will be updated to reflect the receipt of these messages.

##### Abstract Throttle Specification

The throttle specification channel message will be used to shape `Proof 
of Message Sequence` messages, bound their size, and determine how many 
are sent in the current sync.

A `Throttle Specification` channel message has the following form:

1. `max_bytes_per_proof` : The max bytes per `Proof of Message Sequence`
2. `max_num_proofs` : The max number of `Proof of Message Sequence` messages
3. `max_message_count` : The max number of CLPR messages to send. 
4. `max_message_bytes` : The max bytes allowed across all CLPR messages sent

The `max_bytes_per_proof` throttle will place an upper bound on the max size of
a single `Proof of Message Sequence`.


### Abstract `CLPR Application Protocol`

### Application Use Cases

#### Remote Contract Call

### Hiero CLPR Extensions and Implementation

#### Hiero CLPR Endpoint Implementation

##### Connector State
 
#### Hiero CLPR Connector Protocol Extensions

#### Hiero CLPR Remote Contract Call Implementation

### The CLPR SDK

### Impact on Block Nodes

### Impact on Mirror Nodes

## Backwards Compatibility

Interledger Communication is a purely additive capability and has no impact 
on previously existing capabilities. 

## Security Implications

If there are security concerns in relation to the HIP, those concerns should be
explicitly addressed to make sure reviewers of the HIP are aware of them.

## How to Teach This

For a HIP that adds new functionality or changes interface behaviors, it is
helpful to include a section on how to teach users, new and experienced, how to
apply the HIP to their work.

## Reference Implementation

The reference implementation must be complete before any HIP is given the status
of “Final”. The final implementation must include test code and documentation.

## Rejected Ideas

Throughout the discussion of a HIP, various ideas will be proposed which are not
accepted. Those rejected ideas should be recorded along with the reasoning as to
why they were rejected. This both helps record the thought process behind the
final version of the HIP as well as preventing people from bringing up the same
rejected idea again in subsequent discussions.

In a way, this section can be thought of as a breakout section of the Rationale
section that focuses specifically on why certain ideas were not ultimately
pursued.

## Open Issues

While a HIP is in draft, ideas can come up which warrant further discussion.
Those ideas should be recorded so people know that they are being thought about
but do not have a concrete resolution. This helps make sure all issues required
for the HIP to be ready for consideration are complete and reduces people
duplicating prior discussions.

## References

A collections of URLs used as references through the HIP.

## Copyright/license

This document is licensed under the Apache License, Version 2.0 --
see [LICENSE](../LICENSE) or (https://www.apache.org/licenses/LICENSE-2.0)
