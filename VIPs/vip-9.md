---
vip: 9
title: Verida Bank Account
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2023-03-21
---

# Introduction

The Verida protocol will eventually have multiple smart contracts that require users to `bond` or `lock` tokens:

- **Service registry**: Infrastructure providers bond tokens to register database nodes on the network. End users deposit tokens to pay for the infrastructure.
- **Interactions registry**: Verida accounts deposit tokens to send messages (and other interactions), message recipients receive tokens
- **Trust registry**: Verida accounts lock tokens when registering themselves on the trust registry (ie: officially registering themselves as a company, or trusting another company)

The service registry currently implements `addCredit()` and `removeCredit()` methods that support depositing VDA tokens that can then be used transfer VDA tokens between users based on the service infrastructure each account uses.

In the future, the `interactions registry` and the `trust registry` will require similar functionality.

The protocol requires a stand alone "Verida Bank Account" smart contract that provides generic functionality that can be leveraged by any smart contract or decentralized application:

- Support a user `owning` tokens on a per DID basis, instead of per wallet (effectively providing multi-chain account abstraction at the DID level)
- Support `locking` tokens for a specific purpose
- Support `recurring payments` of tokens
- Support applications `sponsoring` users with tokens for a specific purpose (ie: data storage)

This 

This design will allow for a simpler end user experience. A user can lookup at the `vda bank` smart contract and see how many of their tokens are locked by different verida smart contracts, how much credit they have and where their tokens are being spent across the Verida network.

This also provides the best way to allow VDA community token holders to support the network (via gasless transaction payments) while also receiving a return for their staking.

# User stories

- As a `protocol` I can reward a user with VDA tokens that are locked for a period of time
- As a `user` I can agree to a recurring payment to a storage node provider to pay for storage
- As a `smart contract` I can lock tokens 

TBA

- Recurring payments
- Sponsored storage for apps
- Token rewards that are locked for a period of time
- Sponsored transactions, supported by network stakers
- Protocol fees to support the network

# Specification

## Variables

### withdrawalFeePct: number

The percentage fee paid to the protocol when withdrawing tokens.

Default: `0.001` (=`0.1%`)

## Interfaces

### SubscriptionDetail

- `id`
- `interval`
- `amount`

### LockDetail

- `id`: `number` representing the unique lock identifier for the DID
- `type`: `enum` where `0=time`, `1=did`
- `amount`: `number` that is the amount of tokens locked
- `value`: `string` that is a `timestamp` if `type=type` or `address` if `type=did`

## Methods: Deposit

### `depositCredit(did: string, amount: number, tokenAddress: address)`

Add credit to a Verida DID.

### `depositCreditLockedByAccount(did: string, amount: number, tokenAddress: address, lockedByDid: string): number`

Add credit to a Verida DID, but lock it until another DID releases the tokens.

`lockedByDid` specifies the DID that can unlock the tokens.

Returns a `lockId` that is unique for the `did`.

### `depositCreditLockedByTimestamp(did: strijng, amount: number, tokenAddress: address, lockedTimestamp: number): number`

Add credit to a Verida DID, but lock it until a specific unix timestamp.

`lockedTimestamp` specifies the unix timestamp that must be reached before the tokens can be unlocked.

Returns a `lockId` that is unique for the `did`.

## Methods: Unlock

### `unlockCreditLockedByAccount(did:string, lockId: number, contextSignerProof: string, requestProof: string)`

Unlock credit that has been locked by another DID.

`contextSignerProof` is the proof that links a Verida context signing proof to a Verida DID (see Verida Verification Base library).

`requestProof` is a signature of the method parameters (see Verida Verification Base library).

The lock will be set to `expired` and the credit added to the owner's total balance.

### `unlockCreditLockedByTimestamp(did: string, lockId: number): boolean`

Unlock credit that is locked for a given time period. Credit will only unlock if `block.timestamp > unlockTimestamp`

Returns `boolean` = `true` if the unlock was accepted.

The lock will be set to `expired` and the credit added to the owner's total balance.

## Methods: Withdraw

### `withdrawCedit(did: string, amount: number, tokenAddress: address, requestProof: string)`

Remove credit from a Verida DID.

A fee equivalent to `$withdrawalFeePct` will be subtracted from the withdrawal amount and retained by the protocol.

### `withdrawFees(address: address) owner only`

Remove earned protocol fees from the smart contract.

### Methods: Transfers

### `createRecurringPayment()`

TBA

### `transferCredit(sourceDid: addres, destinationDid: address, tokenAddress: address)`

Transfer credit from one DID to another DID.

## Methods: Information

### `getBalance(did: string, tokenAddress: address): number`

Get the current balance for a DID and token.

### `getSubscriptionIds(did: string, tokenAddress: address): number[]`

Get an array of subscription Ids

### `getSubscriptionDetail(did: string, tokenAddress: string, subscriptionId: number): SubscriptionDetail`

Get detail of a subscription

### `getLockIds(did: string, tokenAddress: address): number[]`

Get an array of locked token IDs

### `getLockDetail(did: string, tokenAddress: string, lockId: number): LockDetail`

Get detail of a token lock