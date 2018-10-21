# Bitcoin Cash Payment Channels
Version 1.0, 2018-10-20

Author: Chris Pacia (ctpacia@gmail.com)

## Abstract

This is a draft spec for a payment channel network for Bitcoin Cash.

## Design

Bi-directional payment channels can be used on the Bitcoin Cash network today[[1]](#maliability), however using them requires direct communication
between peers on the network. Bitcoin's lighting network protocol was designed from the ground up. Instead of doing that we will just build our payment
channel protocol on [libp2p](libp2p.io). Libp2p is a modular networking stack that powers the IPFS project (and many others) and provides modular, extensible,
and powerful networking tools that well exceed the design of the lightning network. 

## Spec

Libp2p requires protocols be assigned an ID. For this protocol we will use:
```go
const ProtocolPaymentChannel protocol.ID = "/bitcoincash/paymentchannel/1.0.0"
```

By using a prefix of `/bitcoincash/` we are reserving the ability to create other Bitcoin Cash protocols in the future. 

The ID also includes a version of `1.0.0` which can be incremented in the future as needed. 

### New Address Format

A new cashaddr version is used to represent a payment channel address. 

The version byte for the new address is `0x15` and the payload is `<address_id:16><libp2p_raw_ed25519_pubkey:32>`

Example:
`zhxknkvyf8xec9kdelz780g3c8sg4n9v3zrj9u2rkkhwdrdhcx2yysccua6mqcp5yx6pp5qeypwhl4cu9hped8v`

Since the libp2p public key never changes a random 16 byte address ID is used to make each address unique. This allows recipeints
(such as merchant software) to associate an order with a given payment channel. 

### Messages

Network messages will be serialized using [proto3](https://developers.google.com/protocol-buffers/docs/proto3) and be varint deliminted.

```protobuf
message ChannelOpen {
    bytes addressID    = 1; // From address
    bytes tempID       = 2; // A unqiue ID to use to identify the channel until the first commitment is made
    bytes pubkey       = 3; // Public key to use in the 2 of 2 multisig
    uint32 feePerByte  = 4; // The requested amount of satoshis per byte to use as a fee on the commitment transaction
    uint64 dustLimit   = 5; // The minimum output size to be included in the commitment transaction
    uint32 delay       = 6; // Number of blocks to delay payments to self when the channel is unilaterally closed
    bytes payoutScript = 7; // The scriptpubkey to be used in the payout
}

message ChannelAccept {
    bytes tempID       = 1; // The temorary ID of the channel
    bool accepted      = 2; // If the channel open request was accepted or rejected
    bytes pubkey       = 3; // Public key for the 2 of 2 multisig
    bytes payoutScript = 4; // The scriptpubkey to be used in the payout
}

message InitialCommitment {
    bytes tempID                = 1; // The temorary ID of the channel
    funding txid                = 2; // The txid of the transaction which will fund the multisig
    bytes serializedTransaction = 3; // A partially signed commitment transaction
}

message InitialCommitmentSignature {
    bytes tempID    = 1; // The temorary ID of the channel
    bytes signature = 2; // A signature for the commitment input
}

message ChannelUpdateProposal {
    bytes channelID             = 1; // The ID of the channel
    bytes serializedTransaction = 3; // A partially signed commitment transaction containing the new proposed payout distribution
}

message UpdateProposalAccept {
    bytes channelID         = 1; // The ID of the channel
    bool accepted           = 2; // Whether or not the update proposal has been accepted
    bytes signature         = 3; // A signature for the commitment input
    bytes revocationPrivKey = 4; // The private key used to sign the breach remody transaction
}

message FinalizeUpdate {
    bytes channelID         = 1; // The ID of the channel 
    bytes revocationPrivKey = 2; // The private key used to sign the breach remody transaction
}

message Shutdown {
    bytes channelID = 1; // The ID of the channel 
    bytes signature = 2; // Signature on the payout transaction
}
```

### Protocol

#### Opening a channel

```
    +-------+                                       +-------+
    |       |--(1)---------   OpenChannel   ------->|       |
    |       |<-(2)--------   ChannelAccept  --------|       |
    |   A   |                                       |   B   |
    |       |--(3)------  InitialCommitment  ------>|       |
    |       |<-(4)--- InitialCommitmentSignature ---|       |
    +-------+                                       +-------+

    - where node A is 'funder' and node B is 'fundee'
```

1. To open a new channel Alice will send a `ChannelOpen` message to Bob. The `addressID` is set to the ID from the address. 

2. Bob respond to the `ChannelOpen` message with a `ChannelAccept` message. If Bob does not accept the channel open request he
can respond with `accepted=false`. Otherwise he should set `accepted=true` and include a public key and payout script.

3. Alice uses Bob's public key and her own to build a 2 of 2 multisig P2SH address and creates (but does not broadcast) a channel funding transaction
paying into the multisig address. Alice then sends a `InitialCommitment` message which includes a signed transaction sending funds from the multisig input back to an
an address she controls. Optionally she can include an output paying Bob's revocable delivery address with an initial amount to push with the first commitment.

4. If an inital push output to Bob was included in the transaction, Bob will sign the transaction and save it locally. He then will build and sign a separate
transaction spending from the multisig input to Alice's revocable delivery address (and to his payout address if an amount was included). The signature on this transaction is 
return to Alice in an `InitialCommitmentSignature` message.

5. Alice validates Bob's signature then builds and saves her full commitment transaction.

6. Finally Alice broadcasts her channel funding transaction.

#### Updating a channel
```
    +-------+                                       +-------+
    |       |--(1)----  ChannelUpdateProposal  ---->|       |
    |   A   |<-(2)-----  UpdateProposalAccept  -----|   B   |
    |       |--(3)------    FinalizeUpdate   ------>|       |
    +-------+                                       +-------+

    - where node A is 'sender' and node B is 'recipient'
```

1) Either Alice or Bob sends a `ChannelUpdateProposal` to the other party containing a proposal to update the payout distribution of the channel.
The signed transaction in the message pays out to the other party's revocable delivery address and to their own address (of the balance is above the dust threshold).

2) The other party signs the transaction and saves the new commitment. They then build and sign a new commitment transaction paying to the other party's
revocable delivery address (if above the dust threshold) and to their own address. They transmit the signature as well as their private key for the previous breach remedy
output to the other party in a `UpdateProposalAccept` message. At this point the new commitment should be saved but they should not yet
consider the payment as received as they do not yet have the other party's breach remedy private key.

3) The sender signs the transaction, saves the new commitment and breach remedy private key, then transmits their private key for the previous 
breach remedy output to the other party in a `FinalizeUpdate` message.

4) The recipient should now consider themselves as having been paid.

##### Closing a channel
```
 +-------+                              +-------+
 |       |--(1)-----  Shutdown  ------->|       |
 |   A   |                              |   B   |
 |       |<-(2)-----  Shutdown  --------|       |
 +-------+                              +-------+
 ```
 1) A node can propose a cooperative shutdown by sending a `Shutdown` message containing a signature on a transaction using the balances
 from the last channel state (if over the dust limit) and paying to the payout address of each party.
 
 2) The other node can complete the shutdown by responding with its signature in a `Shutdown` message.
 
 
 #### Timeouts
 
 If ant any point a node is expecting a message from another node that never arrives, it should consider the channel dead. Implementations
 may wish to attempt either a cooperative or hard shutdown at this point or wait for user action. 


#### Notes
<a name="maliability">[1]</a> Payment channels require a maliability fix, however transactions with known maliability issues
are not currently permitted in the mempool and discussions are currently happening to make the remaining maliability fixes consensus rules.
