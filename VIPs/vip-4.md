---
vip: 4
title: Verida DID Resolver
author: Chris Were <chris@verida.io>
discussions-to: #dev-blockchain on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-11-10
---

# Overview

The Verida SDK will provide a Verida DID Resolver package (`vda-did-resolver`) that supports creating, updating and fetching DID Documents. It will provide a standard `getResolver()` function to ensure the Verida DID method will be compatible with existing DID resolver libraries.

# Implementation

This section defines the SDK methods that implements DID Document management.

## resolve(did: string, timestamp?: number, fullVerification?: boolean)

The resolver will execute a two step process:

1. Call `lookup(did)` on the `DID Registry` smart contract to receive a list ofservice endpoints where the DID Document can be found.
2. Fetch the latest DID Document from all the service endpoints to determine consensus on the current DID Document

The SDK will call the [GET REST API endpoint](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#get) on each of the endpoints to receive the latest DID Document from each storage node.

### Timestamp

The optional _timestamp_ parameter is used to select the version of the DID document that was correct at the given timestamp. If _timestamp_ is not provided, the latest DID Document is returned. This is helpful if signed data needs to be verified and it may have been signed with a key in an older version of the DID Document. The DID Document that was correct as of the date/time the data was signed is necessary to verify the signed data.

The retreived DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining > 50% consensus on the correct version will be executed. If consensus can not be reached the SDK will throw a `Invalid DID Document` Error.

The resolved DID document will then be verified as follows:

1. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.

### Full verification

The optional `fullVerification` boolean option will fetch all versions of the DID Document from each endpoint and perform verification of every version of the DID Document.

The SDK will use the storage node [GET REST API endpoint](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#get) with query parameter `allVersions=true`.

## create(didDocument: DIDDocument, endpoints: string[], privateKey: string)

A new DID Document will be a `didDocument` paramaeter as a `DIDDocument` object.

This is a two step process:

1. Call [`register(did: string, endpoints: string[], signature: string)`](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#did-registration) on the `DID Registry` smart contract. `signature` is signed using `privateKey` parameter.
2. Iterate through every endpoint to create the DID Document via a [POST REST API request](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#create). The DID Document must contain a valid signed proof.

## update(didDocument: DIDDocument, privateKey: string)

An updated DID Document will be submitted to all the storage node endpoints hosting the DID Document.

Any HTTP errors will be raised as Javascript `Error()`.

### Rotate controller

The latest DID Document should first be resolved and if the DID Document has a different `controller`.

If the controller has changed, then it's necessary to call `setController(didAddress: address, controller: address, signature: string)`]([https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#did-registration](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#smart-contract) on the `DID Registry` smart contract. `signature` is signed using `privateKey` parameter..

The `DID Registry` controller should be updated before the DID Document should be updated on endpoints.

## delete(did: string, privateKey: string)

A DID Document is deleted.

This is a two step process:

1. Call [revoke(didAddress: address, signature: string)](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#smart-contract) on the `DID Registry` smart contract. `signature` is signed using the `privateKey` parameter.
2. Iterate through every endpoint to create the DID Document via a [DELETE REST API request](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#delete). `signature` is signed using the `privateKey` parameter.

## removeEndpoint(did: string, endpoint: string, privateKey: string)

Remove an endpoint from the pool of endpoints that stores copies of the DID Document.

This is a three step process:

1. Call `resolve()` to fetch the latest DID Document and extract the current list of endpoints. Remove the requested endpoint or throw an error if not found.
2. Call [register(didAddress: address, endpoints: string[], signature: string)](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#smart-contract) on the `DID Registry` smart contract with the modified list of endpoints. `signature` is signed using the `privateKey` parameter.
3. Remove the DID Document from the removed endpoint via a [POST REST API request](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#delete). `signature` is signed using the `privateKey` parameter.

## addEndpoint(did: string, endpoint: string, privateKey: string, verifyAllVersions=true)

Add an endpoint to the pool of endpoints that stores copies of the DID Document.

This is a four step process:

1. Call `resolve()` to fetch the latest DID Document and extract the current list of endpoints. Add the requested endpoint.
2. Call [register(didAddress: address, endpoints: string[], signature: string)](https://github.com/verida/VIPs/blob/develop/VIPs/vip-2.md#smart-contract) on the `DID Registry` smart contract with the modified list of endpoints. `signature` is signed using the `privateKey` parameter.
3. Fetch all historical versions of the DID Document by calling the [GET REST API endpoint](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#get) with `allVersions=true` on all existing endpoints. If `verifyAllVersions=true` then verify the DID Document versions across all storage nodes are all identicial.
4. Migrate the historical versions of the DID Document the new endpoint via a [POST REST API request](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#migrate). `signature` is signed using the `privateKey` parameter.
