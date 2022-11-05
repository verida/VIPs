---
eip: 2
title: Store DID Documents off-chain
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Stagnant
type: Standards Track
category: Core
created: 2022-11-05
---

## Overview

This VIP changes the Verida DID Method implementation to store DID Documents within the Verida network instead of on the blockchain. Instead, Decentralized Identifiers (DIDs) will be written to the blockchain once, with a pointer to the Verida network storage node(s) where the DID document can be found. This enables DID Document updates to occur without a blockchain transaction; providing significantly cheaper and faster updates.

This architecture will also enable the _right to be deleted_ at the highest possible level, where DID owners can delete their DID documents from the off-chain Verida network storage.

## Motivation

The current protocol architecture requires writing to the blockchain to create or update any DID Document. Each blockchain write has an economic cost (currently charged in $MATIC) and takes time to confirm (currently 10-15 seconds). The cost and speed of writing DID Documents to the blockchain creates a poor user experience and severely hurts adoption for new users.

The protocol currently writes to the blockchain when; creating a DID, creating a new application context (when signing into an application for the first time), updating the storage node endpoints and any key rotation. This changes reduces this to a single blockchain write at creation for the lifetime of the DID.

As the price of the underlying token ($MATIC) increases, the cost to maintain a DID will proportionally increase for every single DID Document update. This change provides significantly better long term cost predicatibility for maintaining identities on the network, which is critical for scale.

The current DID method implemented by the Verida protocol forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver) that has significant limitations in the types of DID Documents that can be stored on-chain. These limitations make sense as they are working around storage, cost and performance limitations related to storing data on the blockchain. This change enables storing any DID-Core compliant DID Document without limitation and without requiring any protocol or smart contract upgrades.

Storing the complete DID Document on-chain is permanent. There is a potential for identifying information to be placed in these critical documents. It is not possible to delete these documents from the blockchain. This change ensures a user has self-custody over their DID Document and can delete it forever, providing enhanced privacy and control over their identity.

## Specification

This proposal is a complete replacement of the [current blockcahin based DID Registry implementation](https://github.com/verida/blockchain-contracts/tree/develop/VDA-DID-Registry) that forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver).

### Smart contract

A `DID Registry` smart contract will a map a Decentralized Identifier (`DID`) to a list of Verida compatible storage nodes (`DID Storage Nodes`) that store the actual DID Document

Example data structure:

```
dids: {
  'did:vda:0xb794f5ea0ba39494ce839613fffba74279579268': [
    'https://au.storage.verida.io',
    'https://au-22.storage.alchemy.io',
    'https://au-3.storage.figment.io'
  ],
  'did:vda:0xc894f5ea0ba39494ce839613fffba74279579379': [...],
  ...
}
```

As with most DID Method implementations, anyone is free to create a DID and register it with the `DID Registry` smart contract.

>Question: Support flagging a `DID` as `deleted` on-chain?

The smart contract will support the following methods:

1. `register(did: string, endpoints: string[], signature: string)` &mdash; Register a list of endpoints for a `did` where the DID Document can be located. Updates the endpoints if the `did` is already registered. Increments the `nonce` for the `did`.
2. `lookup(did: string): [nonce: number, endpoints: string[]` &mdash; Lookup the endpoints for a given `did`. Returns the current `nonce` value (representing the latest version number) and the array of endpoints.
3. `nonce(did: string): number` &mdash; Obtain the next `nonce` required to update a list of `did` endpoints.

The `register()` method will verify the `signature` is a signature generated from a`proofString`, where:

```
proofString = sign(`${did}/${nonce}/${endpoints[0}/${endpoints[1}/...}`, didPrivateKey)
```

### DID Document

#### Creation

Empty DID Documents will be created in memory via the Verida SDK using the existing `@verida/did-document` package.

The DID Document will contain a `proof` that is a string representing the full DID Document that has been hashed using keccak256 and signed with ed25519, the default Ethereum based signature scheme.

#### Storage

DID Documents will be submitted to a new storage node endpoint `writeDIDDocument()`. DID Documents can be migrated between storage nodes and retain all their version history. This functionality requires custom, privileged, endpoint for creating and updating stored copies of the DID Document.

##### writeDIDDocument()

Creating a specific `writeDIDDocument()` endpoint enables a new identity to store their DID Document before it's publicly registered in the blockcahin `DID Registry` contract. The Verida protocol requires looking up the `DID` in the `DID Registry` before authorizing access to a database. When creating a DID Document, this entry does not yet exist on-chain, so the usual protocol authorization can not occur. As such, it's necessary to create this custom endpoint to manually authenticate the DID Document and store it on behalf of the DID.

The method signature is:

```
writeDIDDocument(didDocument: string) {}
```

The storage node will perform the following verification checks:

1. There is currently no entry for the given `DID` in the `DID Registry` _OR_ there is currently an entry and it references this storage node endpoint
2. The DID Document is a valid DID Document using the `did-document` npm package
3. The DID Document has a valid `proof` signature

Any failed verification checks will return a `401 Unauthorized` HTTP response.

Once all the above checks have successfully completed, the storage node will save the DID Document.

For creating DID Documents:

1. Create a new database based on the hash of the `DID`, context name `Verida: DID Registry` and database name `did-document`
2. Set the database permissions to be owner write, public read

For creating and updating DID Documents:

1. Save the DID document to the `did-document` database with a document `_id` equal to the `DID` (ie: `did:vda:0x...`)
2. Set a `timestamp` property equal to the current unix epoch timestamp (to support timestamp based version retreival necessary for key revocation)

Upon success the storage node will return a `200 OK` HTTP response.

This process will be repeated for as many storage nodes as desired by the `DID` owner. This enables redundancy and availability of the DID Document across the Verida network.

##### migrateDIDDocument()

Creating a specific `migrateDIDDocument()` endpoint enables an existing DID Document, along with all previous versions, to be migrated to a new storage node. This is necessary if a DID controller whiches to add a new storage node replica to the network after DID creation. This may be to increase the number of copies for increased redundancy or to replace an existing storage node.

The method signature is:

```
writeDIDDocument(didDocuments: string[], signature) {}
```

Where:

- `didDocuments` is an array of all previous, timestamped versions of the DID Document.
- `signature` is a signature, signed by the DID controller private key, of the string represented by `JSON.encode(didDocuments)`

The storage node will verify:

- `signature` is valid
- Each `didDocument` is valid, following the same criteria as `writeDIDDocument()`

The storage node will then create the `did-document` database and store all the `didDocuments` in the same manner as `writeDIDDocument()`

##### deleteDIDDocument()

@todo

#### Registering

The `DID` will be registered on the blockchain via the `DID Registry` smart contract.

The Verida SDK will set the `nonce` to `0` for a new `DID` or fetch the latest `nonce` for an update. It will generate a valid signature, signed by the `DID` private key.

The Verida SDK will support calling `register(did: string, endpoints: string[], signature: string)` on the `DID Registry` smart contract

#### Lookup

The Verida SDK will provide a Verida DID Resolver package (`vda-did-resolver`) that generates a standard `getResolver()` function that implements a `resolve(did: string, timestamp?: number)` method. This ensures the Verida DID method will be compatible with existing DID resolver libraries.

The `resolve()` method will call the `lookup()` method on the `DID Registry` smart contract to determine the service endpoints where the DID Document can be found.

The SDK will call a method `getDidDocument(did: string, timestamp? number)` on each of the endpoints. This method will open the public, read only database (`did-document`) for the application context (`Verdida: DID Registry`) and return the latest DID Document where `_id` matches the `did`. The optional _timestamp_ parameter is used to select the version of the DID document that was correct at the given timestamp. If _timestamp_ is not provided, the latest DID Document is returned.

>_Note: It is possible for the user to open the database remotely without calling this method. This method will return the DID Document without requiring the client side to make a direct connection to the CouchDB database, significantly requiing the dependencies required and increasing performance._

The retreived DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining >50% consensus on the correct version will be executed. If consensus can not be reachd the SDK will throw a `Invalid DID Document` Error.

The resolved DID document will then be verified as follows:

1. The DID Document `versionId` returned from the storage node matches the `nonce` returned from `lookup()`. This ensures the latest DID Document was returned
2. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.

#### Versioning

#### Key rotation

It is possible to rotate _verificationMethod_ keys by writing an updated DID Document via 

https://www.w3.org/TR/did-core/#verification-method-rotation



## Backwards compatibility

This will not be backwards compatible with the existing centralized DID Registry. Users will be required to create new `DID`s.

## Security considerations

The DID Document is stored in a database with public read, owner write permissions. This ensures only the DID Document controller can create, update or delete the DID Document via the Verida protocol.

The storage node operator may be malicious and alter the DID Document, but that is protected via the `nonce`, `versionId` and `proof` verifications below. DID controllers can store their DID Document across multiple storage nodes with different operators. This ensures one malicious operator can be identified (and their response discarded) assuming > 50% of endpoints are trusted.

There is a possibility of replay attacks, where someone attempts to submit an old (previously submitted) list of endpoints to the `DID Registry`. The `nonce` avoids these types of attacks.

There is a possibility of a malicious storage node operator returning an old DID Document version. Ensuring the `nonce` and the `versionId` match ensures consistency between the blockchain's record of truth of the current version and the version returned by the storage node.

There is a possiblity of a malicious storage node operator generating a vulnerable DID Document that replaces a valid DID Document. The `proof` embedded in the DID Document avoids any third party tampering or generating an invalid DID Document.

## Redundancy

There is a possibility of a storage node becoming unavailable (either temporarily or permanently). DID Documents can be stored across an unlimimited number of storage nodes. This ensures redundancy in the event one or more storage nodes become unavailable.

## Compliance

@todo

At a bare minimum, address the _Must_ have documentation requirements from https://www.w3.org/TR/did-core/#methods
https://www.w3.org/TR/did-core/#method-operations

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
