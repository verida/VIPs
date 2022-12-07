---
vip: 6
title: Verida Database: Support Server Side Replication
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-12-07
---

# Overview

The protocol needs to be upgraded to support multiple replica's of storage node data to ensure data redundancy. An intial draft implementation for the Acacia release took a client side approach, whereby the client library opened connections to all replica storage nodes and sync'd updates with all nodes at once. This was too slow and bloated as CouchDB is a very chatty protocol. Opening multiple databases takes more than a few seconds and requires open socket connections for each database.

It's proposed to implement replication server side to address these issues. However, this introduces other issues around security and performance.

# Background information

CouchDB supports database replication between servers. This is very flexible and can be configured as either "push" or "pull" replication.

- "Push" replication will push any new changes from a local database to a remote database
- "Pull" replication will pull any new changes from a remote database to a local database, but requires polling to check for any changes on the remote database

As such, "Push" replication is more efficient and is the preferred approach.

We also need to consider the security of removing replication nodes. If a DID removes a storage node from it's list of valid replication nodes, the other nodes need to be able to verify this change and ensure:

1. They stop pushing data to the removed node
2. They prevent the removed node from pushing data to it

The DID document maintains the source of truth of the valid endpoints that should be replicating a storage node. As such, storage nodes can independently verify if another storage node should have access and can accept / deny any replication authorization requests accordingly.

# Requirements

## Granting storage node replication access

The "destination node" must generate a JWT auth token that enables the "source node" to write new data (via push). It's critical that the token can provide `read/write` access, but not provide full `admin` access that enables things like adding and removing users from accessing the database.

This can be achieved by creating a `storage node replication user` on the "destination node" that represents the "source node". The user will HAVE the following properties:

- `username` = `hash(sourceNodeEndpointUri)`
- `roles` = [`${contextHash}-replicator`]

A `storage node replication user` has a `${contextHash}-replicator` role that grants read and write access to all databases in the given context. This is supported via the `members.roles` in each database security document (read access) and via our custom `validate_doc_update()` method (write access).

A `storage node replication user` can have multiple roles to support permissions for multiple contexts.

Note: `contextHash` is actually a hash of the database owner DID and context name, so this hash is unique for a given DID and context. If the same DID has three contexts on the same storage node, there will be three `contextHash`'s.

The generated JWT must include the username, but not any roles and not expire.

Implementation notes:

1. **Per Database:** The `configurePermissions` write function needs to check `userCtx.roles` includes the role `${contextHash}-replicator`
3. **Per Database:** The `configurePermissions` security doc needs to add the role `${contextHash}-replicator` in the `members` list
4. **JWT:** The generated JWT should specify the username of the storage node, but not the roles (they will be sourced from the `_users` table so they can be revoked without revoking the JWT which isn't supported by CouchDb)
5. **User:** The storage node username must be given the role `${contextHash}-replicator`

## Revoking storage node replication access

A storage node endpoint can have their access removed by simply removing the `${contextHash}-write-replicator` role from their user entry.

## Initiating replication

Assume we have three storage nodes; SN-A, SN-B, SN-C:

1. SN-A pushes to SN-B, SN-C
2. SN-B pushes to SN-A, SN-C
3. SN-C pushes to SN-A, SN-B

A user can ping each storage node and confirm replication is configured.

ie: `https://storageNodeA/user/checkReplication`

This will lookup the DID document and obtain a list of active endpoints for the user DID and context. It will ensure it is pushing data to all the endpoints in the DID document (and remove any it shouldn't be pushing to) for all the databases owned by that user.

## Authorizing and sharing JWT auth tokens

The "source" storage node (SN-A) needs to obtain a JWT auth token from the "destination" storage nodes (SN-B, SB-C).

A token is obtained by talking directly to the other node in a trusted manner. A request is made to the endpoint:

`https://storageNodeB/system/replicationToken(endpointUri, did, contextName, timestampMinutes, signature)`

Where:

1. `endpointUri` is the URI of SN-A that exactly matches the URI stored in the DID document
2. `did` is the DID that owns the application context
3. `contextName` is the context being granted access
4. `timestampMinutes` is a unix epoch timestamp (in minutes not seconds) +/- 1 minute from now()
5. `signature` is a signature generated by singing the first 4 parameters with the private key of the "source" storage node (SN-A)

The destination storage node (SN-B) must verify the requesting storage node (SB-A) should have permission to replicate. It does this by looking up the DID document and confirming the requesting storage node should have access for the given context.

The destination storage node (SB-B) must also verify the request has actually come from the requesting storage node (SB-A). Each node publishes their Verida public key via the `/status` endpoint. The destination storage node will verify the signature was generated by the public key of the requester.

The signature is generated as follows:

```
const signature = sign(PK, {
  endpointUri,
  did,
  contextName,
  timestampMinutes
})
```

Where `timestampMinutes` is unix epoch timestamp in minutes (not the normal seconds). The storage node ensures the timestamp in the query parameters is within the last or next minute. This is effectively a timestamp nonce for enhanced security.

## Creating a database

User submits a request to `/user/createDatabase` endpoint on one of the storage nodes. This will set the `_security` document in the database and define the `validate_doc_update()` method. Both of these are configured to support any user with the `xxx-replicater` permission, so they will be the same across all storage nodes. CouchDb replication will replicate these documents.

This storage node will internally call `checkReplication(databaseName: string)` to ensure it is pushing database updates to the other storage nodes.

It's necessary for the all the other nodes to start replicating that database.

Once the database is created the client side will ping the remaining storage nodes: `https://storageNodeA/user/checkReplication?databaseName={databaseName}` to ensure that database is being replicated.

## Deleting a database

User submits a request to `/user/deleteDatabase` for every storage node. The server removes all replication entries for that database.

# Client side requirements

The client side can establish a connection to just one of the nodes and the server-side replication will ensure all the other nodes have matching data.

# Research next steps

## Confirm the role based data access assumptions are correct



## Confirm `push` replication works as expected