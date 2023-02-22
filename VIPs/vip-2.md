---
vip: 2
title: Verida DID Method
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Review
type: Standards Track
category: Core
created: 2022-11-05
---

# Overview

This VIP proposes an updated Verida DID Method implementation that leverages a blockchain smart contract to define a list of URI's that can be used to locate a DID Document for a given DID using standardized HTTP REST API calls.

This enables DID Documents to be updated or deleted without a blockchain transaction; providing significantly cheaper and faster updates.

This architecture will also enable the _right to be deleted_ at the highest possible level, where DID controllers can delete their DID documents from the off-chain DID Document storage service.

This architecture is similar to DNS, where a small list of addresses provide authority over an unlimited number of endpoints that maintain the actual resolution metadata. With DNS, this metadata is DNS records, in this instance the metadata is the DID Document.

This approach is also flexible enough to support _multiple_ DID methods, using the same approach, within the same smart contract.

# Motivation

The current protocol architecture requires writing to the blockchain to create or update any DID Document. Each blockchain write has an economic cost and takes time to confirm (currently 10-15 seconds). The cost and speed of writing DID Documents to the blockchain creates a poor user experience and severely hurts adoption for new users.

The protocol currently writes to the blockchain when; creating a DID, creating a new application context (when signing into an application for the first time), updating the storage node endpoints and key rotation. This change reduces the blockchain wites to a single transaction at creation for the lifetime of the DID (unless the user decides to rotate the DID Document storage endpoints).

As the price of the underlying token blockchain primary token increases, the cost to maintain a DID will proportionally increase for every single DID Document update. This change provides significantly better long term cost predicatibility for maintaining identities on the network, which is critical for scale.

The current DID method implemented by the Verida protocol forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver) that has significant limitations in the types of DID Documents that can be stored on-chain. These limitations make sense as they are working around storage, cost and performance limitations related to storing data on the blockchain. This change enables storing any DID-Core compliant DID Document without limitation and without requiring any protocol or smart contract upgrades.

Storing the complete DID Document on-chain is permanent. There is a potential for identifying information to be placed in these critical documents. It is not possible to delete these documents from the blockchain. This change ensures a user has self-custody over their DID Document and can delete it forever, providing enhanced privacy and control over their identity.

Blockchain technology is constantly getting faster and cheaper. Separating the storage of the DID Document from the lookup of where the DID Document is stored, makes for a ___much___ lighter use of blockchain. This drastically simplifies the process of migrating to a cheaper / faster blockchain in the future. This also makes it viable to leverage DID's referenced across multiple blockchains with a future upgrade, rather than relying on a single chain.

# Specification

This proposal is a complete replacement of the blockcahin based Verida DID Registry implementation that forks [ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver).

## Verida DID method (did:vda)

### DID Method Name

The method-name for the Verida DID Method will be identified by the string `vda`.
A DID that uses the vda DID method MUST begin with the prefix `did:vda`. This prefix string MUST be in lowercase. The remainder of the DID, after the prefix, is as follows:

### Method Identifier

The Verida DID method's identifier is made up of the namespace component. The namespace is defined as a string that identifies the Verida network (e.g., `mainnet`, `testnet`) where the DID reference is stored.

### Unique Identifier

A did:vda DID must be a unique public key hexadecimal string as per the [did-ethr spec](https://github.com/decentralized-identity/ethr-did-resolver/blob/master/doc/did-method-spec.md#method-specific-identifier), namely it must be represented in compressed form (see https://en.bitcoin.it/wiki/Secp256k1).

The genesis (version 0) record in a DID document must have a controller matching this unique identifier.

### Blockchain v DIDs

A Verida DID unique identifier (and correspondinb private key) is generated from an Ethereum public / private key pair. However, this specification clearly separates a DID identifier keypair from a blockchain keypair.

This is similar to how the [cheqd DID method separates the keyparis from Cosmos layer and the identity layer](https://docs.cheqd.io/identity/architecture/adr-list/adr-001-cheqd-did-method#privacy-considerations).

As a result, the blockchain address that creates or updates a DID reference in the DID Registry smart contract will not necessarily be the same as the DID controller.

### Example identifiers

```
did:vda:testnet:0xb9c5714089478a327f09197987f16f9e5d936e8a
did:vda:mainnet:0xb9c5714089478a327f09197987f16f9e5d936e8a
```

### Multiple blockchain registries

In the future, this specification may be upgraded to support multiple blockchain registries. Such an upgraded would likely support [CAIP Blockchain IDs](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) being an optional inclusion in the DID:

```
did:vda:testnet:eip55:1:0xb9c5714089478a327f09197987f16f9e5d936e8a
did:vda:mainnet:starknet:SN_GOERLI:0xb9c5714089478a327f09197987f16f9e5d936e8a
```

## Smart contract

A `DID Registry` smart contract will map a Decentralized Identifier (`DID`) to a list of URI's where the actual DID Document can be found.

Example data structure:

```
dids: {
  '0xb794f5ea0ba39494ce839613fffba74279579268': [
    'https://au.storage.verida.io/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268',
    'https://au-22.storage.alchemy.io/did/0xb794f5ea0ba39494ce839613fffba74279579268',
    'https://au-3.storage.figment.io/dids/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268'
  ],
  '0xc894f5ea0ba39494ce839613fffba74279579379': [...],
  ...
}
```

Note: For performance reasons, just the `DID Address` (`0xb794f5ea0ba39494ce839613fffba74279579268`) is used when making requests to the DID Registry.

As with most DID Method implementations, anyone is free to create a DID and register it with the `DID Registry` smart contract.

The smart contract will support the following methods:

1. `register(didAddress: address, endpoints: string[], signature: string)` &mdash; Register a list of endpoints for a `did` where the DID Document can be located. Updates the endpoints if the `did` is already registered. Increments the `nonce` for the `did`. Sets the `didControllerAddress` to be the `didAddress`.
2. `lookup(didAddress: address): [didControllerAddress: address, endpoints: string[]]` &mdash; Lookup the endpoints for a given `did`. Returns the current DID Controller address and an array of endpoints. Result is only returned if the DID exists in the smart contract and is not revoked.
3. `setController(didAddress: address, controller: address, signature: string)` &mdash; Change the DID that controls the DID Document. The new `controller` must sign for all new changes to the DID endpoints in this smart contract or the DID Document.
5. `revoke(didAddress: address, signature: string)` &mdash; Revoke a DID so it is no longer valid. This can not be undone.
6. `nonce(didAddress: address): number` &mdash; Obtain the next `nonce` required to update a list of `did` endpoints.


The **`register()`** method will verify the `signature` is a signature generated from a`proofString`, where:

```
proofString = sign(`${didAddress}/${endpoints[0}/${endpoints[1}/.../${nonce}}`, didPrivateKey)
```

The **`setController()`** method will verify the `signature` is a signature generated from a `proofString`, where:

```
proofString = sign(`${didAddress}/setController/${newControllerDidAddress}/${nonce}`, oldControllerDidPrivateKey)
```

The **`revoke()`** method will verify the `signature` is a signature generated from a `proofString`, where:

```
proofString = sign(`${didAddress}/revoke/${nonce}`, controllerPrivateKey)
```

The **`nonce`** value prevents replay attacks when changing the list of endpoints that a storing the DID Document.

### Blockchain deployment

The smart contract will be deployed on:

- `testnet` - Polygon Mumbai
- `mainnet` - Polygon PoS

## DID Document

### Create and Update

DID Documents are stored as [JSON](https://www.w3.org/TR/did-core/#json) or [JSON-LD](https://www.w3.org/TR/did-core/#json-ld) representations. It's recommended to use the [did-document (or custom fork)](https://www.npmjs.com/package/did-document) to generate the DID Documents.

The DID Document ___MUST___ include the following properties:

- [versionId](https://www.w3.org/TR/did-spec-registries/#versionid)
- [created](https://www.w3.org/TR/did-spec-registries/#created)
- [updated](https://www.w3.org/TR/did-spec-registries/#updated)
- [deactivated](https://www.w3.org/TR/did-spec-registries/#deactivated)
- `proof` &mdash; A string representing the full DID Document as a JSON encoded string using the `EcdsaSecp256k1VerificationKey2019` algorithm, to be compatible with Ethereum signatures.

The implementation on how to create the document is out of scope. ie: For a HTTP endpoint a HTTP server will be required and provide a way to create DID Documents on the server, likely stored on disk or in a database.

An endpoint should make every effort to verify the DID Document is valid before creation:

1. Ensure there is currently no entry for the given `DID` in the `DID Registry` _OR_ there is currently an entry and it references this storage node endpoint
2. The DID Document is a valid DID Document using the `did-document` npm package
3. The DID Document has valid properties (ie: valid `proof`, `versionId` etc.)

Endpoints ___MUST___ store all known versions of a DID Document.

### Proof creation

The `proof` in the DID Document is generated as follows:

1. Convert to the JSON object into a string (`JSON.stringify()`)
2. The resulting JSON is converted to a UTF8 byte string
3. The byte string is hashed using `keccak256` (`ethers.utils.keccak256()`)
4. The hash is then signed with the controller private key using `secp256k1` (`ethers.utils.SigningKey.signDigest()`)

This `proof` signature is then added to the JSON object.

This is implemented as [helper method `signProof()` in the Verida DID-Document library](https://github.com/verida/verida-js/blob/9107fa823145ddf746716dbc06c5e219f8b10aa2/packages/did-document/src/did-document.ts#L328)

### Proof verification

The `proof` in the DID Documet is verified as follows:

1. Remove the `proof` property from the JSON object
2. Convert to the JSON object into a string (`JSON.stringify()`)
3. The resulting JSON is converted to a UTF8 byte string
4. The byte string is hashed using `keccak256` (`ethers.utils.keccak256()`)
5. The signing address is then recovered with `ethers.utils.recoverAddress()` using the `proof` string as the signature
6. The signing address must match the DID controller

This is implemented as [helper method `verifyProof()` in the Verida DID-Document library](https://github.com/verida/verida-js/blob/9107fa823145ddf746716dbc06c5e219f8b10aa2/packages/did-document/src/did-document.ts#L344)

### Deletion

It ___MUST___ be possible for a DID Controller to delete their DID Document from an endpoint.

It is up to each endpoint to determine how a DID controller can delete a DID Document.

## DID Registration

The DID will be registered on the blockchain via the `DID Registry` smart contract.

Registering a DID requires:

1. A `nonce`. When creating a DID for the first time `nonce=0`, whereas subsequent updates to the list of endpoints must use the next `nonce` obtained by incrementing the value returned from calling `nonce(did: string)` on the smart contract.
2. A valid `signature` generated as per the [Smart Contract](#smart-contract)

A new DID is registered by calling `register(did: string, endpoints: string[], signature: string)` on the `DID Registry` smart contract

### Endpoint Discovery

The DID Document is stored at one or more URI endpoints.

These endpoints can be obtained by calling `lookup(did:string)` on the `DID Registry` smart contract.

### Retrieval

DID Documents can be retrieved in a universal way from any valid endpoint.

Endpoints that are storing DID Documents must support returning a valid response via a HTTP `GET` request. Endpoints must support the following optional query parameters:

1. [versionId](https://www.w3.org/TR/did-spec-registries/#versionId-param) &mdash; Returns the DID Document that matches the requested `versionId`.
2. [versionTime](https://www.w3.org/TR/did-spec-registries/#versionTime-param) &mdash; Returns the DID Document that was valid at the given timestamp.
3. `allVersions` &mdash; Returns all stored versions of the DID Document if value is `true`.

The latest DID Document version is returned if no query parameters are provided.

When `allVersions` is requested, the response should be a JSON encoded ordered array:

```
{
  versions: [
    documentVersion0,
    documentVersion1,
    documentVersion2,
  ],
}
```

`version` is a version ordered (0 first) array of all previous versions of the DID Document.

Example requests:

1. `https://au.storage.verida.io/did:vda:0xb794f5ea0ba39494ce839613fffba74279579268` &mdash; Resolve latest DID Document from `au.storage.verida.io`
2. `https://au-22.storage.alchemy.io/did/0xb794f5ea0ba39494ce839613fffba74279579268?versionTime=2016-10-17T02:41:00Z` &mdash; Resolve the DID Document version from `au-22. .storage.alchemy.io` that was valid at `2016-10-17T02:41:00Z`.
3. `https://au-22.storage.alchemy.io/did/0xb794f5ea0ba39494ce839613fffba74279579268?allVersions=true` &mdash; Return all versions of a DID Document.

### Resolving

A DID Document is resolved by:

1. Completing a [DID Document Lookup](#did-document-lookup) to locate the list of service endpoints storing the DID Document
2. [Retreiving](#did-document-retrieval) of the DID Document from each of the valid endpoints
3. Reaching consensus on the correct DID Document
4. Validating the DID Document
5. Returning the DID Document

### Consensus

The retrieved DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining > 50% consensus on the correct version will be executed. If consensus can not be reached the DID Document fails to resolve.

### Validation

The resolved DID Document is validated as follows:

1. The DID Document is a valid document meeting the [DID Core](https://www.w3.org/TR/did-core/) specification.
1. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.
2. The DID Controller in the DID Document matches the DID Controller stored on-chain.

### Versioning

DID Documents ___MUST___ be versioned.

This specification ensures all DID Documents have a `versionId` parameter, that all endpoints store all versions and that DID Documents can be queried by `versionId`.

Additionally, the ability to query a DID Document by `timestamp`, ensures the correct DID Document version at a particular time can be resolved. This is critical for key rotation as previously signed data may be using a public key no longer in the latest DID Document.

### Key rotation

It is possible to [rotate _verificationMethod_ keys](https://www.w3.org/TR/did-core/#verification-method-rotation) by writing an updated DID Document with new public keys.

it is possible to [rotate the DID ocontroller](https://www.w3.org/TR/did-core/#changing-the-did-controller) by:

1. Writing an updated DID Document that changes the [controller property of the DID Document](https://www.w3.org/TR/did-core/#did-controller).
2. Calling the `setController()` method on the `DID Registry` smart contract

_Note: Client libraries **must** ensure both of those steps are completed for a successful controller key rotation otherwise DID resolution and version verification will fail.

# Backwards compatibility

This will not be backwards compatible with the existing centralized Verida DID Registry. Users will be required to create new DIDs and previously created DIDs will no longer resolve.

# Security considerations

An endpoint operator may be malicious by altering the DID Document or returning an older version. DID controllers can store their DID Document across multiple storage nodes with different endpoint operators. This ensures one malicious operator can be identified (and their response discarded) assuming > 50% of endpoints are trusted. The consensus mechanism uses the `versionId` and `proof` verifications to ensure the documents themselves can be trusted. An option to hash every update and store that on chain could be added in the future for enhanced trust that may be necessary for high profile DIDs.

There is a possibility of replay attacks, where someone attempts to submit an old (previously submitted) list of endpoints to the `DID Registry`. The `nonce` avoids these types of attacks.

There is a possiblity of a malicious endpoint operator generating a vulnerable DID Document that replaces a valid DID Document. The `proof` embedded in the DID Document avoids any third party tampering or generating an invalid DID Document.

# Redundancy

There is a possibility of an endpoint becoming unavailable (either temporarily or permanently). It is expected taht DID Documents will be stored across any number of storage nodes. This ensures redundancy in the event one or more storage nodes become unavailable.

Ideally DID Documents will be stored across a minimum of 3 endpoints.

# Blockchain

The smart contract will be deployed to the Polygon PoS network due to it's low confirmation times, low cost and widespread adoption.

# Implementations

There is a solidity implmentation available that implements this specification:

[VDA-DID-Registry](https://github.com/verida/blockchain-contracts/tree/develop/VDA-DID-Registry).

There is a client side typescript library that makes it easy to use this smart contract to create, update and delete DIDs:

[@verida/vda-did](https://www.npmjs.com/package/@verida/vda-did) ([source code](https://github.com/verida/verida-js/tree/main/packages/vda-did))

The Verida typescript implementation has a user friendly library that wraps all this functionality:

[@verida/did-client](https://www.npmjs.com/package/@verida/did-client) ([source code](https://github.com/verida/verida-js/tree/main/packages/did-client))

# Related

Also see:

- [VIP-3: DID Document Storage on the Verida Network](./vip-3.md)
- [VIP-4: Verida DID Resolver](./vip-4.md)

# Compliance

@todo Expand this to cover address the _Must_ have documentation requirements from https://www.w3.org/TR/did-core/#methods
https://www.w3.org/TR/did-core/#method-operations

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
