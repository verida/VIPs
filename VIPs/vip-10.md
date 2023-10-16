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

It's expected that the operations relating to selecting storage nodes and handling the removal of nodes will be primarily managed by the Verida protocol and wallets that implement the Verida network.

# Motivation

The current storage node database is centralized and needs to be stored on-chain to decentralize the discovery and selection of nodes on the network.

# Specification

## Data structures

### StorageNode

- `didAddress` (string): The DID that is associated with the storage node. Must be only one `didAddress` registered at any time.
- `endpointUri` (string): The storage node endpoint. Must be only one `endpointURI` registered at any time.
- `countryCode` (string): Unique two-character string code (see [VIP-7](./vip-7.md))
- `regionCode` (string): Unique region string code (see [VIP-7](./vip-7.md))
- `datacenterId` (id): Unique datacenter identifier. Must match an existing datacenter available via `getDatacenters()`.
- `lat` (int): Latitude value multiplied by 10^8
- `long` (int): Longitude value multiplied by 10^8
- `numberslots` (int): The number of storage slots available on this storage node. Must be within the range of `MIN_SLOTS` and `MAX_SLOTS`.

### Datacenter

- `id` (int)
- `countryCode` (string)
- `regionCode` (string)
- `lat` (int)
- `long` (int)

## State variables

- `STAKE_PER_SLOT` (const): The number of tokens that must be staked by a storage node operator for each storage slot they make available to the network. `default=60`. Only contract owner can change at this stage. _In the future this will become dynamic based on market dynamics._
- `NODE_ISSUE_FEE` (const): The number of VDA tokens that must be deposited when recording an issue against a storage node. `default=5`
- `STAKING_REQUIRED` (const): Indicate if staking is required when registering a new node. `default=false`
- `MIN_SLOTS` (const): Minimum number of storage slots a storage node must provide. `default=20000`
- `MAX_SLOTS` (const): Maximum number of storage slots a storage node can provide. `default=20000`

## addNode(nodeInfo: StorageNode, requestSignature: string, requestProof: string, authSignature: string)

Storage nodes must register themselves with the protocol so they can be discovered by new users. Storage nodes provide additional metadata (via `/status` endpoint) that assists users select the most appropriate storage nodes to use.

It's not possible to re-use a DID to register multiple storage nodes. This is because storage endpoints sign their responses using the DID private key to enhance security of the protocol.

The requester must provide VDA tokens to stake, if `STAKING_REQUIRED=true`. The number of staked tokens must be `numberSlots` * `STAKE_PER_SLOT`. The staked tokens will be registered against the storage node being registered so they can be unstaked in the future (see `withdraw()` and `removeNodeComplete()`).

This method registers a new endpoint on the network:

1. `nodeInfo` : Storage node to be added
2. `requestSignature`: The request parameters signed by the `nodeInfo.didAddress` private key. Will be verified by `VDA-Verification-Base` library.
3. `requestProof` : Used to verify request. Signed by private key of `nodeInfo.didAddress`
4.  `authSignature` : Signature signed by a trusted signer.

An `establishmentDate` will be saved that matches the current unix timestamp (`block.timestamp`).

## removeNodeStart(didAddress: string, unregisterDatetime: uint, requestSignature: string, requestProof: string)

Request de-registering of a storage node from the network at the specified date:

1. `didAddress`: The DID that is to be removed from the network
2. `unregisterDatetime`: The unix timestamp of when the storage node will be removed from the network. Must be at least 28 days in the future to ensure users have sufficient time to migrate away from this node to another node. All users must expect the node to cease to operate and their data to be deleted from `unregisterDatetime` onwards.
3. `requestSignature`: The request parameters signed by the `didAddress` private key. Will be verified by `VDA-Verification-Base` library.
4. `requestProof` : Used to verify request. Signed by private key of `nodeInfo.didAddress`

Note: During the 28 day removal window, the node will continue to manage data for users connected to that node. However, it is not available for new connections as the node will become unavailable within 28 days.

## removeNodeComplete(didAddress: string, requestSignature: string, requestProof: string)

Complete the de-registering of a storage node.

1. `didAddress`: The DID that is to be removed from the network
2. `requestSignature`: The request parameters signed by the `didAddress` private key. Will be verified by `VDA-Verification-Base` library.
3. `requestProof` : Used to verify request. Signed by private key of `nodeInfo.didAddress`

Note: Before this function is called, connected `dataCenter` can't be removed.

When successfully called, the staked tokens for this storage node should be released to the requester.

## getNodeByAddress(didAddress: string): [StorageNode, string]

Get a storage node by `didAddress`. Includes an additional `status` value indicating if it is `active` or `removed` (indicating it is in the process of being de-registered).

## getNodeByEndpoint(endpointUri: string): [StorageNode, string]

Get a storage node by `endpointUri`. Includes an additional `status` value indicating if it is `active` or `removed` (indicating it is in the process of being de-registered).

## addDatacenter(data: Datacenter) ownerOnly

Add a data center to the network. `id` will be autoincremented.

Returns `int` the unique ID of the created data center

## removeDatacenter(id: string) ownerOnly

Remove a data center by `id`. Will only remove the data center if there are no storage nodes using that datacenter.

## getDatacenters(ids: string[]): Datacenter[]

Get details of a list of data centers by `id`.

## getDataCentersByCountry(countryCode: string)

Get details of a list of data centers by `countryCode`.

## getDataCentersByRegion(regionCode: string)

Get details of a list of data centers by `regionCode`.

## getNodesByCountry(countryCode: string): StorageNode[]

Find all active nodes by country code.

Question: Does this require pagination? It could return 100's of results.

## getNodesByRegion(regionCode: string): StorageNode[]

Find all active nodes by region code.

Question: Does this require pagination? It could return 100's of results.

## addTrustedSigner(didAddress: string)

Whitelist a Verida DID that can approve the addign of new storage nodes. This is a temporary measure until VDA token staking and other security measures are implemented.

## withdraw(didAddress: string, amount: int, requestSignature: string, requestProof: string)

Allow a storage node to withdraw staked tokens. The storage node can only withdraw excess tokens to ensure `amountStaked` > `STAKE_PER_SLOT * numberSlots` is always true.

This can happen if a node stakes tokens, then the `STAKE_PER_SLOT` decreases and the node can with draw the tokens that are no longer required for staking.

This also allows any account to withdraw tokens that may have been "earned" by logging an issue against a storage node.

## deposit(didAddress: string)

Allow a storage node to deposit more staked tokens.

This is necessary if the `STAKE_PER_SLOT` increases and the node must deposit more tokens to meet their staking requirements.

Note: We have not yet handled the scenario where STAKE_PER_SLOT is increased and storage node operators do not have enough tokens staked. This will be handled later. We will implement storage node operators earning a revenue from their nodes. They won't be able to earn revenue if they don't have enough tokens staked.

## slash(didAddress: string, amount: int, moreInfoUrl: string)

Allow the contract owner to slash staked tokens for a given storage node `didAddress`. This will only be used in extreme circumstances by the Verida Foundation and will be replaced by a more transparent, rules based slashing engine in a future release.

`moreInfoUrl` is a required field to create an on-chain pointer to where more information can be found about this slashing incident.

The slashed tokens will be distributed to all accounts that recorded an issue against the node `didAddress` in proportion the number of `NODE_ISSUE_FEE`s they deposited. These tokens can then be withdrawn via `withdraw()`.

## logNodeIssue(didAddress: string, reasonCode: int, requestSignature: string, requestProof: string)

Allow any valid Verida DID to record an issue against a node.

In order to prevent malicious reporting and spamming:

1. The account logging the issue must include VDA tokens equal to `NODE_ISSUE_FEE`.
2. Accounts can not log an issue against the same node more than every hour.
3. Accounts can not log more than 4 issues against any node for a given 24 hour period.

Valid `reasonCode`s:

- `10`: Poor performance
- `20`: Unavailable > 10 minutes
- `21`: Unavailable > 30 minutes
- `22`: Unavailable > 1 hour
- `23`: Unavailable > 3 hours
- `24`: Unavailable > 24 hours

The list of valid `reasonCode`s will expand over time.

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).