# getblocktemplate2

Version 1.0, 2018-3-23

Author: Chris Pacia (ctpacia@gmail.com)

## Abstract

This document describes a new, more optimized, version of `getblocktemplate` which can be used for pooled mining.

## Motivation

Stratum and getblocktemplate both suck. Stratum has come to dominate the mining industry today due to being far more optimized than getblocktemplate.
However, it only allows for centralized pooled mining and has resulted in enormous centralization of the mining industry. Getblocktemplate offers some limited ability
to mine decentralized but it is so unoptimized nobody uses it.

## Design rationale

`getblocktemplate2` will use protocol buffers over a gRPC API. We choose this because it uses a binary encoding which can be much more compact than both
stratum and getblocktemplate. At the time stratum was created, protobuf was still new and not well supported. Today protobuf is ubiquitious and there are libraries in nearly every
language. Many tools also exists to marshal protobuf to JSON for easy human readability. 

## Specification
```protobuf
syntax = "proto3";

service GetBlockTemplate {
  // The following two RPCs are expected to be implemented by the Bitcoin node software. 
  // They may be proxied by a mining pool or any other public server.
  rpc TemplateStream (Subscribe) returns (stream Template) {}
  rpc SubmitBlock (Template) returns (SubmissionResponse) {}

  // Only mining pools are expected to implement the following RPCs
  rpc GetParams (Empty) returns (MiningParams) {}
  rpc SubmitShare (Share) returns (SubmissionResponse) {}
}

message Subscribe {
	Mode mode = 1;

	enume Mode {
		NO_TRANSACTIONS   = 1;
		THIN_TRANSACTIONS = 2;
		FULL_TRANSACTIONS = 3;
	}
}

message Template {
	string jobID                      = 1;
	Header header                     = 2;
	Coinbase coinbase                 = 3;
	Contraints contraints             = 4;
	bool cleanJobs                    = 5;
	TransactionDiff transactions      = 6;
	
	message Contraints {
		uint32 maxSigOpts   = 1;
		uint32 maxBlockSize = 2;
	}
	
	message TransactionDiff {
		repeated Transaction add    = 1;
		repeated Transaction remove = 2;

		message Transaction {
			bytes txid              = 1;
			bytes rawTransaction    = 2;
			repeated uint32 depends = 3;
			uint64 fee              = 4;
			uint32 size             = 5;
			uint32 sigops           = 6;
		}
	}
}

message Header {
	uint64 time                       = 1;
	uint32 height                     = 2;
	bytes prevBlock                   = 3;
	uint32 bits                       = 4;
	uint32 version                    = 5;
	bytes merkleRoot                  = 6;
}

message Coinbase {
	bytes scriptSig         = 1;
	bytes utxoRoot          = 2;
	repeated Output outputs = 3;
	MerkleProof merkleProof = 4;

	message Output {
		bytes script = 1;
		uint64 value = 2;
	}

	message MerkleProof {
		uint32 numTransactions = 1;
		repeated bytes hashes  = 2;
	}
}

message Share {
	string jobID      = 1;
	string username   = 2;
	Header header     = 3;
	Coinbase coinbase = 4;
}

message SubmissionResponse {
	Response response = 1;
	bytes signature   = 2;
	
	message Response {
		bool accepted       = 1;
		bytes ID            = 2;
		string errorMessage = 3;
	}
}

message Empty {}

message MiningParams {
	bytes shareDifficultyTarget = 1;
	bool allowCoinbaseAppend    = 2;
	bytes poolPubkey            = 3;
}
```

## RPCs

#### TemplateStream
The `TemplateStream` RPC opens a stream with the server over which `Template` objects are pushed to the client. A `Subscribe` object containing the transaction `mode` must be sent with the initial RPC.

Each `Template` object is identical for all three transaction `modes` with the exception of if and how the `transactions` are included.

`NO_TRANSACTIONS`: Behaves similar to the current Stratum protocol in that no information about the mempool is sent with each template. This option will use the minimum amount of client and server bandwidth but will not permit the client to select his own transactions. This mode is suitable for small miners who wish to outsource transaction selection to their pool, or for miners connecting to their own local instance of `bitcoind` and who do not wish to modify the template in any way.

`THIN_TRANSACTIONS`: This mode returns `Transaction` objects with each new template minus the `rawTransaction` field. Thus the template will only contain txids and other transaction metadata such as `sigops` and `size`. This mode is useful to miners who wish to inspect and possibly modify the contents of the template without using excessive bandwidth. 

`FULL_TRANSACTIONS`: Returns the full `rawTransaction` data with each transaction in addition to the metadata returned with `THIN_TRANSACTIONS`. This mode is most useful for middleware which inspects, modifies, and proxies the template.

Excluding the transactions, the `Template` contains all the information necessary to mine, including the merkle proof linking the coinbase to the block header which is needed for incrementing the extra nonce field in the coinbase.

If the `THIN_TRANSACTIONS` or `FULL_TRANSACTIONS` mode is used the first `Template` after the stream is opened will include all the relevant mempool transactions in the `transactions.add` field. 

Subsequent templates will only send the transaction differential between the prior and new template using the `add` and `remove` fields and not the full transaction list. The `cleanJobs` boolean is used to signal to the client that a new block has been found and the client should clear its transaction cache and reset it to the transactions found in`transactions.add`.

If the full transaction list is needed again, the client can make another `TemplateStream` RPC request to reset the server and receive the full list of transactions again in the `transactions.add` field.

#### SubmitBlock

Blocks which meet the full network difficulty requirement can be submitted to the network as a `Template` object to the `SubmitBlock` RPC. If the template has been unmodied the `jobID` can be set to the last received `jobID` and the server will use the transactions associated with that ID to validate the block. If modifications have been made to the block the `TransactionDiff` object should be included. The transactions in the diff may omit the full `rawTransaction` and just send the `txid` but if the server does not have the corresponding transaction in its mempool the validation will fail. Note that when UTXO commitments are added to the protocol the `utxoRoot` must be updated if any transactions are added to or removed from the template.

The `SubmissionResponse` is returned in response to the `SubmitBlock` RPC and includes whether the block was accepted or not, the block ID, and an error message if submission failed.

#### GetParams

This RPC is intended to only be implemented by mining pool software. It returns the mining pool's configuration parameters such as the share difficulty target and whether the pool allows additional data to go into the coinbase. The `poolPubkey` can be used to sign share response to serve as a form of a receipt. 

#### SubmitShare

This RPC is intended to only be implemented by mining pool software. The `Share` contains enough information to validate that the header meets the pool's difficulty requirement. If the transactions in the template have not been modified then the `coinbase.merkleProof` field may be omitted. Otherwise `coinbase.merkleProof` must contain the hashes linking the coinbase to the blockheader to prove the correct coinbase transaction was used when mining this share. 

The `SubmissionResponse` is returned in response to the `SubmitShare` RPC. Like with `SubmitBlock` it includes whether the share was accepted or not, the share ID, and an error message if submission failed. Optionally the mining pool can include a signature covering the serialized `response` object as a receipt. 


## Usage in Mining Software

Mining software implementing getblocktemplate2 should allow miners to enter two urls in the runtime options: `MiningPoolURL` and `GetBlockTemplateURL`. The `MiningPoolURL` specifies the endpoint which implements the `GetParams` and `SubmitShare` RPCs
and is where the mining software should submit shares to. While the `GetBlockTempleURL` specifies where to stream the block templates from and where to push solves blocks. 

Both URLs may be the same implying that the user wishes to get the block template from and submit shares to the same place (as is similar to current Stratum mining). However, the miner may opt to get the template from another source, such as his local instance of bitcoind, and excercise is right as a miner to select his own transactions and improve the health of the network. 
