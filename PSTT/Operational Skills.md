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

## Identifying Document Contention
- Symptom: 
	- Some Slow Writes
	- High CPU usage
	- `writeConflict` in logs

### Identifying Lock Contention
- Before MongoDB 5, reads can be blocked
- Tools
	- Current Op
	- LogFile
- Symptom
	- Many operations pausing simultaneously
	- Many operations finishing at same time
- Reason
	- Some collection/db level operations are being done
	- In older versions (below 4.2) creating foreground index.

# Basic Backup Options

- Backup plans require RPO and RTO
- RPO (Recovery Point Objective)
	- How much data you can afford to lose?
	- At what point in time must the backups be when we have data loss
	- It's important to know how often can we make backups
- RTO (Recovery Time Objective) ^rto
	- How long you can afford to be offline?
	- How long should backups be restored?

## Backup methods

- MongoDB Atlas: Continuous Backups or cloud providers snapshot feature
- MongoDB Cloud Manager or OPS manager: Backup Snapshot
- Self Service approaches using MongoExport or MongoDump.

| Considerations | Mongodump | File System | Cloud Manager | Ops Manager |
| ----- | ------- | ------- | ------- | ------- |
| Initial Complexity | Medium [^b] | High [^a] | Low | High |
| Replica Set Point In Time | Yes | No | Yes | Yes |
| Sharded Snapshot | No | No | Yes [^c] | Yes [^c] |
| Restore Time | Slow | Fast | Medium | Medium |
| Incremental | No | No | Yes | Yes |

[^a]: We may need to find the location and pause the `mongod` to perform backup, or call `db.fsyncLock()`, copy files and call `db.fsyncUnlock()`. This will pause writes.
[^b]: For sharding it may add some issues
[^c]: For enterprise tools


## Self Service backup options
- Document Level
	- Logical
	- `mongodump` and `mongorestore`
- File System Level
	- Physical
	- Copy Files
	- Volume/Disk snapshots
