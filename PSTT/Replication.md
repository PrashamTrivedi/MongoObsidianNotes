Concept of maintaining multiple copies of the data. Replication provides high availability by default. Database that does not use application is called standalone node and that database will not be accessible in events of disaster or maintenance. When data is replicated to different servers, each server will be called nodes in MongoDB.

In MongoDB, a group of nodes which have same data is called `Replica Set`, in which all data is handled in one of the nodes, and it's up to the remaining nodes to be in sync. The node where data is sent is called `Primary Node`, and all other nodes are called `Secondary Nodes`. When primary node goes down, `failover` happens and one of the secondary nodes is promoted as primary node.

## Replica sets on MongoDB

Groups of `mongod` that share same information between them. They can be either primary nodes where all reads and writes happen, or secondary nodes which are responsible for replicate all information and server as HA node so that it can perform failover.

The replication from primary to secondary nodes are done asynchronously. The replication protocol is written such a way that secondary versions. There are two versions of the protocol, PV1 and PV0 (`P`rotocol `V`ersion `1` or `0`). The difference between the protocols is the way durability and availability is forced throughout the set. PV1 is default version, which is based on [RAFT Protocol](http://thesecretlivesofdata.com/raft/) Using [Raft Consensus Algorithm](https://raft.github.io/).

>[!note]- RAFT
>In RAFT there are some election mechanism happening between available nodes. In this election mechanism, all nodes are given random timeout, which consists of election cycle. At the end of first election cycle one or more nodes announce themselves as candidate. As soon as a node becomes candidate, it asks other nodes for votes. All non-candidate nodes vote for exact one node on first-come first-serve basis. The candidate node receives most votes become primary and other nodes become follower.
>Clients connect to primary node and send write command to it. Primary node propagate all write commands to follower nodes. Only when the majority of nodes acknowledge of update command, Primary node acknowledges the write operation.


### PV1 in MongoDB

Protocol version 1 (PV1) is based on RAFT protocol. Here all write operations are tracked using oplog (which is statement based log). Secondary nodes sync their data from primary using oplog. Write statement are recorded in oplog using idempotent form[^1].

Among secondary node, there can be `arbiter` node, which can't hold any data, can not become primary and their voting for primary node will only be considered in place of tie-breakers

## Replica sets topology

- The list of replica set members and their configuration define their topology. 
- Any changes in topology trigger election. 
	- Adding new nodes, Any nodes getting down or changing replica set configuration will be considered as topology change.

- The topology of replica set is defined one of the nodes, and shared between nodes using replication mechanism.

### Other secondary node types

- `Priority 0 Secondary`: Just like regular nodes
	- ![[#^priority0Nodes]]
- `Hidden Nodes`: They can be specific secondary nodes that provides specific read-only workloads (like analytics), or have copies of data which are hidden from Application
- `Delayed Nodes`: Hidden nodes can also be set with some delay in their replication. These can be called delayed nodes.
  - Delayed nodes can be useful to provide resilience. Any data corruption won't reach to delayed node before delayed time, which allows us to recover data from delayed node.

### Some numbers

- There can be at max 50 nodes in the replica sets.
- Maximum 7 nodes can participate in votes.
  - More than 7 nodes may require more time in elections, which is not beneficial for election purpose.
  - Out of those 7 nodes, one of them can be primary and remaining 6 will be able to participate in next round of electing primary nodes

[^1]: ![[Design Skills and Advanced Features#^idempotency]]

### Some commands

- `rs.initiate()`: Should be called from first node which is supposed to be master. This initiates replica sets.
- `rs.status()`: Gets status of replica sets, like nodes, heartbeat time etc.
  - Uses heartbeat data to gather status. So this command can be late because of that.
  - Lists all nodes, current node is marked as `self:true` property.
  - Can check when sync was happened based on `lastHeartbeat` (Primary to secondary), or `lastHeartbeatRecv` (Secondary to Primary) values.
- `rs.add({serverName}:{port})`: Adds a node (as secondary) to replica set. Should be called from master.
- `rs.addArb({serverName}:{port})`: Adds an arbiter to replica set.
- `rs.isMaster()`: Checks if current node is master or not. ^rsIsMaster
  - Gives primary node with `primary` key, current node with `me` key.
- `db.serverStatus()['repl']`: Same as `rs.isMaster()` except it prints `rbid` value which is the number of rollbacks in current node.
- `rs.printReplicationInfo()`: Prints data about oplog for current node.
- `rs.stepDown()`: Should be called from master. Steps down current node as master to force election. ^stepDown


## Replication Configuration

- JSON Object that defines configuration options of our replica set
- Can be configured manually from set
- Can be managed by mongo shell replication helper methods like `rs.add`, `rs.initiate`, `rs.remove`, `rs.reconfig`, `rs.config`.
- Current replication configuration can be fetched from calling `rs.config()` or `rs.conf()` commands.

### Some interesting fields in replicasets

- `_id`: Name of the replica sets
- `version`: Incremented every time the replica set is changed. (Like new node added, removed)
- `term`: (Added after MongoDB 4.4) Incremented every time a primary node is stepped down, and new primary node is elected. That primary node will increment the term. #version #gotchas  
  - The latest configuration will be determined from greater term value. If term is not present or have same value in all nodes, configuration with the greatest version will be considered latest.
- `members`: List all members in Replica set.
  - `host`: Having host name and port.
  - `arbiterOnly`: Indicates if given node is arbiter or not. (Default `false`).
  - `buildIndexes`: Indicates if given node builds index for this member. (Default `true`), build index can only be set at adding time.
  - `hidden`: Sets this secondary as hidden node.
  - `priority`: A number that indicates the relative eligibility of a member to become a priority. (Default 1, 0 if arbiter).
    - Can be set 0-1000 for not arbiter, 0 or 1 for arbiters.
    - Change in priority is topology change, thus forcing elections.
    - Priority must be 0 when node is hidden, because hidden node can not become priority.
  - `secondaryDelaySecs` (Previously `slaveDelay`): Determines number of seconds this node should lag behind primary.
  - `votes`: Votes the given member can cast.

### Reconfiguration of replica set without stopping any nodes

Here are steps to reconfigure replica set.

>[!warning]
>Reconfiguration must be done from master node. Secondary node can not reconfigure themselves


1. Get replica set configuration from `rs.conf()`. And store it in a variable.(e.g. `cfg=rs.conf()`)
2. Change values of configuration on the fly. (e.g.: `cfg.members[i].vote=0` or `cfg.members[2].hidden=true`.)
3. Reapply configuration by calling `rs.reconfig()` and pass the variable.

>[!warning]
>If the configuration changes are not valid, Reconfiguration will not be successful and will keep reapplying until stopped manually


## oplog.rs

When a node is connected to a replicaset, some collections are added to `local` database. Any data written in `local` database, as the name suggests, stays in local node and is not eligible for replication. Local database has many pre-added collections. One important collection is `oplog.rs`. This collection keeps information about [[Production Ready Development#Oplog|oplogs]].

This collection is read only and memory capped. That means older entries in oplogs will be removed if memory is more than capped size. By default, capped size is 5% of disk space. [^2]. Oplog size can be configured by mentioning `replication.oplogSizeMB` in configuration file. ^ Since oplog.rs is capped collection, once the limit is reached, the earliest entry will be overwritten with the newer oplog entries. ^oplogCapped

Oplog size determines how long our node can be down and still be synced once they come back. Database write operations are written to the primary oplog and synced to secondary nodes. If one of the secondary node is down, primary will propagate oplog to other secondary nodes, once the downed secondary node comes back, this secondary node will be marked as recovery node. If the recovery node has oplog entry, and it's last oplog statement is found in other nodes, it will sync those oplog operations before being available. ^oplogRecovery

In case the last oplog entry is recycled from all the other nodes, the recovery node can't be in sync. However, different nodes can have different oplog size, and they can have larger oplog, so that recovery for any node is possible.

Sometimes an operation, while reducing number of commands and roundtrips, can generate many oplog entries. `updateMany` is one of them. The reason is all the bulk write operations should be idempotent. ^oplogIdempotent

[^2]: For Latest cap size, please refer [Official Document](https://www.MongoDB.com/docs/manual/core/replica-set-oplog/#oplog-size)

## Reads and Writes

By default, we can not read from secondary, because MongoDB always prefers consistency. To have a consistent view of data, MongoDB disables read from secondary unless we are explicitly say that we want to see secondary data. To enable read from secondary run `db.getMongo().setReadPref("secondary")` or `rs.secondaryOk()` command from the secondary node you are running.

As noted earlier, shutting down a node triggers re-election. If all (or majority) secondary nodes shut down, there won't be a majority that select any primary. Thus shutting down all secondary nodes will make previous primary node as secondary. As a side effect, we can not read from or write to this node.

## Write Concern

Write concern is a value specified while writing the data to primary. It indicates the level of acknowledgement requested to increase durability.[^3] Increased durability means write is propagated to requested nodes. More replica set members that acknowledge the write, the more likely the write is durable. More durability also requires more time before the write is acknowledged. For [[Sharding|Sharded Clusters]], write concern is pushed down to the shard level.

>[!note]
> Write concern doesn't mean the data is written to only given number of nodes, it only means how concerned you are for your data durability. 

To notify write concerns in write operations, add `{writeConcern:{w,wtiemeout}}` with proper values.

### Write concern specification

- `w`: Value of write concern.

`Majority` write concern means this write is propagated and acknowledged by majority nodes, usually `{Number Of Nodes}/2 and round up` (For example, in case of 3 node replicaset, majority means 2 nodes. For 5 node replica set, majority requires 3 nodes).

| Write Concern Values | Implications |
|------------------- | ------------ |
| `0` | Don't wait for acknowledgement |
| `1` |  Wait for acknowledgement from Primary only |
| `>=2` | Wait for acknowledgement from Primary and one or more secondary  |
| `majority` | **(Default Value from MongoDB 5.0)** Wait from acknowledgement from The Majority of replica set members |

- `wtimeout`: Time to wait for (In milliseconds) write concern before marking the write operation as failed.
  - Facing `wtimeout` does not mean the write operation is failed. It means the requested durability is not reached before the time.
- `j`: If true, requires data is being written to on disk journal before acknowledgement.
	- Having `j` true will increase durability.
	- Starting with (3.2.6), this value is set true if `w` (or write concern) is set as `majority` #version #gotchas 

> [!tip]
> If write concern is specified as `majority`, only voting members' acknowledgement will be counted. If write concern is specified as number, non voting members' acknowledgements will also be counted.




[^3]: Durability means write safe and do not disappear even after database servers crash.

## Read Concern

It's acknowledgement for write concern, this specifies that the documents are only returned if they meet durability guarantee.

>[!note]
>A Document that is not returned according to `readConcern`, is not a guarantee that a document is lost, it's just a (kind of) notification that the document is not propagated to enough nodes that guarantee the durability.


### Read Concern Levels

- `local`: Returns the most recent data in the cluster (Most likely local to the primary and not acknowledged by any other nodes yet.). Default for read against primary.
- `available`: Same as local for replica-set deployments. Default for read against secondary members.
- `majority`: Returns the document only if the majority of nodes has acknowledged the write.
  - Only way the documents are lost when it's written to majority nodes, not written to other nodes and majority nodes go down.
- `snapshot`: Read what was there when query starts. 
	- AKA data on the snapshot. Not the data being written when query was received in server.
- `linearizable`: (Added in 3.4) Wait until majority is caught up till the time client sent the query.
	- i.e. If by some chance a majority has not caught up with 1s of oplog, this read concern forces to wait till 1s before running this query.
	- That guarantees that everything that has written before the query was fired, is synced in majority of nodes and considered in the query.
	- Compared to `majority` it only queries what majority sees, if something is still in primary and not synced, that will miss out.

>[!caution]
> Customer must have some valid reasons to use `snapshot` or `linearizable` read concerns. `local` or `majority` will serve the purpose.



## Read Preferences

By default, apps read and write data to primary node, and it's replicated via [[Production Ready Development#Oplog|oplog]] to secondaries. Read Preference allows apps to route read operations to specific members of replica set, it's mainly a driver side setting.

### Supported Read Preference modes

- `primary`: (Default) Reads are done from primary node only.
- `primaryPreferred`: By default reads are routed to primary, but if primary is not available (due to fail over or election), reads are routed through first available secondary.
- `secondary`: Routes read operations only to the secondary members in replica sets.
- `secondaryPreferred`: Routes read operations from secondary, but if no secondary members are available, reads are routed through primary.
- `nearest`: Routes read operations to the replica set node with the least network latency to the host, regardless of the member type. This supports geographically local read.
- We can also specify tags if we want to read from specific secondaries.

In above modes, if read preference is primary, that means secondaries are for HA only. And in all other modes, we may receive stale data.