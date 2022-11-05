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

Storing DID Documents on-chain is permanent. There is a potential for identifying information to be placed in these critical documents. It is not possible to delete these documents from the blockchain. This change ensures a user has self-custody over their DID Document and can delete it forever, providing enhanced privacy and control over their identity.

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

Question: Support flagging a `DID` as `deleted`?

### DID Document

creation
storage
lookup

## Backwards compatibility

This will not be backwards compatible with 

## Security considerations

## 

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
