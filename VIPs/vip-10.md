---
vip: 10
title: Storage node registration (on-chain)
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2023-03-22
---

# Overview

The selection of storage nodes occurs when a DID is first created and when a context is connected to for the first time. The storage nodes are used to store the actual DID Documents and the same nodes are typically (but not necessarily) used to store data for each application context.

This VIP provides a specification for storing the available storage nodes on-chain.

This is phase 2 of [VIP-7: Storage node discovery and selection (off-chain)](./vip-7.md)

This VIP does not implement any token based staking or slashing of nodes, it's purely focused on node addition, removal and discovery.

# Motivation

The current storage node database is centralized and needs to be stored on-chain to decentralize the discovery and selection of nodes on the network.

# Specification

## Data structures

### StorageNode

- didAddress: string
- endpointUri: string
- countryCode: string
- regionCode: string
- datacenterId: id

### Datacenter

- id: int
- countryCode: string
- regionCode: string
- lat: int
- long: int

## State variables

TBA

## addNode(didAddress: string, endpointUri: string, countryCode: string, regionCode: string, datacenterId: string, requestSignature: string)

Storage nodes must register themselves with the protocol so they can be discovered by new users. Storage nodes provide additional metadata (via `/status` endpoint) that assists users select the most appropriate storage nodes to use.

It's not possible to re-use a DID to register multiple storage nodes. This is because storage endpoints sign their responses using the DID private key to enhance security of the protocol.

This method registers a new endpoint on the network:

1. `didAddress`: The DID that is associated with the storage node. Must be only one `didAddress` registered at any time.
2. `endpointURI`: The storage node endpoint. Must be only one `endpointURI` registered at any time.
3. `countryCode`: Unique two-character string code (see [VIP-7](./vip-7.md))
4. `regionCode`: Unique region string code (see [VIP-7](./vip-7.md))
5. `datacenterId`: Unique datacenter identifier. Must match a datacenter previously created with `addDatacenter()`.
6. `requestSignature`: The request parameters signed by the `didAddress` private key. Will be verified by `VDA-Verification-Base` library.

An `establishmentDate` will be saved that matches the current unix timestamp (`block.timestamp`).

## removeNode(didAddress: string, unregisterDatetime: uint, requestSignature: string)

Unregister a storage node from the network at the specified date:

1. `didAddress`: The DID that is to be removed from the network
2. `unregisterDatetime`: The unix timestamp of when the storage node should no longer be available for selection. Must be at least 28 days in the future.
3. `requestSignature`: The request parameters signed by the `didAddress` private key. Will be verified by `VDA-Verification-Base` library.

Note: This does not mean the node is no longer active and managing data for users on the network. It just means it is no longer discoverable for new connections.

## getNodeByAddress(didAddress: string): StorageNode

## getNodeByEndpoint(endpointUri: string): StorageNode

## addDatacenter(name: string, countryCode: string, regionCode: string, lat: int, long: int) ownerOnly

Add a data center to the network. `id` will be autoincremented.

Returns `int` the unique ID of the created data center

## removeDatacenter(id: string) ownerOnly

Remove a data center by `id`. Will only remove the data center if there are no storage nodes using that datacenter.

## getDatacenters(ids: string[]): Datacenter[]

Get details of a list of data centers by `id`.

## nodesByCountry(countryCode: string): StorageNode[]

Find all active nodes by country code.

Question: Does this require pagination? It could return 100's of results.

## nodesByRegion(regionCode: string): StorageNode[]

Find all active nodes by region code.

Question: Does this require pagination? It could return 100's of results.

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).