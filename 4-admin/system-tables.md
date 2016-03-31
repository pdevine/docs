---
layout: documentation
title: System tables
docs_active: system-tables
permalink: docs/system-tables/
---

Starting with version 1.16, RethinkDB maintains special *system tables* that contain configuration and status information about servers, databases, individual tables, and issues with the cluster. Querying system tables returns information about the status of the cluster and current objects (such as servers and tables) within the cluster. By inserting or deleting records and updating fields in these tables, the configuration of the objects they represent can be modified.

{% toctag %}

# Overview #

Access the system tables through the `rethinkdb` database. These tables aren't real RethinkDB document stores the way user-created tables are, but rather "table-like" interfaces to the system allowing most ReQL commands to be used for control. System tables cannot be created, dropped, reconfigured, or renamed.

The metadata in the system tables applies to the RethinkDB cluster as a whole. Each server in a cluster maintains its own copy of the system tables. Whenever a system table on a server changes, the changes are synced across all the servers.

__Note:__ As of version 2.3, only the `admin` user can write to system tables, but other RethinkDB user accounts can read system tables if they have global read permissions, database-scoped read permissions on the `rethinkdb` database, or table-scoped read permissions on individual tables. (There are some individual exceptions noted where appropriate.) Read [Permissions and user accounts][pua] for more details on user accounts and permissions.

[pua]: /docs/permissions-and-accounts/


## The Tables ##

* `table_config` stores table configurations, including sharding and replication. By writing to `table_config`, you can create, delete, and reconfigure tables.
* `server_config` stores server names and tags. By writing to this table you can rename servers and assign them tags.
* `db_config` stores database UUIDs and names. By writing to this table, databases can be created, deleted or modified.
* `cluster_config` stores the authentication key for the cluster.
* `table_status` is a read-only table which returns the status and configuration of tables in the system.
* `server_status` is a read-only table that returns information about the process and host machine for each server.
* `current_issues` is a read-only table that returns statistics about cluster problems. For details, read the [System current issues table][sit] documentation.
* `users` stores RethinkDB user accounts. (See [Permissions and user accounts][pua].)
* `permissions` stores permissions and scopes associated with RethinkDB user accounts. (See [Permissions and user accounts][pua].)
* `jobs` lists the jobs&mdash;queries, index creation, disk compaction, and other utility tasks&mdash;the cluster is spending time on, and also allows you to interrupt running queries.
* `stats` is a read-only table that returns statistics about the cluster.
* `logs` is a read-only table that stores log messages from all the servers in the cluster.

[sit]: /docs/system-issues/

## Caveats ##

* While system tables support changefeeds, they do not support all of the chaining that real tables do. For instance, aggregation (`max` and `min`) and `limit` commands will not work with system tables.
* Some system tables are read-only. System tables which allow writing require specific document schema, described below.
* Write operations on system tables are non-atomic. Avoid writing to the same system table row from more than one client at the same time.
* The `durability` argument on writes is ignored for system tables.

With system tables only, the `table` command takes a new argument, `identifier_format`. Legal values are `name` and `uuid`. When it's set to `uuid`, references in system tables to databases or other tables will be UUIDs rather than database/table names. This is useful for writing scripts and administration tasks, as UUIDs remain consistent even if object names change. The default is `name`.

# Configuration tables #

## table_config ##

Sharding and replication can be controlled through the `table_config` table, along with the more advanced settings of write acknowledgements and durability. Tables can also be renamed by modifying their rows. A typical row in the `table_config` table will look like this:

```json
{
    id: "31c92680-f70c-4a4b-a49e-b238eb12c023",
    name: "tablename",
    db: "test",
    primary_key: "id",
    shards: [
        {
            primary_replica: "a",
            "replicas": ["a", "b"],
            "nonvoting_replicas": []
        },
        {
            primary_replica: "b",
            "replicas": ["a", "b"]
            "nonvoting_replicas": []
        }
    ],
    indexes: ["index1", "index2"],
    write_acks: "majority",
    durability: "hard"
}
```

* `id`: the UUID of the table. Read-only.
* `name`: the name of the table.
* `db`: the database the table is in, either a name or UUID depending on the value of `identifier_format`. Read-only.
* `primary_key`: the name of the field used as the primary key of the table, set at table creation. Read-only.
* `shards`: a list of the table's shards. Each shard is an object with these fields:
	* `primary_replica`: the name or UUID of the server acting as the shard's primary. If `primary_replica` is `null`, the table will be unavailable. This may happen if the server acting as the shard's primary is deleted.
	* `replicas`: a list of servers, including the primary, storing replicas of the shard.
	* `nonvoting_replicas`: a list of servers which do not participate in "voting" as part of [failover][]. If this field is omitted, it is treated as an empty list. This list must be a subset of the `replicas` field and must not contain the primary replica.
* `indexes`: a list of secondary indexes in the table. Read-only.
* `write_acks`: the write acknowledgement settings for the table. When set to `majority` (the default), writes will be acknowledged when a majority of replicas have acknowledged their writes; when set to `single` writes will be acknowledged when a single replica acknowledges it.
* `durability`: `soft` or `hard` (the default). In `hard` durability mode, writes are committed to disk before acknowledgements are sent; in `soft` mode, writes are acknowledged immediately upon receipt. The `soft` mode is faster but slightly less resilient to failure.

[failover]: /docs/failover/

If you `delete` a row from `table_config` the table will be deleted. If you `insert` a row, the `name` and `db` fields are required; the other fields are optional, and will be automatically generated or set to their default if they are not specified. Do not include the `id` field. The system will auto-generate a UUID. 

If you `replace` a row in `table_config`, you must include all the fields. It's usually easier to `update` specific fields.

Native ReQL commands like `reconfigure` also control sharding and replication, and if you're not using server tags you can change sharding/replication settings in the web UI. Read [Sharding and replication][shrep] for more details.

[shrep]: /docs/sharding-and-replication/

## server_config ##

This table stores the names of servers along with their *tags.* Server tags organize servers into logical groups: servers could be tagged by usage (database, application, etc.), or by data center location ("us_west," "us_east," "london," and so on). For more about server tags, read [Sharding and replication][shrep].

Every server that has ever been part of the cluster and has not been permanently removed will have a row in this table in the following format.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "servername",
    tags: ["default"]
}
```

* `id`: the UUID of the server. (Read-only.)
* `name`: the server's name.
* `tags`: a list of unordered tags associated with the server.

If tags aren't specified when a server starts, the server is automatically assigned the `default` tag. Documents cannot be inserted into `server_config`. A new document gets created when a server connects to the cluster.

Documents cannot be deleted from this table. When a server loses its connection to the cluster, its corresponding document will be automatically deleted.

## db_config ##

One document exists in `db_config` for each database in the cluster, with only two fields in the document.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "dbname"
}
```

* `id`: the UUID of the database. (Read-only.)
* `name`: the name of the database.

Documents can be inserted to create new databases, deleted to remove databases, and modified to rename databases. (Renaming databases is the only task that requires querying the `db_config` table; the other two tasks have native ReQL commands, [dbCreate][dbc] and [dbDrop][dbd].) As with tables, if you `insert` a database, don't include the `id` field: the system will auto-generate the UUID.

[dbc]: /api/javascript/db_create
[dbd]: /api/javascript/db_drop

## cluster_config ##

The `cluster_config` table contains two rows, each one with two key/value pairs: a unique `id` and a variable. Documents cannot be inserted into or deleted from this table.

```js
{
    id: "auth",
    auth_key: null
},
{
    id: "heartbeat",
    heartbeat_timeout_secs: 10
}
```

* `id`: the primary key, `auth` or `heartbeat`.
* `auth_key`: the [authentication key][auth], or `null` if no key is set.
* `heartbeat_timeout_secs`: the time, in seconds, between when a server loses connectivity to a cluster and the [failover][] process begins. The default is 10 seconds.

[auth]: /docs/security/
[failover]: /docs/failover/

Updating the `auth_key` field is the only way to set or change the cluster authentication key. Read [Securing your cluster][auth] to learn about the key and why you might want to set it.

The `auth_key` field is unusual in that it is a *write-only* field. If you try to read its value, you will get `null` or `{hidden: true}` but will not see the actual key.

# Status tables #

All the status tables are read-only. Some of the information in status tables is also returned in config tables (such as object names and UUIDs).

## table_status ##

This table stores information about table availability. There is one document per table (not counting system tables).

```js
{
    id: "31c92680-f70c-4a4b-a49e-b238eb12c023",
    name: "tablename",
    db: "test",
    status: {
        ready_for_outdated_reads: true,
        ready_for_reads: true,
        ready_for_writes: true,
        all_replicas_ready: true
    },
    shards: [
        {
            primary_replicas: ["a"],
            replicas: [{server: "a", state: "ready"}, {server: "b", state: "ready"}]
        },
        {
            primary_replicas: ["b"],
            replicas: [{server: "a", state: "ready"}, {server: "b", state: "ready"}]
        }]
}

```

* `id`: the UUID of the table.
* `name`: the table's name.
* `db`: the database the table is in, either a name or UUID depending on the value of `identifier_format` (see "caveats" in the overview at the top of this document).
* `status`: the subfields in this field indicate whether all shards of the table are ready to accept the given type of query: `outdated_reads`, `reads` and `writes`. The `all_replicas_ready` field indicates whether all backfills have finished.
* `shards`: one entry for each shard in `table_config`. Each shard's object has the following fields:
	* `primary_replicas`: a list of zero or more servers acting as primary replicas for the shard. If it contains more than one server, different parts of the shard are being served by different primaries; this is a temporary condition.
	* `replicas`: a list of all servers acting as a replica for that shard. This may include servers which are no longer configured as replicas but are still storing data until it can be safely deleted. The `state` field may be one of the following:
		* `ready`: the server is ready to serve queries.
		* `transitioning`: the server is between one of the above states. A transitioning state should typically only last a fraction of a second.
		* `backfilling`: the server is receiving data from another server.
		* `disconnected`: the server is not connected to the cluster.
		* `waiting_for_primary`: the server is waiting for its primary replica to be available.
		* `waiting_for_quorum`: the primary is waiting for a quorum of the table's replicas to be available before it starts accepting writes.

## server_status ##

This table returns information about the status and availability of servers within a RethinkDB cluster. A single document is created for each server that connects to the cluster. If a server loses its connection to the cluster, it will be removed from the `server_status` table.

This is a typical document schema for a server connected to the host server&mdash;that is, the server the client's connecting to when they query the `server_status` table.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "servername",
    network: {
        hostname: "companion-cube",
        cluster_port: 29015,
        http_admin_port: 8080,
        reql_port: 28015,
        time_connected: <ReQL time object>,
        connected_to: {
            "companion-orb": true,
            "companion-dodecahedron": true
        },
        canonical_addresses: [
            { host: "127.0.0.1", port: 29015 },
            { host: "::1", port: 29015 }
            ]
    },
    process: {
        argv: ["/usr/bin/rethinkdb"],
        cache_size_mb: 1882.30078125,
        pid: 28580,
        time_started: <ReQL time object>,
        version: "rethinkdb 2.1.0-xxx (CLANG 3.4 (tags/RELEASE_34/final))"
    },
}
```

* `id`: the UUID of the server.
* `name`: the name of the server.
* `network`: information about the network the server is on:
	* `hostname`: the host name as returned by `gethostname()`.
	* `*_port`: the RethinkDB ports on that server (from the server's own point of view).
	* `canonical_addresses`: a list of the canonical addresses and ports of the server. These may differ from `hostname` and `cluster_port` depending on your network configuration.
	* `time_connected`: the time the server connected (or reconnected) to the cluster.
	* `connected_to`: a key/value list of servers this server is either currently connected to (`true`), or knows about but is not currently connected to (`false`). In most cases other servers will be identified by name, but if the server being queried cannot determine the name of a server in the cluster it is not connected to, it will be identified by UUID.
* `process`: information about the RethinkDB server process:
    * `argv`: the command line arguments the server started with, as an array of strings.
	* `cache_size_mb`: the cache size in megabytes. (This can be [configured on startup][startup].)
	* `pid`: the process ID.
	* `time_started`: the time the server process started.
	* `version`: the version string of the RethinkDB server.

[startup]: /docs/cluster-on-startup/

# User account tables #

## users ##

The `users` table contains one document for each user in the system, each with two key/value pairs: a unique `id` and a `password` field. The `id` is the account name. The `password` field behaves differently on writes than on reads; you can change an account's password by writing a value to this field (or remove the password by writing `false`), but the password cannot be read. Instead, on a read operation `password` will be `true` or `false`, indicating whether the account has a password or not.

```js
{
    id: "admin",
    password: true
}
```

Documents can be inserted into `users` to create new users and deleted to remove them. You cannot change the `id` value of an existing document, only change or remove passwords via `update`.

Non-admin user accounts may be granted write permission on the `users` table to allow them to change their own password. This permission will not allow them to write to any other document in the table, or insert or delete any documents (including their own). The `admin` user can perform any read and write operation.

## permissions ##

Documents in the permissions table have two to four key/value pairs.

* `id`: a list of one to three items indicating the user and the scope for the given permission, the items being a username, a database UUID (for database and table scope), and a table UUID (only for table scope).
* `permissions`: an object with one to four boolean keys corresponding to the valid permissions (`read`, `write`, `connect` and `config`).
* `database`: the name of the database these permissions apply to, only present for permissions with database or table scope.
* `table`: the name of the table these permissions apply to, only present for permissions with table scope.

```js
{
    id: [
            "bob"
        ],
    permissions: {
        read: true,
        write: false,
        config: false
    }
}
{
    database: "field_notes",
    id: [
            "bob",
            "8b2c3f00-f312-4524-847a-25c79e1a22d4"
        ],
    permissions: {
        write: true
    }
}
{
    database: "field_notes",
    table: "calendar",
    id: [
            "bob",
            "8b2c3f00-f312-4524-847a-25c79e1a22d4",
            "9d705e8c-4e49-4648-b4a9-4ad82ebba635"
        ],
    permissions: {
        write: false
    }
}
```

__Note:__ The `table` and `database` fields will be automatically filled in when inserting into `permissions`, based on how many items are in the `id` list.

Under most circumstances, it is easier to manipulate the `permissions` table by using the [grant][] command.

[grant]: /api/javascript/grant

# Other tables #

## current_issues ##

This table shows problems that have been detected within the RethinkDB cluster. For details, read the [System current issues table][sit] documentation.

[sst]: /docs/system-stats/

## jobs ##

The `jobs` table provides information about tasks running within the RethinkDB cluster, including queries, disk compaction, and index construction, and allows you to kill query jobs by deleting them from the table. For details, read the [System jobs table][sjt] documentation.

[sjt]: /docs/system-jobs/

## stats ##

The `stats` table provides statistics about server read/write throughput, client connections, and memory usage. For details, read the [System stats table][sst] documentation.

## logs ##

This table stores the log files of the cluster. One row is added to the table for each log message generated by *each* server that's connected to the cluster. A maximum of 1000 entries will be stored for each server.

```js
{
    id: ["2015-01-09T02:11:55.190829899", "5a59c88f-8f66-4703-bf74-bf4cd7205db3"]
    level: "notice",
    message: "Running on Linux 3.13.0-24-generic x86_64",
    server: "companion_cube_3yz",
    timestamp: <ReQL time obj>,
    uptime: 0.389226
}
```

* `id`: a two-element array, consisting of the timestamp of the log entry (in UTC) and the UUID of the server generating the message.
* `level`: a string indicating the log message's severity level. One of `debug`, `info`, `notice`, `warn`, or `error`.
* `message`: the contents of the log message.
* `server`: the UUID or name of the generating server (depending on the value of `identifier_format`).
* `timestamp`: the time when the log message is posted.
* `uptime`: how many seconds the server had been running at the time the log message was generated.

The `logs` table supports changefeeds. Only messages being *written to the logs table* will generate changefeed events.

* The table stores a maximum of 1000 messages per server. The changefeed will not deliver events for log entries when they are removed.
* When a server connects or disconnects, its log entries will be added to or removed from the `logs` table. The action of connecting or disconnecting will not generate changefeed events for those log entries.
