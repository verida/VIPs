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

## resolve(did: string, timestamp?: number)

The resolver will execute a two step process:

1. Call `lookup()` on the `DID Registry` smart contract to receive a list ofservice endpoints where the DID Document can be found.
2. Fetch the latest DID Document from all the service endpoints to determine consensus on the current DID Document

The SDK will call the [GET REST API endpoint](https://github.com/verida/VIPs/blob/develop/VIPs/vip-3.md#get) on each of the endpoints to receive the latest DID Document from each storage node.

The optional _timestamp_ parameter is used to select the version of the DID document that was correct at the given timestamp. If _timestamp_ is not provided, the latest DID Document is returned. This is helpful if signed data needs to be verified and it may have been signed with a key in an older version of the DID Document. The DID Document that was correct as of the date/time the data was signed is necessary to verify the signed data.

The retreived DID Documents will be compared to ensure consistency. If they are different, an attempt at obtaining > 50% consensus on the correct version will be executed. If consensus can not be reached the SDK will throw a `Invalid DID Document` Error.

The resolved DID document will then be verified as follows:

1. The DID Document `proof` was generated from an exact copy of the current DID Document and signed by the private key that controls the DID.
