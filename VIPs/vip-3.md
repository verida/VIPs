---
vip: 3
title: DID Document Storage on the Verida Network
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-11-06
---

# Overview

[VIP-2](./vip-2) defines a Verida DID Method that requires verified storage and retreival of DID Documents. This VIP proposes updates to the [Verida Storage Node](https://github.com/verida/storage-node) that supports the storage node supporting storage of DID Documents.

This requires adding the appropriate REST API endpionts required by the Verida DID Method and other useful REST API methods that can be used by the Verida SDK to make it easy to write and update DID Documents across the network.

# Specification

## Storage

DID Documents are stored in an admin only database (`verida_dids`) with all operations to modify the DID Documents accessible via custom set of `/did/` REST API endpoints.

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
- `POST /did/${did}/migrate` &mdash; Migrate an existing DID Document to this storage node

### Create

DID Documents will be created via a new storage node endpoint `POST /did/${did}`.

Example request:

```
POST /did/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268
{
  document: 'DID Document as JSON'
}
```

It's expected the client side will create empty DID Documents via an upgrade to the Verida SDK using the existing `@verida/did-document` package.

The storage node will perform the following verification checks:

1. The [required DID Document properties](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#create-and-update) have been set on the DID Document
2. There is currently no entry for the given `DID` in the `DID Registry` _OR_ there is currently an entry and it references this storage node endpoint
3. The DID Document is a valid DID Document using the `did-document` npm package
4. The DID Document has a valid `proof` signature

Any failed verification checks will return a `400 Bad Request` HTTP response, except invalid `proof` signature which will return `401 Unauthorized`.

Once all the above checks have successfully completed, the storage node will save the DID Document.

For creating DID Documents:

1. Create a new database based on the hash of the `DID`, context name `Verida: DID Registry` and database name `did-document`
2. Set the database permissions to be owner write, public read
3. Save the DID document to the `did-document` database with a document `_id` equal to the `DID` (ie: `did:vda:0x...`)

Upon success the storage node will return a `200 OK` HTTP response.

The Verida Client SDK will repeat this process for all the DID endpoints specified on-chain for the DID. This enables redundancy and availability of the DID Document across the Verida network.

### Update

The DID Document will be updated via a new storage node endpoint `PUT /did/${did}`.

Example request:

```
PUT /did/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268
{
  document: 'DID Document as JSON'
}
```

The storage node will perform the following verification checks:

1. The [required DID Document properties](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#create-and-update) have been set on the DID Document
2. The DID Document already exists in this storage node
3. The DID Document is a valid DID Document using the `did-document` npm package
4. The DID Document has a valid `proof` signature
5. The `versionId` of the new DID Document is an increment on the currently stored DID Document `versionId`

Any failed verification checks will return a `401 Unauthorized` HTTP response.

Once all the above checks have successfully completed, the existing DID Document will be retreived where `_id` matches the `did` in the HTTP request.

The DID Document will be saved with the new document, creating a new version stored in the database for that `_id`.

Upon success the storage node will return a `200 OK` HTTP response.

The Verida Client SDK will repeat this process for all the DID endpoints specified on-chain for the DID. This enables redundancy and availability of the DID Document across the Verida network.

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

A DID Document can be migrated to storage node via a new storage node endpoint `POST /did/migrate/${did}`.

DID Documents can be migrated between storage nodes and retain all their version history. This is necessary if a DID controller adds a new storage node replica to the network after DID creation. A DID controller may do this to; increase the number of copies for increased redundancy or replace an existing storage node.

Example request:

```
{
  versions: [
    documentVersion0,
    documentVersion1,
    documentVersion2,
  ],
  signature: '<valid sig>'
}
```

Where:

1. `version` is a version ordered (0 first) array of all previous versions of the DID Document.
2. `signature` is a signature, signed by the DID controller private key, of the string represented by `JSON.encode(didDocuments)`

The storage node will verify:

1. `signature` is valid
2. Each `didDocument` is valid, following the same criteria used when updating a DID Document.
3. The last `didDocument` has a `versionId` that matches the on-chain `nonce` for the DID.

The storage node will then create the `did-document` database and store all the `didDocuments` in the same way as when creating a DID Document.

### Get

The DID Document can be resolved via a new storage node endpoint `GET /did/${did}`.

This endpoint will support the [full range of query parameters specified in the Verida DID Method](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#retrieval) to locate the correct DID Document version to return.

# Security considerations

The DID Document is stored in a database with public read, owner write permissions. This ensures only the DID Document controller can create, update or delete the DID Document via the Verida protocol.

The [security considerations of the Verida DID Method](./vip-2.md#security-considerations) also apply.

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
