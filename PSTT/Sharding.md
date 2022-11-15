Sharding is a horizontal scaling of a database. Where instead of storing whole data in one server (regardless of the server is [[Replication|primary or read replica]] the data is distributed in shards and stored in different servers. All shards make up the sharded cluster and that cluster is deployed as replica set to ensure HA.

Sharding comes with a complexity. After sharding is applied, client doesn't know which shard has the data that it's looking for, so either they have to make more round trips to get the exact data or server has to keep additional entries (or logic) to determine which shard hosts which data. Also adding a shard or removing shard can be cumbersome because of these entries or logic. This alone can make our app slower.

## You don't need to shard always.

A regular data-set doesn't need much sharding. Unless you're dealing with big-data issues.

### How do you tell if you need your shard.
- You have optimised all your queries, index and your schema AND
- You still have resources maxed out.
- You are reaching limits of vertical scaling that you can't afford to scale vertically.
- Or you need to meet [[Operational Skills#^rto|RTO]] target and need parallelism to do so.





## `mongos`

To fix this problem, instead of connecting to a specific shard, client should only connect to a process which has knowledge of which data is stored in which shard, that process can accept the query and then figures out which shard should receive the query. In MongoDB this process is called `mongos`. Client should connect a `mongos` instead of specific node or shard. And there can be any number of `mongos` processes.

`mongos` doesn't know anything. It uses sharding metadata (data about which data is stored in which shard) stored in config server replica set so Sharding metadata are HA.

### `mongos` and Sharding distribution

`mongos` connects with configuration server which knows about which data is connected to which shard. This data is versioned, the latest version is stored in all the shards. If `mongos` queries with old versions, shard reject that query and forces `mongos` to update the configuration to fetch  the latest sharding info. `mongos` makes sure all the data is distributed proportionally across the shards. And config servers can move around data to make sure this thing.

`mongos` requires `configDB` to be specified. And they don't have the shards added automatically, we need to call `sh.addShard()` with replicaset to add that in shard.

If a shard is down, ideally only a limited operation is down. Anything touches that shard, including [[#Targeted queries]] or [[#Scatter Gather]] queries involving said shard will fail.

## Configuration DB

To see the data of configuration DB from `mongos`, we should switch to `config` db.

### Some collections

- `databases`: Databases part of the shard
- `collection`: Collections of the shard. They have the `key` information which tells us the shard key, and they also tell us if that key is `unique` or not.
- `shards`: Stores information about shards.
- `chunks`: Each chunk for every collection in DB is stored here as documents.
  - `min`: Defines **inclusive minimum** value of the chunk
  - `max`: Defines **exclusive maximum** value of the chunk
  - `ns`: Associated collection on which shard is applied.
  - Any document in given `ns` with `shard-key`>=`min` or `shard-key`< `max` will fall in this chunk.
- `mongos`: Number of `mongos` processes connected to this cluster.

#### Primary Shard

Sharded clusters will also have primary shards, where all non-sharded collections are stored. Primary Shards are also responsible for merging operations where the documents required from query are in all shards.

## Shard Key

A shard key (or set of keys) determines on which chunk the data must reside. Based on that, on write operation `mongos` stores given document on given chunk. Shard Key must be present in every new document write. Shard Key must be indexed before we define these keys as shard keys. 


### Targeted queries

Queries that include shard key, or in compound shard keys queries that include prefix (e.g. for index that includes keys a, b and c, queries in order of a, b or a, b, c will include prefix.) will be targeted queries as they are being targeted to one or predefined number of shards.

### Scatter Gather

When we query something where result can be stored in more than one shard, `mongos` has to collect the data from all the shards and then merge the results. Reasons behind this is query without shard keys or queries where shard keys are hashed (see below) and ranges are being used. This degrades performance.

### How to shard

1. Use `sh.enableSharding("{DBName}")` to enable sharding for the given database, this only makes given database eligible for sharding, but does not shard any actual collections.
2. Create index in a collection that you want to shard.
3. Use `sh.shardCollection({"DBName.collection"},{shardKey:})` to start sharding.

### Strategies to shard

To chose shard keys, follow these recommendations #recommendation 

1. High Cardinality: The chosen shard key should have high cardinality (Number of elements in set). i.e. It should have many unique values.
2. High Frequency: Low repetition of given unique sharding key values, so one shard do not end up having data with 90% queries.
3. Don't change monotonically: Keys that don't change at a frequent and steady space (Like counters or Dates). If we consider that as shard key, all document that has the value of higher boundary then previous chunk will end up in same (newer) document.

### Hashed Shard Key

Hashed shard key is a key where underlying index is hashed. With hashed shard key, `mongodb` hashes the value using hash function and then decide which chunk to place the document based on the hash. Here hashing is only used to determine chunks, the actual data remains unchanged. More useful in scenarios where your original shard keys are changing monotonically, where instead of pushing these values in one single collection by default, these values will be pushed evenly between shards.

#### Considerations

- Hashed shard keys are not performant on ranges. This involves [[#Scatter Gather]]
- Hashed shard keys can not support zoned sharding or geographically isolated reads.
- Hashed shard key must be single non-array, non-object field.
- Hashed shard key can not support index performance gain.

### How to shard using hashed key

1. Use `sh.enableSharding("{DBName}")` to enable sharding for the given database, this only makes given database eligible for sharding, but does not shard any actual collections.
2. Create index in a collection that you want to shard. Here instead of regular index you should provide `"hashed"` as value. (Like instead of creating index like `createIndex({field:1})`, `createIndex({field:"hashed"})` should be used).
3. Use `sh.shardCollection({"DBName.collection"},{shardKey:"hashed"})` to start sharding.


## Chunks

When we initiate sharding, every document is stored in one chunk, which will be split eventually in different chunks. Following aspect define number of available chunks. Data is only redistributed in chunks if they are getting changed.

- Chunk Size:
  - 64 Mb (Default).
  - If the chunk is about the chunk size or within chunk size limit it will be split.
  - We can define Chunk size between 1mb and 1gb.
  - It can be configured during runtime

### Jumbo Chunks

- Larger than defined chunk size
- They can not be moved
- Balancer skips this chunk and avoid trying to move them
- In extreme cases they will not be able to split

## Balancing

MongoDB Balancer Identifies which shard has too many chunks and moves chunks across shard in the sharded cluster to achieve even distribution. It runs on primary node of config server replica set. If it detects an imbalance, it starts a balancer round, the balancer can migrate chunks in parallel, and given shard can not participate in more than one rounds. Balancer can migrate `Math.floor(numberOfShards/2)` chunks in given round. Balancing rounds keeps happening until balancer detects all the shards are as evenly distributed as possible. Balancer can split chunks if needed.

We can start the balancer using `sh.startBalancer(timeout,interval)` and stop the balancer using `sh.stopBalancer(timeout,interval)` commands, balancer will only stop after current round completes. Here `timeout` means how long to wait before start or stop balancer, and `interval` means how long to wait before balancer check and start new round if required.

## Querying sharded cluster

As stated earlier, `mongos` (via config server) has the information of which document is in which chunk. If the query has the `shard key`, `mongos` will redirect the query to specific chunks, otherwise `mongos` will search in all shards and do merging to deliver final results. Mostly `mongos` passes cursor methods like `sort`, `limit` and `skip` to shards, merges the result and re applies all cursor methods before delivering final result. For aggregation and sharded clusters please refer to [MongoDB discussion here](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-sharded-collections/)