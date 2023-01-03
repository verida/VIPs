---
vip: 7
title: Storage Node Discovery &amp; Selection
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-01-03
---

# Overview

The selection of storage nodes occurs when a DID is first created. The storage nodes are used to store the actual DID Documents and the same nodes are typically (but not necessarily) used to store data for each application context.

This VIP provides a specification for the discovery of the available storage nodes and default rules for selecting an appropriate storage node.

This functionality is only required by Verida wallet implementations (ie: `Verida Vault`) and Verida clients that maintain their own wallet (ie: Server applications that maintain their own Verida network private key).

# Motivation

The Verida protocol requires data to be stored on multiple storage nodes to enable redundancy and maximise performance in a decentralized environment.

It's necessary to provide a simple storage node discovery and selection capability into the protocol to:

1. Minimize developer friction when using the protocol
2. Minimize new user friction when creating a DID
3. Maximising flexibility for advanced users
4. Maximising network performance for end users

# Discovery

Storage nodes must register theselves with the protocol so they can be discovered by new users. These storage nodes will provide global metadata that can assist new users select the most appropriate storage nodes to use.

The protocol will provide a default set of logic to determine the most appropriate storage nodes for a new user, however users will be able to override this and manually select the storage nodes they wish to use. It is, after all, their data.

Self-hosted storage nodes can be created and used by advanced users. They don't need to be registered with Verida. These nodes can be shared privately amongst groups of users, but won't be discoverable by general users of the Verida network.

The discovery process will be implemented in two phases.

## Phase 1

Verida will manually verify storage node operators and maintain a list of trusted storage nodes with accurate metadata about each storage node.

## Phase 2

A Verida smart contract will be deployed that permits any storage node operator to register themselves on the network.

This phase is out of scope for this VIP.

# Selection considerations

The process of selecting appropriate storage nodes requires consideration of:

- Geographic location
- Availability
- Performance
- Regulatory jurisdiction
- Node operator
- User preference

The protocol must specify a default selection process for new users. It's intended that advanced users can select their own storage nodes at any time (including selecting self-hosted storage nodes).

## Geographic location

The location of the storage node could consist of the following relevant metadata:

- Country
- Region (ie: Europe, South East Asia)
- Datacenter
- Latitude / Longitude

## Recommendation

1. A registry of all datacenters will be created (`Datacenter Registry`) that has a `uniqueId`, `name`, `countryLocation`, `latitude` and `longitude`.
2. Each storage node will be included in the `Storage Node Registry` and specify the `uniqueId` of the datacenter where the node is located.
3. A registry of all regions will be created (`Region Registry`) that maps `country` to `region`. This will be helpful for the client protocol to narrow down the most appropriate storage nodes to consider for selection.

## Availability

The availability of a storage node could consider the following relevant metadata:

- % uptime (ie: 99.99% uptime)
- Establishment date (ie: created 1 day ago is less trustworthy than created 1,000 days ago)
- Outages (ie: 5 outages in the last day)

## Recommendation

1. Providing dynamic feedback on uptime and outages is a non-trivial process, so `% uptime` and `outages` is out of scope for now.
2. `establishmentDate` will be included in the `Storage Node Registry`.

## Performance

The performance of a storage node could consider the following relevant metadata:

- Latency for the current user
- Average latency for all users
- Upload and download speed

### Recommendations

1. Verida client implementation will provide a method to easily obtain the `latency` for a storage node (measured in milliseconds).
2. Providing dynamic feedback on uptime and outages is a non-trivial process, so `average latency` and `up / down speed` is out of scope for now.

## Regulatory jurisdiction

The regulatory jurisdiction of a storage node could consider the following relevant metadata:

- Country where the storage node is located
- Country of legal entity operating the storage node

Some governments have the right to access digital infrastructure if it is located in a particular country. Similarly some governments have the right to access digital infrastructure, regardless of country the data is located, if the infrastructure operator falls under their jurisdiction.

### Recommendations

1. Require storage nodes to specify their `countryResidence`. This information will not be used other than to display to the user.
2. Require storage nodes to specify their `countryLocation` (as per geographic region above). This will be enabled by the storage node specifying their `datacenter`, which includes the `countryLocation`.

## Node operator

It may be preferable for Users to have nodes operated by different operators. ie: You may not want all your nodes operated by a particular organization.

## User preference

Users may have specific preferences. Here are some examples:

1. I only care about having the fastest possible experience
2. I don't trust my government, so want to ensure my data is stored in another country, regardless of how slow it is
3. I want to maximise redundancy, so want to choose storage nodes across 5 different countries

The protocol will focus on providing sensible default for all users, but it will be possible for advanced users to completely customize where their data is stored.

# Specification

## Discovery

### Storage Nodes

The list of verified storage node operators will be maintained in a JSON file at a known URL (`Storage Node Registry`). This URL endpoint will be updated by the Verida Foundation and embedded in the Verida protocol client implementation.

This file will be located at:

1. https://meta.verida.network/registry/storageNodes/testnet.json
2. https://meta.verida.network/registry/storageNodes/mainnet.json

Example file:

```
[
    {
        "id": "verida-testnet-aws-us-east1-001",
        "name": "Verida Foundation Testnet: AWS-US-EAST1-001",
        "description": "Node operated by the Verida Foundation on Testnet",
        "datacenter": "aws-us-east1",
        "serviceEndpoint": "https://001-use1.tn.verida.tech/",
        "establishmentDate": "2023-01-03T08:22:35Z",
        "countryResidence": "SG"
    },
    ...
]
```

### Data Centers

The list of official data centers will be maintained in a JSON file at a known URL (`Datacenter Registry`). This URL endpoint will be updated by the Verida Foundation and embedded in the Verida protocol client implementation.

This file will be located at:

1. https://meta.verida.network/registry/dataCenters/testnet.json
2. https://meta.verida.network/registry/dataCenters/mainnet.json

Example file:

```
[
    {
        "id": "aws-us-east1",
        "name": "Amazon Web Services: US East1 (North Virginia)",
        "countryLocation": "US",
        "latitude": 38.9809166,
        "longitude": -77.4692733,
    },
    ...
]
```

### Regions

The list of official regions and their associated countries will be maintained in a JSON file at a known URL (`Datacenter Registry`). This URL endpoint will be updated by the Verida Foundation and embedded in the Verida protocol client implementation.

This file will be located at:

1. https://meta.verida.network/registry/regions/testnet.csv
2. https://meta.verida.network/registry/regions/mainnet.csv

Country codes will be two character alpha representations of the country. Regions will be fixed strings.

The data will be stored as a CSV instead of JSON to save space. The data will be sourced from [an existing csv file](https://github.com/lukes/ISO-3166-Countries-with-Regional-Codes/blob/master/all/all.csv).

Example file:

```
"AF","Afghanistan","Asia"
"AU","Australia","Oceana"
"US","United States of America","North America"
```

## Selection

The default storage node selection process needs to prioritize:

1. Minimize latency
2. Maximize redundancy

The default selection process will aim to efficiently find the three lowest latency nodes stored in different datacenters located in the most jurisdictioanlly appropriate country.

This will ensure data is replicated at least three times for redundancy, while the fastest nodes will be selected based on the user's location.

### Jurisdiction filter

The list of all available nodes will be queried, with only storage nodes that match the current user's `countryLocation` included.

If there is less than three storage nodes available (across different data centres), then the filter is expanded to also include all storage nodes where the storage node `countryLocation` matches the region of the end user.

For example; The user is located in "Vietnam" which has no available storage nodes on the network, so matches all storage nodes from countries tht are located in the region "Southeast Asia".

### Datacenter grouping

The list of available nodes will be grouped by datacenter `uniqueId`. The list of nodes associated with each datacenter will be randomized.

### Lowest latency (Fastest)

A large network could have thousands of nodes after the jurisdiction filter is applied. As such, it's necessary to further shortlist the storage nodes to those most likely to have the lowest latency.

This may include:

1. `longitude` / `latitude` of the current user (obtainable on a mobile device by requesting location) is used as a query input against the `longitude` / `latitude` of all available storage nodes
2. Pinging each storage node to obtain the latency

The first storage node in each datacenter group will be pinged to obtain the latency. The three storage nodes with the lowest latency will be selected.

This can be improved in the future by:

1. Pinging more than one server per datacenter to allow for storage nodes with faster connectivity / more resources
2. Pinging datacenters closest to the user based on their `longitude` / `latitude`.


### Availability filter

A user can only connect to storage nodes that have sufficient capacity. This can be obtained by pinging the `/status` on a storage node.

As such, when pinging servers to measure latency, the `/status` endpoint will be hit. If the storage node has no availability, the next storage node in the list of that datacenter will be pinged.

# Backwards compatibility

# Security considerations

# Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).