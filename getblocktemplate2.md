# getblocktemplate2

Version 1.0, 2018-3-23

Author: Chris Pacia (ctpacia@gmail.com)

## Abstract

This document describes a new, more optimized, version of `getblocktemplate` which can be used for pooled mining.

## Motivation

Stratum and getblocktemplate both suck. Statum has come to dominate the mining industry today due to being far more optimized than getblocktemplate.
However, it only allows for centralized pooled mining and has resulted in enormous centralization of the mining industry. Getblocktemplate offers some limited ability
to mine decentralized but it is so unoptimized nobody uses it.

## Design rationale

`getblocktemplate2` will use protocol buffers over a gRPC API. We choose this because it uses a binary encoding which can be much more compact than both
statum and getblocktemplate. At the time stratus was created, protobuf was still new and not well supported. Today protobuf is ubiquitious and there are libraries in nearly every
language. Many tools also exists to marshal protobuf to JSON for easy human readability. 

## Specification
```protobuf
syntax="proto3";

service API {
  // The following three RPCs are expected to be implemented by the Bitcoin node software. 
  // They may be proxied by a mining pool or any other public server.
  rpc SubscribeTemplate (Subscription) returns (SubscribeResponse) {}
  rpc SubmitBlock (Template) returns (SubmissionResponse) {}
  rpc Templates (Worker) returns (stream Template) {}

  // Only mining pools are expected to implement the SubmitShare RPC
  rpc SubmitShare (Share) returns (SubmissionResponse) {}
}

message Subscription {
	Mode mode = 1;

	enume Mode {
		NO_TRANSACTIONS   = 1;
		THIN_TRANSACTIONS = 2;
		FULL_TRANSACTIONS = 3;
	}
}

message SubscribeResponse {
	Status status       = 1;
	uint32 maxSigOpts   = 2;
	uint32 maxBlockSize = 3;
	string workerID     = 4;
	bytes poolKey       = 5;
	
	enum Status {
		OK = 200;
		UNSUPPORTED = 405;
	}
}

message Worker {
	string workerID = 1;
}

message Template {
	string workID                     = 1;
	Header header                     = 2;
	Coinbase coinbase                 = 3;
	TransactionDiff transactions      = 4;
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
	repeated Output outputs = 2;
	MerkleProof merkleProof = 3;

	message Output {
		bytes script = 1;
		uint64 value = 2;
	}

	message MerkleProof {
		uint32 numTransactions = 1;
		repeated bytes hashes  = 2;
	}
}

message TransactionDiff {
	repeated Transaction add    = 1;
	repeated Transaction remove = 2;
}

message Transaction {
	bytes txid              = 1;
	bytes rawTransaction    = 2;
	repeated uint32 depends = 3;
	uint64 fee              = 4;
	uint32 size             = 5;
	uint32 sigops           = 6;
}

message Share {
	string workID     = 1;
	string username   = 2;
	Header header     = 3;
	Coinbase coinbase = 4;
}

message SubmissionResponse {
	Response response = 1;
	bytes signature   = 2;
	
	message Response {
		bool accepted       = 1;
		bytes shareID       = 2;
		string errorMessage = 3;
	}
}
```
