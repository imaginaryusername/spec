# Bip70 version 2

Version 1.0, 2018-3-23

Author: Chris Pacia (ctpacia@gmail.com)

## Abstract

This document describes an update to the Bip70 payment protocol to modernize it and bring it up to date with the latest trends.

## Motivation

Bip70 has been around for over five years now and has not really been updated. In the intervening time we've seen Google develop proto3 as a replacement for the
proto2 syntax used in Bip70. We've also seen the publication of HTTP/2 and subsequent wide spread adoption. We've also seen Google more or less
combine proto3 and HTTP/2 into a new high performance, universal RPC framework ― gRPC. gRPC is an improvement over straight proto3/HTTP/2 as the protobuf
compiler will automatically generate all the code necessary to populate, serialize, and retrieve the request and response message types. The libraries also make
it extemely easy to spin up compatible clients and servers. Given these advancements we are defining a version 2 of Bip70 designed around gRPC. 

## Specification
```proto
syntax = "proto3";

import "google/protobuf/timestamp.proto";
import "google/protobuf/any.proto";

service PaymentProtocol {
        rpc GetPaymentRequest(Empty) returns (PaymentRequest) {}
        rpc SendPayment(Payment) returns (PaymentACK) {}
}

message Output {
        uint64 amount = 1; // amount is integer-number-of-satoshis
        bytes script  = 2; // usually one of the standard Script forms
}
message PaymentDetails {
        string network                    = 1;  // "main" or "test"
        repeated Output outputs           = 2;  // Where payment should be sent
        google.protobuf.Timestamp time    = 3;  // Timestamp; when payment request created
        google.protobuf.Timestamp expires = 4;  // Timestamp; when this request should be considered invalid
        string memo                       = 5;  // Human-readable description of request for the customer
        bool sendPayment                  = 6;  // Whether the payment should be pushed to the server
        bytes merchant_data               = 7;  // Arbitrary data to include in the Payment message
}
message PaymentRequest {
        uint32 payment_details_version = 1;
        PkiType pki_type               = 2;  // none / x509+sha256
        google.protobuf.Any pki_data   = 3;  // depends on pki_type; typically an X509Certificates object
        PaymentDetails payment_details = 4;  // PaymentDetails
        bytes signature                = 5;  // pki-dependent signature covering serialized PaymentRequest (minus signature)
        
        enum PkiType {
                NONE        = 0
                X509_SHA256 = 1; // The SHA1 option is removed as it's no longer secure.
        }
}
message X509Certificates {
        repeated bytes certificate = 1; // DER-encoded X.509 certificate chain
}
message Payment {
        bytes merchant_data         = 1;  // From PaymentDetails.merchant_data
        repeated bytes transactions = 2;  // Signed transactions that satisfy PaymentDetails.outputs
        repeated Output refund_to   = 3;  // Where to send refunds, if a refund is necessary
        string memo                 = 4;  // Human-readable message for the merchant
}
message PaymentACK {
        Payment payment = 1;  // Payment message that triggered this ACK
        string memo     = 2;  // human-readable message for customer
}
message Empty {}
```

## URI Extensions

Bip72 specifies a URI for a payment request of the format:
```
bitcoincash:?r=https://merchant.com/pay.php?h%3D2a8628fc2fbe
```

To maintain backwards compatibility a URI with a `r` value will be treated as a Bip70 version 1 and follow the original specification.

However, URIs containing `r2` should be treated as Bip70 version 2 and follow this specification. For example:
```
bitcoincash:?r2=https://merchant.com/pay.php?h%3D2a8628fc2fbe
```

Services may wish to maintain version 1 endpoints to support older wallets. The following URI offers both versions as an option and wallets
will select whichever version they understand:
```
bitcoincash:?r=https://merchant.com/pay.php?h%3D2a8628fc2fbe&r2=https://merchant.com/pay.php?h%3D2a8628fc2fbe
```

## Protocol MIME types

Bip71 specified that a custom `Content-Type` be set in the HTTP request or response. Because gRPC uses `application/grpc` and does not permit
a custom `Content-Type` we will discontinue use of Bip71.

