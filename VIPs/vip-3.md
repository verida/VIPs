---
vip: 3
title: DID Document Storage on the Verida Network
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Stagnant
type: Standards Track
category: Core
created: 2022-11-06
---

# Overview

[VIP-2](./vip-2) defines a Verida DID Method that requires verified storage and retreival of DID Documents. This VIP proposes updates to the [Verida Storage Node](https://github.com/verida/storage-node) that supports these requirements.

# Specification

## Storage

DID Documents are stored in the same way as other data on the Verida network. The documents are stored in databases owned and controlled by the DID controller. This ensuresthe DID controller to be responsible for any document updates or deletion, ensuring self-sovereignty.

The Verida protocol requires looking up the `DID` in the `DID Registry` before authorizing access to a database. When creating a DID Document for the first time, this entry does not yet exist on-chain, so the usual protocol authorization can not occur.

As such, it's necessary to offer a `create` endpoint to manually authenticate the DID Document and store it on behalf of the DID.

In order to properly support the [VIP-2](./vip-2) specification, increase performance and simplify the implementation of the Verida DID Resolver, additional endpoints relating to DID Document management are also included in this specification.

>Note: Once a DID Document is created (both on-chain and resolvable at a valid endpoint) the DID controller can use the Verida SDK to authenticate and manage their DID Document just like any other database on the Verida network.

## Endpoints

The following new endpoints will be supported:

- `POST /did/${did}` &mdash; Create a new DID Document
- `PUT /did/${did}` &mdash; Update an existing DID Document
- `DELETE /did/${did}` &mdash; Delete an existing DID Document
- `GET /did/${did}` &mdash; Get an existing DID Document
- `POST /did/migrate/${did}` &mdash; Migrate an existing DID Document to this storage node

### Create

DID Documents will be created via a new storage node endpoint `POST /did/${did}`.

Example request:

```
POST /did/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268
{
  document: 'DID Document as JSON'
  signature: 'valid sig'
}
```

Empty DID Documents will be created in memory via the Verida SDK using the existing `@verida/did-document` package.

The [required DID Document properties](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#create-and-update will be set in the DID Document.

The storage node will perform the following verification checks:

1. There is currently no entry for the given `DID` in the `DID Registry` _OR_ there is currently an entry and it references this storage node endpoint
2. The DID Document is a valid DID Document using the `did-document` npm package
3. The DID Document has a valid `proof` signature

Any failed verification checks will return a `401 Unauthorized` HTTP response.

Once all the above checks have successfully completed, the storage node will save the DID Document.

For creating DID Documents:

1. Create a new database based on the hash of the `DID`, context name `Verida: DID Registry` and database name `did-document`
2. Set the database permissions to be owner write, public read
3. Save the DID document to the `did-document` database with a document `_id` equal to the `DID` (ie: `did:vda:0x...`)

Upon success the storage node will return a `200 OK` HTTP response.

The Verida Client SDK will repeat this process for all the DID endpoints specified on-chain for the DID. This enables redundancy and availability of the DID Document across the Verida network.

### Update

The DID Document will be deleted via a new storage node endpoint `PUT /did/${did}`.

1. Save the DID document to the `did-document` database with a document `_id` equal to the `DID` (ie: `did:vda:0x...`)
2. Set a `timestamp` property equal to the current unix epoch timestamp (to support timestamp based version retreival necessary for key revocation)

### Delete

The DID Document will be deleted via a new storage node endpoint `DELETE /did/${did}`.

This will permanently delete all record of this DID on the storage node, including previous versions.

Example request:

```
DELETE /did/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268
{
  signature: 'valid sig'
}
```

Where:

- `signature` is a signature, signed by the DID controller private key, of the string `Delete DID Document ${did} at ${timestamp/60}`

The storage node will verify:

- `signature` is valid for any timestamp in the previous or future 60 seconds

Th storage node will then delete the `did-document` database.

### Migrate

The DID Document will be deleted via a new storage node endpoint `POST /did/migrate/${did}`.

DID Documents can be migrated between storage nodes and retain all their version history. This functionality requires custom, privileged, endpoint for creating and updating stored copies of the DID Document.

Enables an existing DID Document, along with all previous versions, to be migrated to a new storage node. This is necessary if a DID controller whiches to add a new storage node replica to the network after DID creation. This may be to increase the number of copies for increased redundancy or to replace an existing storage node.

Example request:

```
{
  didDocuments: [
    documentVersion0,
    documentVersion1,
    documentVersion2,
  ],
  signature: '<valid sig>'
}
```

Where:

1. `didDocuments` is a version ordered (0 first) array of all previous versions of the DID Document.
2. `signature` is a signature, signed by the DID controller private key, of the string represented by `JSON.encode(didDocuments)`

The storage node will verify:

1. `signature` is valid
2. Each `didDocument` is valid, following the same criteria used when updating a DID Document.

The storage node will then create the `did-document` database and store all the `didDocuments` in the same way as when creating a DID Document.

### Get

The DID Document can be resolved via a new storage node endpoint `GET /did/${did}`.

@todo Implement [retrieval parameters support](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#retrieval

# VDA DID Resolver

@todo (move this to a separate VIP)

The Verida SDK will provide a Verida DID Resolver package (`vda-did-resolver`) that generates a standard `getResolver()` function that implements a `resolve(did: string, timestamp?: number)` method. This ensures the Verida DID method will be compatible with existing DID resolver libraries.

The `resolve()` method will call the `lookup()` method on the `DID Registry` smart contract to determine the service endpoints where the DID Document can be found.

The SDK will call a method `getDidDocument(did: string, timestamp? number)` on each of the endpoints. This method will open the public, read only database (`did-document`) for the application context (`Verdida: DID Registry`) and return the latest DID Document where `_id` matches the `did`. The optional _timestamp_ parameter is used to select the version of the DID document that was correct at the given timestamp. If _timestamp_ is not provided, the latest DID Document is returned.

>_Note: It is possible for the user to open the database remotely without calling this method. This method will return the DID Document without requiring the client side to make a direct connection to the CouchDB database, significantly requiing the dependencies required and increasing performance._

The retreived DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining >50% consensus on the correct version will be executed. If consensus can not be reachd the SDK will throw a `Invalid DID Document` Error.

The resolved DID document will then be verified as follows:

1. The DID Document `versionId` returned from the storage node matches the `nonce` returned from `lookup()`. This ensures the latest DID Document was returned
2. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.

# Security considerations

The DID Document is stored in a database with public read, owner write permissions. This ensures only the DID Document controller can create, update or delete the DID Document via the Verida protocol.

https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#

The [security considerations of the Verida DID Method](./vip-2.md#security-considerations) also apply.

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
