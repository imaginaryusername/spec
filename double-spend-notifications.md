# Double Spend Notifications

Version 1.0, 2018-3-23

Author: Chris Pacia (ctpacia@gmail.com)

## Abstract

This document describes a new Bitcoin Cash network message for notifying nodes when a double spend of an unconfirmed transaction has occured.

## Motivation

Accepting unconfirmed transactions as payment for real good and services is known to be risky as the protocol cannot guarantee the payment will not be double spent.
One of the easiest ways to double spend an unconfirmed transaction is to exploit a race condition in network propogation. A user can either send the payment directly 
to the merchant (as in the case of Bip70) or broadcast it to a select group of network nodes while simultanously sending a double spend either to different nodes or
directly to mining pools. In such a senario it is likely that at least some miners will see the double spend first, creating a non-zero probability that the double
spend will get mined, even if all miners are using the frist-seen mempool policy. 

Double spend relaying is a way to mitigate this risk. If nodes relay the first double spend of a UTXO around the network, merchants can decline the payment if a double
spend is detected within a few seconds of receiving the payment. The longer merchant listens for double spends before accepting the unconfirmed payment as valid, the
lower the probability that a double spend due to a race condition will be mined.

## Design rationale

To avoid propagating actual double spends around the network we instead relay a double spend proof which has sufficient data to prove a double spend took place but insfficient
data to reconstruct the transaction.

## Specification



The `dblspndnotif` command is defined as follows:

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 36     | outpoint | outpoint | The outpoint that was double spent |
| ?      | proof | double_spend_proof | Proof a double spend took place |

`double_spend_proof`:

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 4     | version | int32_t | Transaction data format version (note, this is signed) |
| 1+      | tx_in count | 	var_int | Number of Transaction inputs |
| 41+      | 	tx_in | 		tx_in[] | A list of 1 or more transaction inputs or sources for coins |
| 32      | tx_out_hash | 	char[32] | The hash of the serialized transaction outputs |
| 4 | lock_time | 	uint32_t | The transaction locktime |

A new inventory type is added:

| Value | Name |
| -----------|:-----------:|
| 4     | DOUBLE_SPEND_NOTIFICATION |

The inventory vector hash shall be set to the hash of the serialized outpoint which was double spent. This implies that it is possible for 
two different double spend notifications to share the same hash, but this is OK since we only care about relaying one double spend notification per input.

## Message Relaying

When a node detects a double spent input it should check to see if it has the corresponding double spend proof in inventory.
If not it should construct a `dblspndnotif` notification for the input and relay it to its peers via `inv` packets.

When receiving an `inv` packet for a double spend notification a node should first check to see if it has the corresponding double spend notification
in inventory. If not, it should download and validate the double spend notification. 

The validation of the double spend notification must fail if:

* The double spent `outpoint` is not in he mempool. 
* The `outpoint` is not in the `proof`.
* The `proof` calculated from the mempool tx equals the `proof` in the double spend notification. 
* The signature script for any of the `proof` inputs is invalid.

If validation is successful the node should relay the `dblspndnotif` message to its peers.

Double spend notifications can be removed from inventory once the outpoint confirms.

## Using Double Spend Notifications

Merchants or payment processors which accept zero confirmation payments should wait T seconds after receiving the payment to see if a 
double spend notification comes off the wire. If so they payment should be declined. T is configurable by merchants or those implementing point of
sale software. It should be set based on the merchant's tolerance for risk.

## Further Considerations

Finally, it should be noted that double spend detection does *not* prevent all types of double spends. Miners are not required to use the first-seen mempool
policy and may still include double spends in their blocks. To prevent these type of double spends futher technical advances are needed.
