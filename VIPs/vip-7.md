---
vip: 7
title: Storage Node Discovery &amp; Economics
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-01-28
---

# Overview

Need:

- Storage nodes to be discoverable by users of the protocol
- Storage nodes provide a reliable, fast service
- Storage nodes get paid

Payment will be with the VDA token

Users pay for `slots` of storage, equivalent to 50MB of database storage.

# Flow

1. User selects 3x nodes
2. User 

## Storage node registration

## Storage node discovery

## User registering a slot

# Economics

Assume the Verida token is equivalent to 0.15 USDC

## Example: User

_TLDR; A user connected to 10 applications will pay  in VDA tokens (4.50 USDC) which provides a total of 500MB decentralized storage with at least 3x replicas of all data._

A user opens an application context on the network and selects 3x storage nodes to host their application data. By default, the user acquires 50MB of database storage. Each unit of 50MB of storage on a storage node is called a `slot`. The user is acquiring 3 `slots` on the Verida network.

The user pays 1 VDA (0.15 USDC) to each of the storage nodes per month. This is a total cost of 3 VDA (0.45 USDC) / month to store their data in that application.

The user pays upfront for a given month. They can switch storage nodes during a given month, but will not be given refunds for partial months.

A user joining part way through a given month will only pay for a partial month.

## Example: Storage Node

A storage node operator spins up a node with 1TB of storage. This enables them to provide a maximum of 20,000 `slots`:

```
1,000,000 MB / 50 MB = 20,000
```

The storage node operator must stake 5x the number of tokens they can earn in a given month. For 1TB of storage that will be 100,000 VDA. The node operator may optimize this by gradually increasing the amount staked, with the number of slots they provide to the network.

After the first month, the storage node operator reaches 50% capacity (10,000 `slots`) that generates 10,000 VDA tokens in revenue.

The storage node operator makes a claim on-chain to receive the tokens owed for that month.

Running the storage node for 12 months at full capacity would generate 120,000 VDA tokens. This is an annual return of 120%, ignoring operational costs of running the node.

## Variables

### Stake ratio

_The number of VDA tokens a storage node operator must stake, per `slot`, in order to join the network._

A higher number requires more up-front cost to provide storage providing a lower return on capital. A lower number requires less up-front cost and increases the return on capital.

The network will start with a stake ratio of 5, meaning a storage node must stake 5 VDA per 50Mb `slot` of storage they provide the network.

This stake ratio will initially be managed by the Verida team to best support the network, but will eventually become dynamic based on the storage requirements of the network.

### Slot price

_The number of VDA tokens a user must pay per month, per `slot`, to access 50MB of database storage_

The swap rate of the VDA token against stable coins will fluctuate over time. This, in turn, will fluctuate the price of storage on the network when converted to a stable coin price (ie: USDC).

The slot price will initially be managed by the Verida team, with an object to keep the price of storage close to 0.15 USDC per `slot` per month.

This will default to 1, meaning a user pays 1 VDA for 50Mb of database storage on the network. For 3x replicas they will need to pay 3 VDA.

### Slot capacity

_The number of Megabytes (MB)

### Epoch length

_The number of days between payments for a `slot`_

This will default to 28 days, meaning a user that pays for a `slot` is paying for 28 days of storage. Anywhere this document references a `month`, is a reference to this epoch length.

## Paying for storage

## Claiming earnings

## User payment failure

A user may have insufficient funds to pay for storage for the upcoming month.

Storage node operators will retain the user data for a minimum of 30 days. After that period, storage node operators are free to delete the user data and free up that slot.

If a user adds sufficient funds to their account within that 30 day period, the storage node operator must continue 



# Reliability

## Storage node failure

## Uptime

All the storage nodes detect when nodes they are replicating go down. Automatically sign and record, on-chain, the downtime. Provide a proof (from DID doc?) that they are replicating to that node at the request of a user.

If there is poor uptime, the user will stop using the node.

## Latency

If there is poor latency, the user will stop using the node.

# Switching nodes

Users can switch nodes. How? Economic impact?

# Protocol changes

1. Need a 
2. Need a bank account with recurring payments and "holds"
3. Need the concept of a "month"