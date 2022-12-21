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

This is achieved by creating a storage node `replication user` on the `destination node` for the `source node`. The user will have the following attributes:

- `username` = `hash(sourceNodeEndpointUri)` (ensures each endpoint has a unique username)
- `roles` = [`${contextHash}-replicator`] (ensures each endpoint has consistent replication permissions for a given application context)

A storage node `replication user` has a `${contextHash}-replicator` role that grants read and write access to all databases in the given context.

Each database in the application context has this role added to it's `members.roles` attribute to provide read access. Similarly, custom security document in CouchDB (`validate_doc_update()`) accepts write requests for this role.

A storage node `replication user` can have multiple roles to support permissions for multiple contexts. This allows each storage node to only require a single set of credentials for every other storage node it connects to.

Note: `contextHash` is actually a hash of the database owner DID and context name. It is unique for a given DID and context. If the same DID has three contexts on the same storage node, there will be three `contextHash`'s.

Implementation notes:

1. **Per Database:** The `configurePermissions` write function needs to check `userCtx.roles` includes the role `${contextHash}-replicator`
2. **Per Database:** The `configurePermissions` security doc needs to add the role `${contextHash}-replicator` in the `members` list
3. **User:** The storage node username must be given the role `${contextHash}-replicator`
4. **Revocation:** A storage node can revoke the `${contextHash}-replicator` role to disable read/write access to databases it should no longer have replication access to

>Note: It's not possible to use JWT token authorization for replicaters due to the way CouchDB works. The JWT authentication implementation in CouchDB doesn't read the `roles` property of the `user` specified in the access token. As such, we can't use JWT tokens and specify a `user` to manage access. CouchDB does support specifying a `role` in the JWT token, however it doesn't support revoking access tokens. As such, it will be impossible to revoke that access when a replicater needs to be removed. The above approach specifies a `role` in the user and then uses normal HTTP username/password authentication (which does respect the user roles).

## Initiating and revoking replication

Assume we have three storage nodes; SN-A, SN-B, SN-C:

1. SN-A pushes to SN-B, SN-C
2. SN-B pushes to SN-A, SN-C
3. SN-C pushes to SN-A, SN-B

A user pings each storage node to confirm replication is configured for a given context.

ie: `https://storageNodeA/user/checkReplication`

This will have the following parameters:

- `did` (required)
- `contextName` (required)
- `databaseName` (optional parameter, otherwise checks all databases)

If a node is being added or removed it **must** be the last node to have `checkReplication` called. This ensures the new node has a list of all the active databases and can ensure it is replicating correctly to the other nodes.

The client SDK will call `checkReplication()` when opening a context to ensure the replication is working as expected.

### Verify valid storage nodes

The DID document will be fetched, to obtain a list of active endpoints for the user DID and context. The storage node will ensure it is pushing data to all the endpoints in the DID Document (and remove any it shouldn't be pushing to) for all the databases owned by that user.

This ensures the DID Document is the source of truth for the list of active database replicas for a `context`.

### Node has been added

The storage node may identify it has been added to the list of endpoints. The storage node doesn't need to do anything as the other nodes will contact it to initialize replication.

The storage node may identify a new replication node has been added. The storage node will request credentials from the new replication node (see below) and will create replication entries to `push` all the `context` databases to the new node.

This will include pushing the database that maintains a list of all the `context` databases.

Implementation notes:

- Need to refactor the storage of databases in a context to have their own database so they can be syncronized via CouchDB replication
- Need the client SDK to not start using the new node until it is fully syncronized (need to check `/status`?)

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

The client SDK submits a request to the `/user/createDatabase` endpoint on one of the storage nodes. This will set the `_security` document in the database and define the `validate_doc_update()` method. Both of these are configured to support any user with the `${contextHash}-replicater` role, so they will be the same across all storage nodes. CouchDb replication will replicate these documents ensuring consistent permissions across all storage nodes.

This storage node will internally call `checkReplication(databaseName: string)` to ensure it is pushing database updates to the other storage nodes. It's necessary for the all the other nodes to start replicating that database.

The database will be added to the list of `context` databases, which will also be syncronized to the other storage nodes.

Once the database is created, the client side will ping the remaining storage nodes: `https://storageNodeA/user/checkReplication?databaseName={databaseName}` to ensure that database is being replicated correctly.

It's necessary for the SDK to call `createDatabase()` on all endpoints when creating a database. It's not possible to rely on CouchDB replication to create the database as the replicator credentials (intentionally) don't have permission to create any database on the other storage nodes.

CouchDB requires storing credentials in the `_replicator` database for every database that is replicated. As such, storage nodes have a dedicated replication user (`DB_REPLICATION_USER` in `.env`) that has the special role `relication-local`. This replication user is automatically created when the server starts. All databases have this user as a `read-only` member allowing the local replication to read any changes and send them to the other endpoints.

>Note: The `replication-local` credentials must not change as they are stored in every entry in the `_replicator` database. Changing the credentials will break all replication that has already been configured.

## Deleting a database

The client SDK submits a request to `/user/deleteDatabase` for every storage node. The server removes all replication entries for that database.

# Database change log

The user database maintains a change log that tracks every `create`, `update` and `delete` database request from the Client SDK. These changelog entries are timestamped (for ordering) and signed by the controlling DID.

These change logs are automatically replicated as part of the user database list. Each storage node uses this as the source of truth for ensuring the correct databases exist, have correct permissions and have correct replication.

# Client side usage

The client SDK can establish a connection to one of the storage nodes for everyday database operations (ie: CRUD) and the server-side replication will ensure all the other nodes have matching data.

The replication will occur server side.

This maximises efficiency for clients, ensuring they don't need to maintain a connection to every storage node replica for the current application context.

# Auto-repair

There is code in the `checkReplication()` method on the storage node that gracefully cleans up the following issues:

1. A database fails to `create` on a storage node
2. A database fails to `update` on a storage node
3. A database fails to `delete` on a storage node
4. A database replication fails to be created on a storage node
5. A database replication fails to be deleted on a storage node
6. Storage node is removed from the application context
7. Storage node is temporarily unavailable

(`1`,`2`,`3`) is resolved by ensuring the current databases match the databases found in the list of application context databases maintained on the storage node.

(`4`, `5`) is resolved by ensuring the current replicated databases match the databases found in the list of application context databases maintained on the storage node.

`6` is resolved by detecting the storage node is no longer included in the DID document and deletes all data and replication associated with the application context.

`7` is resolved in different ways depending on the scenario:

1. Failure to `create`, `update`, `delete` a database on a storage node on the client side will fail quickly and will be auto-resolved next time `checkReplication()` is run on the node (when it becomes available).
2. Replication failure between nodes will be gracefully handled by CouchDB replication processes that will retry with a backoff function.
3. The Client SDK ensures it only talks to storage nodes with the most up-to-date data, giving the opportunity for temporarily down storage nodes to repair themselves and catch up

# Security risks

1. A malicious storage node could attempt to inject fake databases into the database containing all the valid databases for an application context. Similarly they could attempt to update the permissions or delete a database. The Client SDK signs all database changes and this changelog is also stored against each database entry. The storage node verifies this signed changelog before making any changes. This also ensures `_deleted` is ignored.
2. A malicious storage node could attempt to replicate local databases for a known DID, context and database name to a remote database. This will not work because the remote database will not have granted the malicious storage node read/write access for that storage node to that database.