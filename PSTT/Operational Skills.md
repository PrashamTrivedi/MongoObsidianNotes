# Caching and Durability

## Journals
- Persists change between two checkpoints
- Compressed using snappy algorithm
- Contains a record per write
- Data is durable locally once journal has bin written to disk and flushed.
- Flush is synchronised across IO Threads to improve efficiency.
- In newer versions (Probably 4.4 an above) restoring reads from oplog instead of Journal, and journaling is used for Oplog.

## DB Cache
- WireTiger has Block cache
	- Default size `Max(((AVAILABLE RAM-1GB)/2),256 MB)` ^defaultCacheSize
	- Only change if running multiple instances on same server
- A block in cache can handle multiple version of document if required.
- Unused blocks are flushed when no longer used.
- ![[Internals#Cache overflow]]
- ![[Internals#Cache eviction thresholds]]
- As Noted, [[Internals#^wtKeyValDocIndex|blocks can be collections or indexes]]

# Observing Behaviour

## `db.currentOp()`

- List Current Operations running in server. [Read More](https://www.mongodb.com/docs/v4.4/reference/operator/aggregation/currentOp/)
    - Users with `clusterMonitor` can use `currentOp`.

## `mongostat`
- Reports realtime stats
	- Inserts, queries, updates, deletes
	- Memory usage
	- Collection and network traffic
	- Separate CLI from `mongosh` or `mongo` shell.
- Some options
	- Last argument is number indicates delta time (Default is 2 seconds)
		- mongostat updates information after delta time is passed
	- `-o=` shows some fields
	- `--discover` shows all nodes in cluster.

## `mongotop`
- Tracks a time MongoDB instances spends reading and writing data in each collection.
- Finds most used collection in given second.
- Updates every seconds

# Logs And Metrics

## Access the logs

- File System: `/var/log/mongodb/mongodb.log`, 
	- or whatever is provided via command line
	- Or whatever is provided in config file.
	Default location of config files ![[Mongod#Location of config file]]
- Hosted in Atlas: Download via UI or APIs
- Database: Access with `getLog` command
![[Optimizing Storage#^dbLogs]]

![[Mongod#Profiler]]

# Finding Slow Ops in Atlas

- Performance Advisor (Not available in free tier)
- Real time performance panel 
- Query Profile

# Identifying Common Performance Issues
