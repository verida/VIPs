---
vip: 2
title: Verida DID Method
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Stagnant
type: Standards Track
category: Core
created: 2022-11-05
---

# Overview

This VIP proposes an updated Verida DID Method implementation that leverages a blockchain smart contract to define a list of URI's that can be used to locate a DID Document for a given DID using standardized REST API calls.

This enables DID Documents to be updated or deleted without a blockchain transaction; providing significantly cheaper and faster updates.

This architecture will also enable the _right to be deleted_ at the highest possible level, where DID owners can delete their DID documents from the off-chain DID Document storage service.

This architecture is also flexible enough to be used for _any_ DID method within the same smart contract.

# Motivation

The current protocol architecture requires writing to the blockchain to create or update any DID Document. Each blockchain write has an economic cost and takes time to confirm (currently 10-15 seconds). The cost and speed of writing DID Documents to the blockchain creates a poor user experience and severely hurts adoption for new users.

The protocol currently writes to the blockchain when; creating a DID, creating a new application context (when signing into an application for the first time), updating the storage node endpoints and key rotation. This change reduces the blockchain wites to a single transaction at creation for the lifetime of the DID (unless the user decides to rotate the DID Document storage endpoints).

As the price of the underlying token blockchain primary token increases, the cost to maintain a DID will proportionally increase for every single DID Document update. This change provides significantly better long term cost predicatibility for maintaining identities on the network, which is critical for scale.

The current DID method implemented by the Verida protocol forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver) that has significant limitations in the types of DID Documents that can be stored on-chain. These limitations make sense as they are working around storage, cost and performance limitations related to storing data on the blockchain. This change enables storing any DID-Core compliant DID Document without limitation and without requiring any protocol or smart contract upgrades.

Storing the complete DID Document on-chain is permanent. There is a potential for identifying information to be placed in these critical documents. It is not possible to delete these documents from the blockchain. This change ensures a user has self-custody over their DID Document and can delete it forever, providing enhanced privacy and control over their identity.

Blockchain technology is constantly getting faster and cheaper. Separating the storage of the DID Document from the lookup of where the DID Document is stored, makes for a ___much___ lighter use of blockchain. This drastically simplifies the process of migrating to a cheaper / faster blockchain in the future. This also makes it viable to leverage DID's referenced across multiple blockchains with a future upgrade, rather than relying on a single chain.

# Specification

This proposal is a complete replacement of the [current blockcahin based DID Registry implementation](https://github.com/verida/blockchain-contracts/tree/develop/VDA-DID-Registry) that forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver).

## Smart contract

A `DID Registry` smart contract will a map a Decentralized Identifier (`DID`) to a list of URI's where the actual DID Document can be found.

Example data structure:

```
dids: {
  'did:vda:0xb794f5ea0ba39494ce839613fffba74279579268': [
    'https://au.storage.verida.io/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268',
    'https://au-22.storage.alchemy.io/did/0xb794f5ea0ba39494ce839613fffba74279579268',
    'https://au-3.storage.figment.io/dids/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268'
  ],
  'did:vda:0xc894f5ea0ba39494ce839613fffba74279579379': [...],
  ...
}
```

As with most DID Method implementations, anyone is free to create a DID and register it with the `DID Registry` smart contract.

>Question: Support flagging a `DID` as `deleted` on-chain?

The smart contract will support the following methods:

1. `register(did: string, endpoints: string[], signature: string)` &mdash; Register a list of endpoints for a `did` where the DID Document can be located. Updates the endpoints if the `did` is already registered. Increments the `nonce` for the `did`.
2. `lookup(did: string): [nonce: number, endpoints: string[]]` &mdash; Lookup the endpoints for a given `did`. Returns the current `nonce` and the array of endpoints.
3. `nonce(did: string): number` &mdash; Obtain the next `nonce` required to update a list of `did` endpoints.

The `register()` method will verify the `signature` is a signature generated from a`proofString`, where:

```
proofString = sign(`${did}/${nonce}/${endpoints[0}/${endpoints[1}/...}`, didPrivateKey)
```

## DID Document

### Create and Update

DID Documents are stored as [JSON](https://www.w3.org/TR/did-core/#json) or [JSON-LD](https://www.w3.org/TR/did-core/#json-ld) representations. It's recommended to use the [did-document (or custom fork)](https://www.npmjs.com/package/did-document) to generate the DID Documents.

The DID Document ___MUST___ include the following properties:

- [versionId](https://www.w3.org/TR/did-spec-registries/#versionid)
- [created](https://www.w3.org/TR/did-spec-registries/#created)
- [updated](https://www.w3.org/TR/did-spec-registries/#updated)
- [deactivated](https://www.w3.org/TR/did-spec-registries/#deactivated)
- `proof` &mdash; A string representing the full DID Document as a JSON encoded string that has been hashed using keccak256 and signed with ed25519, the default Ethereum based signature scheme.

The implementation on how to create the document is out of scope. ie: For a HTTP endpoint a HTTP server will be required and provide a way to create DID Documents on the server, likely stored on disk or in a database.

An endpoint should make every effort to verify the DID Document is valid before creation:

1. Ensure there is currently no entry for the given `DID` in the `DID Registry` _OR_ there is currently an entry and it references this storage node endpoint
2. The DID Document is a valid DID Document using the `did-document` npm package
3. The DID Document has valid properties (ie: valid `proof`, `versionId` etc.)

Endpionts ___MUST___ store all known versions of a DID Document.

### Deletion

It ___MUST___ be possible for a DID Controller to delete their DID Document from an endpoint.

It is up to each endpoint to determine how a DID cCntroller can delete a DID Document.

## DID Registration

The DID will be registered on the blockchain via the `DID Registry` smart contract.

Registering a DID requires:

1. A `nonce`. When creating a DID for the first time `nonce=0`, whereas subsequent updates must use the next `nonce` obtained by incrementing the value returned from calling `nonce(did: string)` on the smart contract.
2. A valid `signature` generated as per the [Smart Contract](#smart-contract)

A new DID is registered by calling `register(did: string, endpoints: string[], signature: string)` on the `DID Registry` smart contract

### Endpoint Discovery

The DID Document is stored at one or more URI endpoints.

These endpoints can be obtained by calling `lookup(did:string)` on the `DID Registry` smart contract.

### Retrieval

DID Documents can be retrieved in a universal way from any valid endpoint.

Endpoints that are storing DID Documents must support the following optional query parameters:

1. [versionId](https://www.w3.org/TR/did-spec-registries/#versionId-param) &mdash; Returns the DID Document that matches the requested `versionId`.
2. [versionTime](https://www.w3.org/TR/did-spec-registries/#versionTime-param) &mdash; Returns the DID Document that was valid at the given timestamp.

The latest DID Document version is returned if no query parameters are provided.

Example requests:

1. `https://au.storage.verida.io/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268` &mdash; Resolve latest DID Document from `au.storage.verida.io`
2. `https://au-22.storage.alchemy.io/did/0xb794f5ea0ba39494ce839613fffba74279579268?versionTime` &mdash; Resolve the DID Document version from `au-22. .storage.alchemy.io` that was valid at `2016-10-17T02:41:00Z`.

### Resolving

A DID Document is resolved by:

1. Completing a [DID Document Lookup](#did-document-lookup) to locate the list of service endpoints storing the DID Document
2. [Retreiving](#did-document-retrieval) of the DID Document from each of the valid endpoints
3. Reaching consensus on the correct DID Document
4. Validating the DID Document
5. Returning the DID Document

#### Consensus

The retreived DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining > 50% consensus on the correct version will be executed. If consensus can not be reached the DID Document fails to resolve.

#### Verification

The resolved DID Document will then be verified as follows:

1. The DID Document `versionId` returned from the storage node matches the `nonce` returned from `lookup()`. This ensures the latest DID Document was returned (assuming the client is seeking the latest).
2. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.

### Versioning

DID Documents ___MUST___ be versioned.

This specification ensures all DID Documents have a `versionId` parameter, that all endpoints store all versions and that DID Documents can be queried by `versionId`.

Additionally, the ability to query a DID Document by `timestamp`, ensures the correct DID Document version at a particular time can be resolved. This is critical for key rotation as previously signed data may be using a public key no longer in the latest DID Document.

### Key rotation

It is possible to [rotate _verificationMethod_ keys](https://www.w3.org/TR/did-core/#verification-method-rotation) by writing an updated DID Document with new public keys.

it is possible to [rotate the DID ocontroller](https://www.w3.org/TR/did-core/#changing-the-did-controller) by changing the [controller` property of the DID Document](https://www.w3.org/TR/did-core/#did-controller).

### Verification method revocation

It is possible to [revoke a verificationMethod](https://www.w3.org/TR/did-core/#verification-method-revocation) by writing an updated DID Document with the previous `verificationMethod` removed.

# Backwards compatibility

This will not be backwards compatible with the existing centralized Verida DID Registry. Users will be required to create new DIDs and previously created DIDs will no longer resolve.

# Security considerations

An endpoint operator may be malicious and alter the DID Document, but that is protected via the `nonce`, `versionId` and `proof` verifications below. DID controllers can store their DID Document across multiple storage nodes with different endpoint operators. This ensures one malicious operator can be identified (and their response discarded) assuming > 50% of endpoints are trusted.

There is a possibility of replay attacks, where someone attempts to submit an old (previously submitted) list of endpoints to the `DID Registry`. The `nonce` avoids these types of attacks.

There is a possibility of a malicious endpoint operator returning an old DID Document version. Ensuring the `nonce` and the `versionId` match ensures consistency between the blockchain's record of truth of the current version and the version returned by the storage node.

There is a possiblity of a malicious endpoint operator generating a vulnerable DID Document that replaces a valid DID Document. The `proof` embedded in the DID Document avoids any third party tampering or generating an invalid DID Document.

# Redundancy

There is a possibility of an endpoint becoming unavailable (either temporarily or permanently). It is expected taht DID Documents will be stored across any number of storage nodes. This ensures redundancy in the event one or more storage nodes become unavailable.

Ideally DID Documents will be stored across a minimum of 3 endpoints.

# Blockchain

The smart contract will be deployed to the Polygon PoS network due to it's low confirmation times, low cost and widespread adoption.

# Compliance

@todo Expand this to cover address the _Must_ have documentation requirements from https://www.w3.org/TR/did-core/#methods
https://www.w3.org/TR/did-core/#method-operations

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
