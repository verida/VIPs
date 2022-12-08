---
vip: 6
title: Verida Database Support Server Side Replication
author: Chris Were <chris@verida.io>
discussions-to: #dev-protocol on Verida Discord
status: Draft
type: Standards Track
category: Core
created: 2022-12-07
---

# Overview

The protocol needs to be upgraded to support multiple replicas of storage node data to ensure data redundancy. An intial draft implementation for the Acacia release took a client side approach, whereby the client library opened connections to all replica storage nodes and sync'd updates with all nodes at once. This was too slow and bloated as CouchDB is a very chatty protocol. Opening multiple databases takes more than a few seconds and requires open socket connections for each database.

It's proposed to implement replication server side to address these issues. However, this introduces other issues around security and performance.

# Background information

CouchDB supports database replication between servers. This is very flexible and can be configured as either "push" or "pull" replication.

- `Push` replication will push any new changes from a local database to a remote database
- `Pull` replication will pull any new changes from a remote database to a local database, but requires polling to check for any changes on the remote database

As such, "Push" replication is more efficient and is the preferred approach.

We also need to consider the security of removing replication nodes. If a DID removes a storage node from it's list of valid replication nodes, the other nodes need to be able to verify this change and ensure:

1. They stop pushing data to the removed node
2. They prevent the removed node from pushing data to it

The DID document maintains the source of truth of the valid endpoints that should be replicating a storage node. As such, storage nodes can independently verify if another storage node should have access and can accept / deny any replication authorization requests accordingly.

# Requirements

## Granting storage node replication access

The `destination node` must generate a CouchDb user that enables the `source node` to write new data (via push). It's critical that the user has `read/write` access to the DID context databases, but not have full `admin` access that enables things like adding and removing users from accessing the database.

This can be achieved by creating a storage node `replication user` on the `destination node` for the `source node`. The user will have the following attributes:

- `username` = `hash(sourceNodeEndpointUri)`
- `roles` = [`${contextHash}-replicator`]

A storage node `replication user` has a `${contextHash}-replicator` role that grants read and write access to all databases in the given context. This is supported via the `members.roles` in each database security document (providing read access) and via our custom `validate_doc_update()` method (providing write access).

A storage node `replication user` can have multiple roles to support permissions for multiple contexts.

Note: `contextHash` is actually a hash of the database owner DID and context name. It is unique for a given DID and context. If the same DID has three contexts on the same storage node, there will be three `contextHash`'s.

Implementation notes:

1. **Per Database:** The `configurePermissions` write function needs to check `userCtx.roles` includes the role `${contextHash}-replicator`
2. **Per Database:** The `configurePermissions` security doc needs to add the role `${contextHash}-replicator` in the `members` list
3. **User:** The storage node username must be given the role `${contextHash}-replicator`
4. **Revocation:** A storage node can revoke the `${contextHash}-replicator` role to disable read/write access to databases it should no longer have replication access to

>Note: It's not possible to use JWT token authorization for replicaters due to the way CouchDB works. The JWT authentication implementation in CouchDB doesn't read the `roles` property of the `user` specified in the access token. As such, we can't use JWT tokens and specify a `user` to manage access. CouchDB does support specifying a `role` in the JWT token, however it doesn't support revoking access tokens. As such, it will be impossible to revoke that access when a replicater needs to be removed The above approach specifies a `role` in the user and then uses normal HTTP username/password authentication (which does respect the user roles).

## Initiating and revoking replication

Assume we have three storage nodes; SN-A, SN-B, SN-C:

1. SN-A pushes to SN-B, SN-C
2. SN-B pushes to SN-A, SN-C
3. SN-C pushes to SN-A, SN-B

A user can ping each storage node and confirm replication is configured for a given context.

ie: `https://storageNodeA/user/checkReplication`

This will have the following parameters:

- `databaseName` (optional parameter, otherwise checks all databases)
- `did` (via auth token)
- `contextName` (via auth token)

If a node is being added or removed it **must** be the last node to have `checkReplication` called. This ensures the node has a list of all the active databases and can ensure it is replicating correctly to the other nodes.

The client SDK should call `checkReplication()` when opening a context to ensure the replication is working as expected.

### Identify valid storage nodes

This will lookup the DID document and obtain a list of active endpoints for the user DID and context. It will ensure it is pushing data to all the endpoints in the DID document (and remove any it shouldn't be pushing to) for all the databases owned by that user.

This ensures the DID Document is the source of truth for the list of active database replicas for a `context`.

### Node has been added

The storage node may identify it has been added to the list of endpoints. The storage node doesn't need to do anything as the other nodes will contact it to initialize replication.

The storage node may identify a new replication node has been added. The storage node will request credentials from the new node (see below) and will create replication entries to `push` all the `context` databases to the new node.

This will include pushing the database that maintains a list of all the `context` databases.

Implementation notes:

- Need to refactor the storage of databases in a context to have their own database so they can be syncronized via CouchDB replication
- Need the client SDK to not start using the new node until it is fully syncronized (need to check `/status`)?

### Node has been removed

The storage node may identity it has been removed from the list of endpoints. The storage node can remove all replication entries associated with the `context` databases and delete the databases (including the database that maintains the list of `context` databases).

The storage node may identify a node it was replicating to has been removed. The storage node will remove the `${contextHash}-replicator` role from the `_users` collection so the removed node can no longer replicate data to this node for that `context`. It will also remove all replication entries associated with the removed node to stop pushing data to the removed node.

## Authorizing and sharing credentials

The `source` storage node (SN-A) needs to obtain credentials (username and password) from the `destination` storage nodes (SN-B, SB-C) when replication is configured for the very first time.

The credentials are obtained by talking directly to the other node in a trusted manner. A request is made to the endpoint:

`https://storageNodeB/system/replicationCreds(endpointUri, did, contextName, timestampMinutes, signature)`

Where:

1. `endpointUri` is the URI of SN-A that exactly matches the URI stored in the DID document
2. `did` is the DID that owns the application context
3. `contextName` is the context being granted access
4. `timestampMinutes` is a unix epoch timestamp (in minutes not seconds) +/- 1 minute from now()
5. `signature` is a signature generated by singing the first 4 parameters with the private key of the `source` storage node (SN-A)

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

The `destination` storage node will respond with:

```
{
  username: hash(endpointUri),
  password: random32CharString()
}
```

## Creating a database

An end user submits a request to `/user/createDatabase` endpoint on one of the storage nodes. This will set the `_security` document in the database and define the `validate_doc_update()` method. Both of these are configured to support any user with the `${contextHash}-replicater` role, so they will be the same across all storage nodes. CouchDb replication will replicate these documents ensuring permission consistency across all storage nodes.

This storage node will internally call `checkReplication(databaseName: string)` to ensure it is pushing database updates to the other storage nodes. It's necessary for the all the other nodes to start replicating that database.

The database will be added to the list of `context` databases, which will also be syncronized.

Once the database is created the client side will ping the remaining storage nodes: `https://storageNodeA/user/checkReplication?databaseName={databaseName}` to ensure that database is being replicated correctly.

## Deleting a database

User submits a request to `/user/deleteDatabase` for every storage node. The server removes all replication entries for that database.

# Client side usage

The client SDK can establish a connection to one of the storage nodes for everyday database operations (ie: CRUD) and the server-side replication will ensure all the other nodes have matching data.