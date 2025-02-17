# DsLink Node.js Implementation Document

This document focuses on the Node.js implementation of the DsLink protocol.

## Overview

![Architecture diagram](./diagrams/architecture.png)

## Node Types

There will exist several DsLink node configurations, which offer a variety of modes that support different features and trade-offs. The choice to run one type or another largely depends on the type of user, machine, and intent the operator has in mind.

### Full Node

A full node offers the largest set of features and highest resolution performance of DIDs, but also requires more significant bandwidth, hardware, storage, and system resource consumption to operate. A full node will attempt to fetch and retain all data associated with the Sidetree operations present in the target system. As such, full nodes are able to quickly resolve DID lookup requests and may feature more aggressive caching of DID state than other node configurations.

### Light Node

A light node is a node that retains the ability to independently resolve DIDs without relying on a trusted party or trusted assertions by other nodes, while minimizing the amount of bandwidth and data required to do so. Light nodes run a copy of the target system's blockchain node and fetch only minimal DsLink data required to create an independent lookup table that enables just-in-time resolution of DIDs.

> NOTE: Light node support is in development.

## Observer

The _Observer_ watches the public blockchain to identify DsLink operations, then parses the operations into data structures that can be used for efficient DID resolutions.
The primary goals for the _Observer_ are to:
1. Maximize ingestion processing rate.
1. Allow horizontal scaling for high DID resolution throughput.
1. Allow sharing of the processed data structure by multiple DsLink nodes to minimize redundant computation.

The above goals lead to the design decision of minimal processing of the operations at the time of ingestion, and deferring the heavy processing such as signature validations to the time of DID resolution.

## Versioning
As the DsLink protocol evolves, existing nodes executing an earlier version of the protocol need to upgrade to execute the newer version of the protocol while remaining backward compatible to processing of prior transactions and operations.

### Protocol Versioning Configuration
The implementation exposes a JSON configuration file with the following schema for specifying protocol version progressions:
```json
[
  {
    "startingBlockchainTime": "An inclusive number that indicates the time this version takes effect.",
    "version": "The name of the folder that contains all the code specific to this protocol version."
  }
]
```

Protocol versioning configuration file example:
```json
[
  {
    "startingBlockchainTime": 1500000,
    "version": "0.4.0"
  },
  {
    "startingBlockchainTime": 2000000,
    "version": "0.5.0"
  }
]
```

![Versioning diagram](./diagrams/versioning.png)

### Orchestration Layer
There are a number of top-level components (classes) that orchestrate the execution of multiple versions of protocol simultaneously at runtime. These components are intended to be independent from version specific changes. Since code in this orchestration layer need to be compatible with all protocol versions, the orchestration layer should be kept as thin as possible.

- Version Manager - This component handles construction and fetching of implementations of protocol versions as needed.
- Batch Scheduler - This component schedules the writing of new operation batches.
- Observer - This component observes the incoming Sidetree transactions and processes them.
- Resolver - This component resolves a DID resolution request.

The orchestration layer cannot depend on any code that is protocol version specific, this means its dependencies must either be external or be part of the orchestration layer itself, such dependencies include:
- Blockchain Client
- CAS (Content Addressable Storage) Client
- MongoDB Transaction Store
- MongoDB Operation Store

### Protocol Version Specific Components
The orchestration layer requires implementation of following interfaces per protocol version:
- `IBatchWriter` - Performs operation batching, batch writing to CAS, and transaction writing to blockchain. Used by the _Batch Scheduler_.
- `ITransactionProcessor` - Used by the _Observer_ to perform processing of a transaction written in a particular protocol version.
- `IOperationProcessor` - Used by the _Resolver_ to apply an operation written in a particular protocol version.
- `IRequestHandler` - Handles REST API requests.


## Blockchain REST API
The blockchain REST API interface aims to abstract the underlying blockchain away from the main protocol logic. This allows the underlying blockchain to be replaced without affecting the core protocol logic. The interface also allows the protocol logic to be implemented in an entirely different language while interfacing with the same blockchain.

### Response HTTP status codes

| HTTP status code | Description                              |
| ---------------- | ---------------------------------------- |
| 200              | Everything went well.                    |
| 400              | Bad client request.                      |
| 401              | Unauthenticated or unauthorized request. |
| 404              | Resource not found.                      |
| 500              | Server error.                            |



### Get latest blockchain time
Gets the latest logical blockchain time. This API allows the Observer and Batch Writer to determine protocol version to be used.

A _blockchain time hash_ **must not** be predictable/pre-computable, a canonical implementation would be to use the _block number_ as the time and the _block hash_ as the _time hash_. It is intentional that the concepts related to _blockchain blocks_ are  hidden from the layers above.

#### Request path
```
GET /time
```

#### Request headers
None.

#### Request body schema
None.

#### Request example
```
Get /time
```

#### Response body schema
```json
{
  "time": "The logical blockchain time.",
  "hash": "The hash associated with the blockchain time."
}
```

#### Response body example
```json
{
  "time": 545236,
  "hash": "0000000000000000002443210198839565f8d40a6b897beac8669cf7ba629051"
}
```



### Get blockchain time by hash
Gets the time identified by the time hash.

#### Request path
```
GET /time/<time-hash>
```

#### Request headers
None.

#### Request body schema
None.

#### Request example
```
Get /time/0000000000000000001bfd6c48a6c3e81902cac688e12c2d87ca3aca50e03fb5
```

#### Response body schema
```json
{
  "time": "The logical blockchain time.",
  "hash": "The hash associated with the blockchain time, must be the same as the value given in query path."
}
```

#### Response body example
```json
{
  "time": 545236,
  "hash": "0000000000000000002443210198839565f8d40a6b897beac8669cf7ba629051"
}
```



### Fetch Sidetree transactions
Fetches Sidetree transactions in chronological order.

> Note: The call may not to return all Sidetree transactions in one batch, in which case the caller can use the transaction number of the last transaction in the returned batch to fetch subsequent transactions.

#### Request path
```
GET /transactions?since=<transaction-number>&transaction-time-hash=<transaction-time-hash>
```

#### Request headers
None.


#### Request query parameters
- `since`

  Optional. A transaction number. When not given, all Sidetree transactions since inception will be returned.
  When given, only Sidetree transactions after the specified transaction will be returned.

- `transaction-time-hash`

  Optional, but MUST BE given if `since` parameter is specified.

  This is the hash associated with the time the transaction specified by the `since` parameter is anchored on blockchain.
  Multiple transactions can have the same _transaction time_ and thus the same _transaction time hash_.

  The _transaction time hash_ helps the blockchain layer detect block reorganizations (temporary forks); `HTTP 400 Bad Request` with `invalid_transaction_number_or_time_hash` as the `code` parameter value in a JSON body is returned on such events.

#### Request example
```
GET /transactions?since=170&transaction-time-hash=00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253
```

#### Response body schema
```json
{
  "moreTransactions": "True if there are more transactions beyond the returned batch. False otherwise.",
  "transactions": [
    {
      "transactionNumber": "A monotonically increasing number (need NOT be by 1) that identifies a Sidtree transaction.",
      "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
      "transactionTimeHash": "The hash associated with the transaction time.",
      "anchorString": "The string written to the blockchain for this transaction.",
      "feePaid": "A number representing the fee paid for this transaction."
    },
    ...
  ]
}
```

#### Response example
```http
HTTP/1.1 200 OK

{
  "moreTransactions": false,
  "transactions": [
    {
      "transactionNumber": 89,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000002352597f8ec45c56ad19994808e982f5868c5ff6cfef2e",
      "anchorString": "QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf",
      "feePaid": 40000
    },
    {
      "transactionNumber": 100,
      "transactionTime": 545236,
      "transactionTimeHash": "00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253",
      "anchorString": "QmbJGU4wNti6vNMGMosXaHbeMHGu9PkAUZtVBb2s2Vyq5d",
      "feePaid": 600000
    }
  ]
}
```

#### Response example - Block reorganization detected

```http
HTTP/1.1 400 Bad Request

{
  "code": "invalid_transaction_number_or_time_hash"
}
```


### Get first valid Sidetree transaction
Given a list of Sidetree transactions, returns the first transaction in the list that is valid. Returns 404 NOT FOUND if none of the given transactions are valid. This API is primarily used by the Sidetree core library to determine a transaction that can be used as a marker in time to reprocess transactions in the event of a block reorganization (temporary fork).


#### Request path
```http
POST /transactions/firstValid HTTP/1.1
```

#### Request headers
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Request body schema
```json
{
  "transactions": [
    {
      "transactionNumber": "The transaction to be validated.",
      "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
      "transactionTimeHash": "The hash associated with the transaction time.",
      "anchorString": "The string written to the blockchain for this transaction.",
      "feePaid": "A number representing the fee paid for this transaction."
    },
    ...
  ]
}
```

#### Request example
```http
POST /transactions/firstValid HTTP/1.1
Content-Type: application/json

{
  "transactions": [
    {
      "transactionNumber": 19,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000002352597f8ec45c56ad19994808e982f5868c5ff6cfef2e",
      "anchorString": "Qm28BKV9iiM1ZNzMsi3HbDRHDPK5U2DEhKpCYhKk83UPEg",
      "feePaid": 5000
    },
    {
      "transactionNumber": 18,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000000054f9719ef6ca646e2503a9c5caac1c6ea95ffb4af587",
      "anchorString": "Qmb2wxUwvEpspKXU4QNxwYQLGS2gfsAuAE9LPcn5LprS1nb",
      "feePaid": 30
    },
    {
      "transactionNumber": 16,
      "transactionTime": 545200,
      "transactionTimeHash": "0000000000000000000f32c84291a3305ad9e5e162d8cc363420831ecd0e2800",
      "anchorString": "QmbBPdjWSdJoQGHbZDvPqHxWqqeKUdzBwMTMjJGeWyUkEzK",
      "feePaid": 50000
    },
    {
      "transactionNumber": 12,
      "transactionTime": 545003,
      "transactionTimeHash": "0000000000000000001e002080595267fe034d370897b7b506d119ad29da1541",
      "anchorString": "Qmss3gKdm9uU9YLx3MPRHQTcUq1CR1Xv9Zpdu7EBG9Pk9Y",
      "feePaid": 1000000
    },
    {
      "transactionNumber": 4,
      "transactionTime": 544939,
      "transactionTimeHash": "00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253",
      "anchorString": "QmdcDrVPWy3ZXoZcuvFq7fDVqatks22MMqPAxDqXsZzGhy"
      "feePaid": 100
    }
  ]
}
```

#### Response body schema
```json
{
  "transactionNumber": "The transaction number of the first valid transaction in the given list",
  "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
  "transactionTimeHash": "The hash associated with the transaction time.",
  "anchorString": "The string written to the blockchain for this transaction.",
  "feePaid": "A number representing the fee paid for this transaction."
}
```

#### Response example
```http
HTTP/1.1 200 OK

{
  "transactionNumber": 16,
  "transactionTime": 545200,
  "transactionTimeHash": "0000000000000000000f32c84291a3305ad9e5e162d8cc363420831ecd0e2800",
  "anchorString": "QmbBPdjWSdJoQGHbZDvPqHxWqqeKUdzBwMTMjJGeWyUkEzK",
  "feePaid": 50000
}
```

#### Response example - All transactions are invalid
```http
HTTP/1.1 404 NOT FOUND
```


### Write a Sidetree transaction
Writes a Sidetree transaction to the underlying blockchain.


#### Request path
```
POST /transactions
```

#### Request headers
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Request body schema
```json
{
  "fee": "A number representing the transaction fee to be paid to write this transaction to the blockchain.",
  "anchorString": "The string to be written to the blockchain for this transaction."
}
```

#### Request example
```http
POST /transactions HTTP/1.1

{
  "fee": 200000,
  "anchorString": "QmbJGU4wNti6vNMGMosXaHbeMHGu9PkAUZtVBb2s2Vyq5d"
}
```

#### Response body schema
None.


### Fetch normalized transaction fee for proof-of-fee calculation.
Fetches the normalized transaction fee used for proof-of-fee calculation, given the blockchain time.

Returns `HTTP 400 Bad Request` with `blockchain_time_out_of_range` as the `code` parameter value in the JSON body if the given blockchain time is:
1. earlier than the genesis Sidetree blockchain time; or
1. later than the current blockchain time.

#### Request path
```
GET /fee
```

#### Request path
```
GET /fee/<blockchain-time>
```

#### Request headers
None.

#### Request example
```
GET /fee/654321
```

#### Response body schema
```json
{
  "normalizedTransactionFee": "A number representing the normalized transaction fee used for proof-of-fee calculation."
}
```

#### Response example
```http
HTTP/1.1 200 OK

{
  "normalizedTransactionFee": 200000
}
```

#### Response example - Blockchain time given is out of computable range.

```http
HTTP/1.1 400 Bad Request

{
  "code": "blockchain_time_out_of_range"
}
```


### Fetch the current service version
Fetches the current version of the service. The service implementation defines the versioning scheme and its interpretation.

Returns the service _name_ and _version_ of the blockchain service.

#### Request path
```
GET /version
```

#### Request headers
None.

#### Request example
```
GET /version
```

#### Response body schema
```json
{
  "name": "A string representing the name of the service",
  "version": "A string representing the version of currently running service."
}
```

#### Response example
```http
HTTP/1.1 200 OK

{
  "name": "bitcoin",
  "version": "1.0.0"
}
```


## CAS REST API
The CAS (content addressable storage) REST API interface aims to abstract the underlying Sidetree storage away from the main protocol logic. This allows the CAS to be updated or even replaced if needed without affecting the core protocol logic. Conversely, the interface also allows the protocol logic to be implemented in an entirely different language while interfacing with the same CAS.

All hashes used in the API are encoded multihash as specified by the Sidetree protocol.

### Response HTTP status codes

| HTTP status code | Description                              |
| ---------------- | ---------------------------------------- |
| 200              | Everything went well.                    |
| 400              | Bad client request.                      |
| 401              | Unauthenticated or unauthorized request. |
| 404              | Resource not found.                      |
| 500              | Server error.                            |


### Read content
Read the content of a given address and return it in the response body as octet-stream.

#### Request path
```
GET /<hash>?max-size=<maximum-allowed-size>
```

#### Request query parameters
- `max-size`

  Required.

  If the content exceeds the specified maximum allowed size, `HTTP 400 Bad Request` with `content_exceeds_maximum_allowed_size` as the value for the `code` parameter in a JSON body is returned.


#### Request example
```
GET /QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf
```
#### Response headers
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/octet-stream``` |

#### Response example - Resoucre not found

```http
HTTP/1.1 404 Not Found
```

#### Response example - Content exceeds maximum allowed size

```http
HTTP/1.1 400 Bad Request

{
  "code": "content_exceeds_maximum_allowed_size"
}
```

#### Response example - Content not a file

```http
HTTP/1.1 400 Bad Request

{
  "code": "content_not_a_file"
}
```

#### Response example - Content hash is invalid

```http
HTTP/1.1 400 Bad Request

{
  "code": "content_hash_invalid"
}
```

### Write content
Write content to CAS.

#### Request path
```
POST /
```

#### Request headers
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/octet-stream``` |

#### Response headers
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Response body schema
```json
{
  "hash": "Hash of data written to CAS"
}
```

#### Response body example
```json
{
  "hash": "QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf"
}
```

### Fetch the current service version
Fetches the current version of the service. The service implementation defines the versioning scheme and its interpretation.

Returns the service _name_ and _version_ of the CAS service.

#### Request path
```
GET /version
```

#### Request headers
None.

#### Request example
```
GET /version
```

#### Response body schema
```json
{
  "name": "A string representing the name of the service",
  "version": "A string representing the version of currently running service."
}
```

#### Response example
```http
HTTP/1.1 200 OK

{
  "name": "ipfs",
  "version": "1.0.0"
}
```
