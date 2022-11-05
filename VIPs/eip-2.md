---
eip: 2
title: Store DID Documents off-chain
author: Chris Were <chris@verida.io>
discussions-to: 
status: Stagnant
type: Standards Track
category: Core
created: 2022-11-05
---

## Simple Summary

This VIP changes the Verida DID Method implementation to store DID Documents within the Verida network instead of on the blockchain. Instead, Decentralized Identifiers (DIDs) will be written to the blockchain only once, with a pointer to the Verida URI where the DID document can be found. This will mean any DID Document update can happen without a blockchain transaction; providing cheaper and faster updates. This architecture will also enable the "right to be deleted" at the highest possible level, where DID owners can delete their DID documents from the off-chain Verida network storage.

## Motivation

The current protocol architecture requires writing to the blockchain to create or update any DID Document. Each blockchain write has an economic cost (currently charged in $MATIC) and takes time to confirm (currently 10-15 seconds). The cost and speed of writing DID Documents to the blockchain creates a poor user experience and severely hurts adoption for new users.

The protocol currently writes to the blockchain when; creating a DID, creating a new application context (when signing into an application for the first time), updating the storage node endpoints and any key rotation. This changes reduces this to a single blockchain write at creation for the lifetime of the DID.

As the price of the underlying token ($MATIC) increases, the cost to maintain a DID will proportionally increase for every single DID Document update. This change provides significantly better long term cost predicatibility for maintaining identities on the network, which is critical for scale.

The current DID method implemented by the Verida protocol forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver) that has significant limitations in the extent it supports the DID Document specification. This change enables storing any DID-Core compliant DID Document without any protocol or blockchain upgrades.

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

>Question: Support flagging a `DID` as `deleted`?

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

The DID Document will contain a `proof` that is a string representing the full DID Document that has been hashed using keccak256 and signed with ed25519.

#### Storage

New DID Documents will be submitted to a new storage node endpoint `createDIDDocument()`. This functionality requires a custom, privileged, endpoint enabling a new identity to store their DID document before it's publicly registered in the blockcahin `DID Registry` contract.

The method signature is:

```
createDIDDocument(didDocument) {}
```

The storage node will perform the following verification checks:

1. There is currently no entry for the given `DID` in the `DID Registry`
2. The DID Document is a valid DID Document using the `did-document` npm package
3. The DID Document has a valid `proof` signature

Any failed verification checks will return a `401 Unauthorized` HTTP response.

Once all the above checks have successfully completed, the storage node will:

1. Create a new database based on the hash of the `DID`, context name `Verida: DID Registry` and database name `did-document`
2. Set the database permissions to be owner write, public read
3. Save the DID document to the newly created database with a document `_id` equal to the `DID` (ie: `did:vda:0x...`)

Upon success the storage node will return a `200 OK` HTTP response.

This process will be repeated for as many storage nodes as desired by the `DID` owner. This enables redundancy and availability of the DID Document across the Verida network.

#### Registering

The `DID` will be registered on the blockchain via the `DID Registry` smart contract.

The Verida SDK will set the `nonce` to `0` for a new `DID` or fetch the latest `nonce` for an update. It will generate a valid signature, signed by the `DID` private key.

The Verida SDK will support calling `register(did: string, endpoints: string[], signature: string)` on the `DID Registry` smart contract

#### Lookup

The Verida SDK will provide a Verida DID Resolver package (`vda-did-resolver`) that generates a standard `getResolver()` function that implements a `resolver(did: string)` method. This ensures the Verida DID method will be compatible with existing DID resolver libraries.

The `resolver(did: string)` method will call the `lookup()` method on the `DID Registry` smart contract to determine the service endpoints where the DID Document can be found.

One endpoint will be chosen and the SDK will call a method `getDidDocument(did: string)`. This method will open the public, read only database (`did-document`) for the application context (`Verdida: DID Registry`) and return the latest DID Document where `_id` matches the `did`.

>_Note: It is possible for the user to open the database remotely without calling this method. This method will return the DID Document without requiring the client side to make a direct connection to the CouchDB database, significantly requiing the dependencies required and increasing performance._

The found DID document will then be verified as follows:

1. The DID Document `versionId` returned from the storage node matches the `nonce` returned from `lookup()`. This ensures the latest DID Document was returned
2. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.


## Backwards compatibility

This will not be backwards compatible with the existing centralized DID Registry. Users will be required to create new `DID`s.

## Security considerations

There is a possibility of replay attacks, where someone attempts to submit a `DID` update to the `DID Registry` using an older copy of the DID Document. The `nonce` avoids these types of attacks.

There is a possibility of a malicious storage node operator returning an old DID Document version. Ensuring the `nonce` and the `versionId` match ensures consistency between the blockchain's record of truth of the current version and the version returned by the storage node.

There is a possiblity of a malicious storage node operator generating a vulnerable DID Document that replaces a valid DID Document. The `proof` embedded in the DID Document avoids any third party tampering or generating an invalid DID Document.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
